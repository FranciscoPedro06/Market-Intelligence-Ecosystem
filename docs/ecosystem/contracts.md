# Contratos de Dados do Ecossistema — C0 · C1 · C2 · C3

> **Responsabilidade deste documento:** ser a fonte única e versionada dos contratos de dados entre os produtos. Nenhum repositório é dono de um contrato que compartilha (governança: contratos vivem no ecossistema).
> **Guardião:** EM → Sprint Lead durante a Sprint 1.
> **Princípio contract-first:** C1 e C2 são congelados cedo para os três produtos evoluírem em paralelo sem se quebrarem.

Legenda de status: 🔒 congelado/versionado · 🧪 validado (fonte externa) · 🕓 a definir quando entregue.

| Contrato | Entre | Status |
|---|---|---|
| **C0** | Fonte ANAC/VRA → Collector | 🧪 validado na Fase 0 (Sprint 1) |
| **C1** | Collector → Analytics | 🔒 `v1.0.0` |
| **C2** | Analytics → API | 🔒 `v1.2.0` |
| **C3** | API → Consumidor | 🔒 `v1.0.0` — núcleo normativo (protocolo e superfície 🕓) |

---

## C0 — Fonte (ANAC/VRA) → Collector · validado 2026-07-24

Contrato **externo** (não controlamos). Validado por spike contra o dado real (`VRA_2023_06.csv`, 79.762 linhas).

- **Fonte:** ANAC SIROS, arquivo CSV mensal em URL determinística:
  `https://siros.anac.gov.br/siros/registros/diversos/vra/{YYYY}/VRA_{YYYY}_{MM}.csv`
- **Formato:** CSV `;`-delimitado, UTF-8, cabeçalho, datas `DD/MM/YYYY HH:MM` (horário de Brasília).
- **Granularidade:** um registro por etapa de voo.
- **Cadência:** mensal, defasagem ~1 mês (mês fechado M disponível ~fim de M+1).
- **Situação do voo** (`Situação Voo`): exatamente `REALIZADO`, `CANCELADO`, `NÃO INFORMADO`.
- **Códigos de aeroporto:** ICAO (`SBSP`, `SBRJ`), **não** IATA.
- **Correções:** arquivos mensais podem ser re-publicados na mesma URL → tratar re-fetch como sobrescrita (idempotência via chave natural + hash do arquivo).
- ⚠️ **Não** usar a API JSON do dados.gov.br (401 sem token) — usar as URLs diretas do SIROS.

---

## C1 — Collector → Analytics · 🔒 `v1.0.0` (congelado 2026-07-24)

> Esquema do **registro bruto de voo**. Camada bruta: valores gravados **exatamente como publicados** (sem re-mapeamento semântico). Toda derivação (IATA, minutos de atraso, % de pontualidade) pertence ao Analytics, não ao C1.
> **Origem:** rascunho entregue pelo Collector Engineer no spike da Fase 0, revisado e congelado pelo Sprint Lead.

### Garantias
- **Completude:** todas as colunas da fonte preservadas; nenhuma linha descartada silenciosamente (`NÃO INFORMADO` e `CANCELADO` chegam ao Analytics).
- **Particionamento:** por `source_year_month` (`YYYY-MM`).
- **Idempotência:** upsert por `record_natural_key`; reexecutar o mesmo mês não duplica. Re-publicação detectável por `file_sha256`.
- **Proveniência:** cada linha carrega origem, URL, mês de referência, hash do arquivo e timestamp de ingestão.

### Campos derivados da fonte (1:1 com o CSV VRA)

| Campo | Tipo | Coluna VRA | Notas |
|---|---|---|---|
| `airline_icao` | string(3) | Sigla ICAO Empresa Aérea | ex. `TAM`, `GLO`, `AZU` |
| `airline_name` | string | Empresa Aérea | |
| `flight_number` | string | Número Voo | manter string (zeros à esquerda) |
| `di_code` | string | Código DI | |
| `line_type` | string(1) | Código Tipo Linha | |
| `aircraft_type` | string | Modelo Equipamento | tipo ICAO |
| `seats` | integer | Número de Assentos | |
| `origin_icao` | string(4) | Sigla ICAO Aeroporto Origem | ex. `SBSP` |
| `origin_desc` | string | Descrição Aeroporto Origem | |
| `dest_icao` | string(4) | Sigla ICAO Aeroporto Destino | ex. `SBRJ` |
| `dest_desc` | string | Descrição Aeroporto Destino | |
| `scheduled_departure` | timestamp (naive, America/Sao_Paulo) | Partida Prevista | nullable |
| `actual_departure` | timestamp | Partida Real | nullable (null se cancelado) |
| `scheduled_arrival` | timestamp | Chegada Prevista | nullable |
| `actual_arrival` | timestamp | Chegada Real | nullable |
| `flight_status` | enum {`REALIZADO`,`CANCELADO`,`NÃO INFORMADO`} | Situação Voo | string bruta, sem tradução |
| `justification` | string (nullable) | Justificativa | quase sempre vazio |
| `reference_date` | date | Referência | `YYYY-MM-DD` dia de serviço |
| `departure_punctuality` | string (nullable) | Situação Partida | bucket ANAC (informativo) |
| `arrival_punctuality` | string (nullable) | Situação Chegada | bucket ANAC (informativo) |

### Campos de proveniência (adicionados pelo Collector)

| Campo | Tipo | Propósito |
|---|---|---|
| `source_file` | string | ex. `VRA_2023_06.csv` |
| `source_url` | string | URL SIROS completa |
| `source_year_month` | string | `2023-06` (chave de partição) |
| `file_sha256` | string | hash do arquivo (detecta re-publicação) |
| `ingested_at_utc` | timestamp | momento da ingestão |
| `row_number` | integer | posição na origem (auditoria) |
| `record_natural_key` | string | hash de (airline_icao, flight_number, origin_icao, dest_icao, scheduled_departure, reference_date) para upsert idempotente |

**Nota de derivação para o Analytics:** o mapa ICAO→IATA (`SBSP`→`CGH`, `SBRJ`→`SDU`) e o cálculo de pontualidade (`metrics-definitions.md`) são responsabilidade do Analytics, **não** do C1.

---

## C2 — Analytics → API · 🔒 `v1.2.0` (congelado 2026-07-24; emendado 2026-07-25; artefato identificado 2026-07-26; envelope 2026-08-31)

> Esquema do **indicador de pontualidade** por **rota (direcional) × companhia × mês**, aplicando exatamente `metrics-definitions.md → pontualidade v1.1.0` sobre o registro bruto `C1 v1.0.0`.
> **Princípio (RT5):** a API **não** calcula nada. O C2 carrega o número já pronto (`on_time_rate`), numerador, denominador, contadores de transparência e linhagem — a API apenas serve.
> **Origem:** rascunho do Analytics Engineer na Fase 0, revisado e congelado pelo Sprint Lead (ambiguidades R1–R5 resolvidas).

### Grão e identidade
- **Grão:** um registro por **(rota direcional origem→destino) × (companhia ICAO) × (mês de referência)**.
- Rota **direcional**: a pontualidade é medida na **chegada ao destino**; misturar CGH→SDU com SDU→CGH conflaria duas operações distintas. A pergunta-âncora **CGH↔SDU** é respondida pela API **somando os dois registros direcionais** que compartilham `route_pair_id`.
- Mapa ICAO→IATA (`SBSP`→`CGH`, `SBRJ`→`SDU`) é responsabilidade do Analytics, materializado como colunas explícitas.

### Campos — Chaves / Dimensões

| Campo | Tipo | Exemplo | Notas |
|---|---|---|---|
| `route_id` | string | `SBSP-SBRJ` | Chave direcional canônica `originICAO-destICAO`. |
| `route_pair_id` | string | `SBRJ-SBSP` | Chave **não direcional** (par ICAO ordenado). API agrupa as duas direções para a comparação ↔. |
| `origin_icao` | string(4) | `SBSP` | Herdado do C1. |
| `origin_iata` | string(3) | `CGH` | Derivado pelo Analytics (mapa ICAO→IATA). |
| `dest_icao` | string(4) | `SBRJ` | Herdado do C1. |
| `dest_iata` | string(3) | `SDU` | Derivado pelo Analytics. |
| `airline_icao` | string(3) | `TAM` | Herdado do C1. Chave da companhia. |
| `airline_name` | string (nullable) | `TAM LINHAS AEREAS S.A.` | Herdado do C1; `null` se ausente — nunca inventado. |
| `reference_month` | string(7) | `2023-06` | `YYYY-MM`. Igual à partição C1 consumida. |
| `timezone` | string | `America/Sao_Paulo` | Constante; explícito para reconciliação. |

### Campos — Medidas

| Campo | Tipo | Notas |
|---|---|---|
| `flights_operated` | integer (≥0) | **Denominador.** `flight_status = REALIZADO` **E** `actual_arrival` não nulo **E** `scheduled_arrival` não nulo. *(v1.1.0: passou a exigir também `scheduled_arrival` — sem previsão, a pontualidade é indefinida.)* |
| `flights_on_time` | integer (≥0) | **Numerador.** Subconjunto com `(actual_arrival − scheduled_arrival) ≤ 15 min` (antecipado = pontual; +15 inclusivo). |
| `on_time_rate` | decimal[0,1] (nullable) | `flights_on_time / flights_operated`. **`null` quando denominador = 0** (nunca 0/0). Fração, não percentual; **precisão plena** — nem o C2 nem o C3 arredondam; arredondar é de quem exibe (C3 §*Nota de escopo*). |
| `flights_cancelled` | integer (≥0) | Transparência: `CANCELADO`. Fora do denominador (fora de escopo), nunca descartado. |
| `flights_not_reported` | integer (≥0) | Transparência: `NÃO INFORMADO`. Fora do denominador. |
| `flights_operated_missing_arrival` | integer (≥0) | Transparência: `REALIZADO` sem `actual_arrival`. Dado ausente explícito. |
| `flights_operated_missing_schedule` | integer (≥0) | *(novo em v1.1.0)* Transparência: `REALIZADO` com chegada real mas **sem chegada prevista** — pontualidade **indefinida**. Fora do denominador; **nunca** contado como atrasado. |
| `flights_source_total` | integer (≥0) | `operated + missing_arrival + missing_schedule + cancelled + not_reported`. Fecha a reconciliação (AC4): nenhuma linha C1 some. |

### Campos — Proveniência da métrica

| Campo | Tipo | Valor |
|---|---|---|
| `metric_id` | string | `pontualidade` |
| `metric_version` | string | `v1.1.0` |
| `metric_definition_source` | string | `docs/product/metrics-definitions.md#pontualidade` |
| `on_time_basis` | string | `arrival` |
| `on_time_threshold_minutes` | integer | `15` (inclusivo; encodado para a API não reimplementar a regra) |

### Campos — Linhagem C1 (reconciliação · AC4)

| Campo | Tipo | Notas |
|---|---|---|
| `c1_contract_version` | string | `v1.0.0`. |
| `source_year_month` | string(7) | Partição C1 que originou o registro. |
| `source_lineage` | array\<object\> | Uma entrada por arquivo de origem C1; cada objeto: `source_file`, `file_sha256`, `source_year_month`. Fixa o input exato; muda se a ANAC republicar. |

### Campos — Auditoria / Determinismo

| Campo | Tipo | Notas |
|---|---|---|
| `analytics_version` | string | Versão semver da lógica do Analytics. Com `file_sha256`, define reprodutibilidade. |
| `computed_at_utc` | timestamp | **Metadado informativo, fora** da igualdade determinística (único campo que varia entre execuções). |

### Garantias
- **Sem lógica na API (RT5):** todo cálculo (IATA, janela 15 min, denominador, taxa) já resolvido no C2. A API filtra e devolve.
- **Transparência:** `CANCELADO`, `NÃO INFORMADO`, `REALIZADO`-sem-chegada-real e `REALIZADO`-sem-chegada-prevista aparecem como contadores próprios; `flights_source_total` prova que nenhuma linha C1 sumiu. Exclusões reportadas, não apagadas.
- **Nulos nunca inventados:** `on_time_rate = null` se denominador 0; contadores default 0; `airline_name` ausente → `null`.
- **Reconciliação manual (AC4):** filtrar C1 por (`airline_icao`, rota, `reference_month`) com os `file_sha256` de `source_lineage`, contar `REALIZADO` com `actual_arrival` **e** `scheduled_arrival`, aplicar 15 min, dividir → deve bater com `on_time_rate`.
- **Determinismo (AC5):** mesmo input C1 (mesmos `file_sha256`) + mesmo `analytics_version` → C2 idêntico em todos os campos exceto `computed_at_utc`.
- **Idempotência / grão:** no máximo um registro por (`route_id`, `airline_icao`, `reference_month`); reprocessar sobrescreve, não duplica.

### Artefato de referência

Este contrato é materializado pelo artefato **`c2_punctuality.json`**.

O nome do artefato **integra a identidade deste contrato**: ele é o ponto de integração entre
Analytics e API, não um detalhe de execução. Consequência prática — quem abrir apenas este
documento consegue responder *qual contrato*, *qual versão* e *qual artefato o representa*, sem
navegar para o plano de engenharia.

Uma mudança desse nome é **mudança de interface** e exige revisão documental deste contrato,
ainda que o esquema permaneça idêntico campo a campo.

Esta seção estabelece **identidade**, não implementação. A **forma do documento** que carrega os
registros é especificada na seção seguinte; **armazenamento e framework** permanecem deferidos —
ver *Nota de escopo*.

> Ratificado pelo **ADR-0001** (Issue **GOV-002**, 2026-07-26).

### Forma do documento · normativa a partir de `v1.2.0`

As seções acima especificam os campos **de cada registro**. Esta especifica o **documento** que
os carrega — a lacuna que produtor e consumidor preencheram de formas incompatíveis até 2026-08-31.

**Forma canônica** (`v1.2.0`): objeto com envelope, nesta ordem de chaves.

```json
{
  "contract": "C2",
  "contract_version": "v1.2.0",
  "records": [ { "route_id": "SBSP-SBRJ", "…": "…" } ]
}
```

| Campo do envelope | Tipo | Valor | Propósito |
|---|---|---|---|
| `contract` | string | `C2` | O documento declara **qual** contrato carrega. |
| `contract_version` | string | `v1.2.0` | O documento declara **qual versão** carrega. |
| `records` | array\<objeto\> | registros C2 | Os registros especificados acima; ordem estável (`route_id`, `airline_icao`, `reference_month`). |

**Regras**

- O envelope declara **apenas a identidade do contrato**. Proveniência de cálculo
  (`analytics_version`, `metric_version`, `c1_contract_version`, `source_lineage`) é **por
  registro** e não se duplica no topo — duas fontes para o mesmo fato criam um modo de falha novo.
- **Versão do contrato ≠ versão da métrica.** Um documento `C2 v1.2.0` carrega registros com
  `metric_version: v1.1.0` — correto: o envelope não toca na pontualidade.
- **Forma legada tolerada:** um array JSON puro (`[ {...}, {...} ]`) continua sendo consumível
  — é o que os artefatos anteriores a 2026-08-31 contêm. Um documento legado **não declara** o
  que carrega, então o consumidor **deve** registrar essa degradação (na API: avisos `D1`/`D2`),
  jamais aceitá-la em silêncio.
- **Por que isto é normativo.** Sem versão declarada, a única defesa contra consumir um C2 de
  versão não suportada é inerte: não há o que comparar. Um contrato versionado cujo consumidor
  não consegue ler a versão do documento não está versionado na prática.

> Ratificado pelo **ADR-0002** (Issue **GOV-003**, 2026-08-31).

#### Produtor canônico × consumidor tolerante — três níveis distintos

*Aceitar* um formato, *produzi-lo* e *exigi-lo num teste* são obrigações diferentes. Confundi-las
foi o que permitiu que um artefato obsoleto fosse publicado como se fosse vigente (**GOV-005**).

| Nível | Obrigação | Sujeito |
|---|---|---|
| **1. Produzido pelo pipeline canônico** | O produtor vigente (`market-intelligence-analytics`) **deve** emitir a **versão vigente** do C2 — hoje o envelope `v1.2.0`. Emitir forma anterior é defeito do produtor, não escolha de estilo. | Analytics |
| **2. Aceito por compatibilidade** | O consumidor **deve** continuar lendo `v1.0.0`, `v1.1.0` e o array puro pré-envelope. Retrocompatibilidade é garantia do contrato e **não** é revogada por este registro. | API |
| **3. Exigido pelo teste de integração** | Um teste de integração contra o C2 **real** deve verificar o nível 1, não o nível 2: ele afirma *"o produtor vizinho está atualizado"*. | Suíte da API |

**Corolário operacional.** *Tolerar* uma forma nunca autoriza *publicá-la como vigente*. Um
consumidor que sirva um documento abaixo da versão vigente deve exigir escolha explícita de quem
opera (na API: `--allow-legacy`) e sinalizar a escolha na saída. O silêncio é o que se proíbe —
não o formato.

> Registrado em 2026-08-31 pela **GOV-005**. **Não é emenda de esquema:** nenhum campo, tipo,
> medida ou garantia mudou, e o C2 permanece em **`v1.2.0`**. Esta subseção escreve uma política
> que já era imposta por teste sem estar em lugar nenhum.

### Nota de escopo
**Armazenamento e framework permanecem deferidos** (seção 7 do plano). Este contrato descreve
campos, tipos, garantias, a **identidade do artefato** e a **forma do documento** (seções acima),
e nada além disso — em particular, não diz *onde* o artefato é persistido nem *por qual
tecnologia* é produzido.

## C3 — API → Consumidor · 🔒 `v1.0.0` (núcleo normativo congelado 2026-09-20)

> Esquema do **documento de resposta** que a API devolve ao consumidor: a comparação de
> pontualidade entre companhias numa rota, por mês, e a resposta à pergunta-âncora.
> **Princípio (RT5, na fronteira de saída):** a API **não** calcula nada. O C3 transporta os
> números do C2 verbatim e rotula explicitamente o pouco que deriva deles.
> **Escopo desta versão:** o **núcleo normativo** — ver *Nota de escopo* ao final da seção.
> **Origem:** rascunho `C3-draft v0.1.0` servido desde a Sprint 1, promovido a contrato pelo
> ADR-0003 (o rascunho vivia no `api/README.md`, o que violava **I7**).

### Grão e identidade
- **Grão:** um documento de resposta por **(par de rota) × (filtro de mês, opcional)**.
- O par de rota é **não-direcional** (`CGH-SDU`), aceito em IATA ou ICAO, em qualquer ordem.
  Dentro dele, os registros C2 direcionais são apresentados de um de dois modos, sempre declarado
  no campo `aggregation`:

| `aggregation` | Modo | O que faz |
|---|---|---|
| `none-per-direction` | **padrão** | Apresenta cada direção separadamente. Não agrega nada. |
| `count-sum` | opcional | Soma os **contadores** das duas direções por companhia e reexpressa a razão a partir dessas somas. **Agregação pura de inteiros do C2** — nunca reaplicação da métrica. |

- Por que o padrão é por direção: a pontualidade é medida na **chegada ao destino**, então
  misturar CGH→SDU com SDU→CGH conflaria duas operações — a mesma razão que faz o grão do C2 ser
  direcional.

### Forma do documento

**Forma canônica** (`v1.0.0`): objeto com envelope, seguindo a **mesma convenção do C2**
(ADR-0002).

```json
{
  "contract": "C3",
  "contract_version": "v1.0.0",
  "query":      { "route_pair": "CGH-SDU", "month": "2023-06", "combine_directions": true },
  "validation": { "status": "pass", "c2_declared_version": "v1.2.0" },
  "provenance": { "metric_id": "pontualidade", "metric_version": "v1.1.0" },
  "months":     [ { "reference_month": "2023-06", "aggregation": "count-sum" } ],
  "warnings":   []
}
```

| Campo de topo | Tipo | Propósito |
|---|---|---|
| `contract` | string | O documento declara **qual** contrato carrega — sempre `C3`. |
| `contract_version` | string | O documento declara **qual versão** carrega — com prefixo `v`, como no C2. |
| `query` | objeto | O pedido **como foi interpretado**: par normalizado, par requisitado, família de códigos, mês, modo. |
| `validation` | objeto | Resultado do gate do contrato **de entrada** (C2) — ver **G3**. Um documento servido **sempre** o carrega; sem relatório de validação não há resposta a servir (na API: `503`). |
| `provenance` | objeto | Proveniência da métrica e linhagem das fontes — ver **G3**. |
| `months` | array\<objeto\> | Um bloco por mês de referência, cada um com `aggregation` e a comparação. |
| `warnings` | array\<string\> | Degradações visíveis — ver **G6**. Vazio significa "nada a declarar". |

### Garantias

As seis garantias abaixo são o que o C3 `v1.0.0` **congela**. Nenhuma é preferência de desenho:
cada uma deriva de regra já ratificada a montante.

- **G1 — Auto-descrição.** Toda resposta declara `contract` e `contract_version`. Um consumidor
  sabe o que recebeu sem consultar a versão do produto que serviu. *(ADR-0002.)*
- **G2 — Verbatim × derivado.** `on_time_rate`, `flights_operated`, `flights_on_time` e os **5**
  contadores de transparência saem do C2 **sem recálculo, sem arredondamento e sem reescrita de
  tipo**. O conjunto de campos derivados é **fechado** nesta versão, e cada um é rotulado:

  | Derivado | O que é | O que **não** é |
  |---|---|---|
  | `rank` e a ordenação melhor→pior | ordenação do `on_time_rate` do C2 | regra de métrica |
  | `answer` / `answers` | leitura da ordenação | cálculo novo |
  | `rate_gap_vs_runner_up` | subtração de dois valores do C2, para exibição | medida do C2 |
  | taxa sob `aggregation: count-sum` | razão de duas somas de inteiros do C2 | reaplicação da janela de 15 min |

  *(RT5 — `engineering-execution-plan.md`.)*
- **G3 — Proveniência obrigatória.** Toda resposta carrega `provenance` (`metric_id`,
  `metric_version`, `metric_definition_source`, `on_time_basis`, `on_time_threshold_minutes`,
  `c1_contract_version`, `c2_contract_version`, `source_lineage`) e `validation`. **Um número sem
  a versão da métrica que o produziu não é comparável** — servir um sem o outro não é resposta
  incompleta, é resposta inútil. *(AC4.)*
- **G4 — Recusa explícita.** A resposta **declara** a resposta-âncora ou **recusa-se a
  respondê-la**; jamais a fabrica. `conclusive` (booleano) é obrigatório em todo bloco de
  resposta:

  | Situação | O que o C3 devolve |
  |---|---|
  | Um líder claro | `most_reliable`, `runner_up`, `rate_gap_vs_runner_up`, `conclusive: true` |
  | `on_time_rate` nulo (denominador 0) | companhia **excluída** e listada em `excluded_no_denominator` — ausência de medição não é um nível de confiabilidade |
  | Empate exato no `on_time_rate` | `most_reliable: null` + `tie: [...]` + `conclusive: false` — o C2 não oferece critério de desempate, e criar um seria regra de negócio |
  | Menos de 2 companhias comparáveis | `conclusive: false` — o AC3 pede uma *comparação* |
  | Todas as taxas nulas | `most_reliable: null`, `conclusive: false` |

  O empate é aferido por **igualdade exata** da fração do C2: uma tolerância seria regra de
  negócio, e regra de negócio pertence ao Analytics. *(AC3 · *nulos nunca inventados*.)*
- **G5 — Nulos nunca inventados.** `on_time_rate = null` é servido **nulo**, nunca `0` — `0`
  afirmaria 0% de pontualidade, que é um fato que a fonte não sustenta. Um contador **ausente**
  numa versão anterior do C2 é servido `null`, nunca `0` — inclusive ao somar direções, onde
  somar ausências não fabrica um zero. *(`pontualidade v1.1.0`.)*
- **G6 — Avisos visíveis.** Insumo sintético, registro posto em quarentena pela validação do C2 e
  publicação de documento em versão legada aparecem em `warnings`. **O silêncio é o que se
  proíbe** — não a degradação. *(GOV-005.)*

### Campos informativos

Campos não listados acima são **informativos**: podem mudar de texto sem emenda de contrato.
Hoje são dois, ambos em prosa: `_value_source` (nas entradas por direção) e `on_time_rate_note`
(nas entradas combinadas). A assimetria é **correta**: uma entrada combinada não é verbatim, então
afirmar `_value_source` nela seria falso.

> Ratificado pelo **ADR-0003** (Issue **GOV-009**, 2026-09-20).

### Nota de escopo

Este contrato descreve o **documento de resposta** — envelope, campos, tipos e as garantias
acima — **e nada além disso**. Permanecem **adiados** (`engineering-execution-plan.md` §7,
*"Protocolo e formato da API"*, hoje **parcialmente** decidida):

- **protocolo de transporte** (HTTP, CLI, arquivo) e **superfície de endpoints** — URLs,
  *status codes*, cabeçalhos;
- **ergonomia** de nomes e aninhamento, inclusive a divisão `answer` × `answers`;
- **paginação e filtros** além de par de rota + mês.

Essas escolhas vivem hoje no `market-intelligence-api/README.md` e só serão congeladas quando
houver **consumidor real** para validá-las — a condição que o `sprint-01-acceptance.md` §6
registrou e que **continua não satisfeita**. Congelar agora o que nenhum consumidor exercitou
repetiria o risco que o ADR-0002 §4 deixou registrado.

**Arredondamento não é do C3.** O documento transporta o `on_time_rate` em precisão plena do C2
(G2); formatar `86.75%` é apresentação, e apresentação é de quem exibe. O renderizador de texto
da CLI é **um consumidor do C3**, não parte dele.

---

### Histórico de versões

| Contrato | Versão | Data | Mudança |
|---|---|---|---|
| C0 | — | 2026-07-24 | Validado contra dado real (spike Fase 0). |
| C1 | `v1.0.0` | 2026-07-24 | Congelamento inicial do registro bruto de voo (Sprint 1, Fase 0). |
| C2 | `v1.0.0` | 2026-07-24 | Congelamento inicial do indicador de pontualidade (rota direcional × companhia × mês) aplicando `pontualidade v1.0.0` sobre `C1 v1.0.0`. |
| C2 | `v1.1.0` | 2026-07-25 | Emenda aditiva (CCR do Analytics): denominador passa a exigir `scheduled_arrival`; novo contador de transparência `flights_operated_missing_schedule`; `flights_source_total` inclui o novo bucket. Aplica `pontualidade v1.1.0`. Compatível: nenhum campo removido/renomeado. |
| C2 | `v1.1.0` | 2026-07-26 | **Revisão documental, sem mudança de esquema** (versão inalterada): nova seção *Artefato de referência* fixando `c2_punctuality.json` como identidade do contrato. Ratificado pelo ADR-0001 / GOV-002. Nenhum campo, tipo ou garantia alterado. |
| C2 | `v1.2.0` | 2026-08-31 | **Emenda aditiva no nível do documento** (ADR-0002 / GOV-003): o artefato passa a carregar o envelope `{contract, contract_version, records[]}`, tornando-se auto-descritivo. **Esquema do registro inalterado** em relação a `v1.1.0` — nenhum campo, tipo, medida ou garantia de registro mudou, e `metric_version` segue `v1.1.0`. Array JSON puro tolerado como forma legada. Compatível: um documento `v1.0.0`/`v1.1.0` continua servível. |
| C3 | `v1.0.0` | 2026-09-20 | **Congelamento inicial do núcleo normativo** do documento de resposta (ADR-0003 / GOV-009): envelope `{contract, contract_version, …}` alinhado à convenção do C2, fronteira verbatim × derivado, proveniência obrigatória, semântica de recusa, nulos e avisos (G1–G6). Promove o rascunho `C3-draft v0.1.0`, que vivia no `api/README.md` — **violação de I7** encerrada. **Protocolo, superfície de endpoints e ergonomia permanecem adiados** (ver *Nota de escopo*). |
