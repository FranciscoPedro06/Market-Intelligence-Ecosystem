# Contratos de Dados do Ecossistema — C0 · C1 · C2 · C3

> **Responsabilidade deste documento:** ser a fonte única e versionada dos contratos de dados entre os produtos. Nenhum repositório é dono de um contrato que compartilha (governança: contratos vivem no ecossistema).
> **Guardião:** EM → Sprint Lead durante a Sprint 1.
> **Princípio contract-first:** C1 e C2 são congelados cedo para os três produtos evoluírem em paralelo sem se quebrarem.

Legenda de status: 🔒 congelado/versionado · 🧪 validado (fonte externa) · 🕓 a definir quando entregue.

| Contrato | Entre | Status |
|---|---|---|
| **C0** | Fonte ANAC/VRA → Collector | 🧪 validado na Fase 0 (Sprint 1) |
| **C1** | Collector → Analytics | 🔒 `v1.0.0` |
| **C2** | Analytics → API | 🔒 `v1.0.0` |
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

## C2 — Analytics → API · 🔒 `v1.0.0` (congelado 2026-07-24)

> Esquema do **indicador de pontualidade** por **rota (direcional) × companhia × mês**, aplicando exatamente `metrics-definitions.md → pontualidade v1.0.0` sobre o registro bruto `C1 v1.0.0`.
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
| `flights_operated` | integer (≥0) | **Denominador.** `flight_status = REALIZADO` **E** `actual_arrival` não nulo. |
| `flights_on_time` | integer (≥0) | **Numerador.** Subconjunto com `(actual_arrival − scheduled_arrival) ≤ 15 min` (antecipado = pontual; +15 inclusivo). |
| `on_time_rate` | decimal[0,1] (nullable) | `flights_on_time / flights_operated`. **`null` quando denominador = 0** (nunca 0/0). Fração, não percentual; precisão plena (arredondamento é do C3). |
| `flights_cancelled` | integer (≥0) | Transparência: `CANCELADO`. Fora do denominador (fora de escopo), nunca descartado. |
| `flights_not_reported` | integer (≥0) | Transparência: `NÃO INFORMADO`. Fora do denominador. |
| `flights_operated_missing_arrival` | integer (≥0) | Transparência: `REALIZADO` sem `actual_arrival`. Dado ausente explícito. |
| `flights_source_total` | integer (≥0) | `operated + missing_arrival + cancelled + not_reported`. Fecha a reconciliação (AC4): nenhuma linha C1 some. |

### Campos — Proveniência da métrica

| Campo | Tipo | Valor |
|---|---|---|
| `metric_id` | string | `pontualidade` |
| `metric_version` | string | `v1.0.0` |
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
- **Transparência:** `CANCELADO`, `NÃO INFORMADO` e `REALIZADO`-sem-chegada aparecem como contadores; `flights_source_total` prova que nenhuma linha C1 sumiu. Exclusões reportadas, não apagadas.
- **Nulos nunca inventados:** `on_time_rate = null` se denominador 0; contadores default 0; `airline_name` ausente → `null`.
- **Reconciliação manual (AC4):** filtrar C1 por (`airline_icao`, rota, `reference_month`) com os `file_sha256` de `source_lineage`, contar `REALIZADO` com `actual_arrival`, aplicar 15 min, dividir → deve bater com `on_time_rate`.
- **Determinismo (AC5):** mesmo input C1 (mesmos `file_sha256`) + mesmo `analytics_version` → C2 idêntico em todos os campos exceto `computed_at_utc`.
- **Idempotência / grão:** no máximo um registro por (`route_id`, `airline_icao`, `reference_month`); reprocessar sobrescreve, não duplica.

### Nota de escopo
**Armazenamento, formato de serialização e framework permanecem deferidos** (seção 7 do plano) — este contrato descreve apenas campos, tipos e garantias.

## C3 — API → Consumidor · 🕓 Fase 1

Perguntas por rota e a comparação de pontualidade entre companhias devolvida. A definir na Fase 1.

---

### Histórico de versões

| Contrato | Versão | Data | Mudança |
|---|---|---|---|
| C0 | — | 2026-07-24 | Validado contra dado real (spike Fase 0). |
| C1 | `v1.0.0` | 2026-07-24 | Congelamento inicial do registro bruto de voo (Sprint 1, Fase 0). |
| C2 | `v1.0.0` | 2026-07-24 | Congelamento inicial do indicador de pontualidade (rota direcional × companhia × mês) aplicando `pontualidade v1.0.0` sobre `C1 v1.0.0`. |
