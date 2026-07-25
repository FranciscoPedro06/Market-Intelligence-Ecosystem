# Contratos de Dados do Ecossistema — C0 · C1 · C2 · C3

> **Responsabilidade deste documento:** ser a fonte única e versionada dos contratos de dados entre os produtos. Nenhum repositório é dono de um contrato que compartilha (governança: contratos vivem no ecossistema).
> **Guardião:** EM → Sprint Lead durante a Sprint 1.
> **Princípio contract-first:** C1 e C2 são congelados cedo para os três produtos evoluírem em paralelo sem se quebrarem.

Legenda de status: 🔒 congelado/versionado · 🧪 validado (fonte externa) · 🕓 a definir quando entregue.

| Contrato | Entre | Status |
|---|---|---|
| **C0** | Fonte ANAC/VRA → Collector | 🧪 validado na Fase 0 (Sprint 1) |
| **C1** | Collector → Analytics | 🔒 `v1.0.0` |
| **C2** | Analytics → API | 🕓 em elaboração (Analytics, Fase 0) |
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

## C2 — Analytics → API · 🕓 em elaboração

Esquema do **indicador de pontualidade** por rota × companhia × mês, aplicando a definição de `metrics-definitions.md` (`pontualidade v1.0.0`). Será congelado quando entregue pelo Analytics Engineer na Fase 0 e versionado aqui.

## C3 — API → Consumidor · 🕓 Fase 1

Perguntas por rota e a comparação de pontualidade entre companhias devolvida. A definir na Fase 1.

---

### Histórico de versões

| Contrato | Versão | Data | Mudança |
|---|---|---|---|
| C0 | — | 2026-07-24 | Validado contra dado real (spike Fase 0). |
| C1 | `v1.0.0` | 2026-07-24 | Congelamento inicial do registro bruto de voo (Sprint 1, Fase 0). |
