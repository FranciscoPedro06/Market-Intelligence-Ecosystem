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
| **C3** | API → Consumidor | 🕓 Fase 1 |

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
| `on_time_rate` | decimal[0,1] (nullable) | `flights_on_time / flights_operated`. **`null` quando denominador = 0** (nunca 0/0). Fração, não percentual; precisão plena (arredondamento é do C3). |
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

## C3 — API → Consumidor · 🕓 Fase 1

Perguntas por rota e a comparação de pontualidade entre companhias devolvida. A definir na Fase 1.

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
