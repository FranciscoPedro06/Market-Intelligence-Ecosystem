# ADR-0002 — Envelope do documento C2

- **Status:** 🟡 **Proposto** — aguarda ratificação do Sprint Lead
- **Data:** 2026-07-25
- **Decisor:** Sprint Lead (guardião dos contratos)
- **Proponente:** auditoria de integração do ecossistema (2026-07-25)
- **Decisão adiável correspondente:** `engineering-execution-plan.md` §7 — *"⏳ Formato dos conjuntos de dados intermediários"*
- **Supera:** —
- **Depende de:** ADR-0001 (nome do artefato)

## 1. Contexto

O `contracts.md` §C2 especifica os campos **de cada registro**, mas **nada** sobre a forma do
**documento** que os carrega. Produtor e consumidor preencheram a lacuna de formas incompatíveis:

| Lado | Forma emitida / esperada |
|---|---|
| **Produtor** — `analytics/src/analyze.py:293` | `json.dump(records, …)` → **array JSON puro**: `[ {...}, {...} ]` |
| **Consumidor** — `api/src/serve.py:57-69` (`load_c2`) | Aceita **as duas**: `{"records": [...]}` **ou** `[...]` |
| **Fixture do consumidor** — `api/input/sample_c2.json` | Usa **envelope**: `{"contract": "C2", "contract_version": "v1.0.0", "records": [...]}` |

Ou seja: a API é tolerante, mas **a sua própria fixture usa uma forma que o produtor real nunca
emite**. Os testes exercitam o caminho com envelope; a produção roda pelo caminho sem envelope.

**Consequência verificada.** Com array puro, `meta = {}`, e `c2_validation.validate_document()`
emite dois avisos permanentes:

- `D1_CONTRACT_NAME` (warning) — *"document does not declare 'contract'; assuming C2"*
- `D2_CONTRACT_VERSION` (warning) — *"cannot confirm it is one of v1.0.0, v1.1.0"*

E `_provenance()` (`serve.py:227`) devolve `c2_contract_version: null`. Reproduzido contra o C2
real: `Metric: pontualidade v1.1.0 | … | C2=None` e `pass_with_warnings (25 valid, 0 quarantined,
0 error, 2 warning)`. O `null` aparece em **toda** resposta servida e em `GET /meta`.

**O que isso custa de verdade.** O `D2_CONTRACT_VERSION` é o **único gate que recusa um documento
C2 de versão não suportada** — e em produção ele nunca dispara como erro, porque não há versão
declarada para comparar. A confirmação de versão hoje é **indireta**, via `metric_version` de cada
registro. Isso é um proxy razoável (`D4_PROVENANCE_MIXED` recusa registros que discordem entre si,
e `P5_MISSING_V110_COUNTER` avisa quando um registro v1.1.0 chega sem o contador novo) — mas
`metric_version` é a versão **da métrica**, não **do contrato**. Uma futura `C2 v1.2.0` que mude
dimensões sem tocar a métrica passaria pelo gate sem ser notada.

Ambos os lados já documentaram a lacuna honestamente (`api/README.md`, seção *Observação para o
Analytics*; `sprint-01-acceptance.md` §6, follow-up *"Envelope de C2 (cosmético)"*). Nenhum dos dois
está errado: **o contrato não pede envelope nenhum.** Falta a decisão.

## 2. Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| **A — Adotar envelope** `{contract, contract_version, records[]}` | Ativa `D2` como gate real; alinha produtor à fixture que a API já testa; elimina `c2_contract_version: null` de toda resposta; a versão do contrato passa a viajar **com o dado**, não com o repo | Produtor muda (`analyze.py`); artefatos existentes viram forma legada; ligeiramente mais verboso |
| **B — Fixar array puro no contrato** | Zero mudança de código; formato mais simples; `metric_version` + `D4` cobrem a maior parte do risco | `D1`/`D2` viram código morto em produção; 2 avisos permanentes em toda resposta (ruído que dessensibiliza); a fixture da API precisa ser reescrita sem envelope para testar o caminho real; versão do contrato nunca é auto-declarada |
| **C — Manter tolerância, não decidir** | Nenhum esforço | É o estado atual e a causa do achado; perpetua "produção e teste usam caminhos diferentes" |

## 3. Decisão

**Recomendação: Opção A — o documento C2 passa a carregar envelope.**

```json
{
  "contract": "C2",
  "contract_version": "v1.1.0",
  "records": [ { "route_id": "SBSP-SBRJ", "…": "…" } ]
}
```

Três razões, em ordem de peso:

1. **Ativa a única defesa contra versão errada.** Um contrato versionado cujo consumidor não
   consegue ler a versão do documento não está versionado na prática.
2. **Fecha a divergência produção × teste.** A fixture da API já usa envelope; adotar A alinha o
   produtor ao que já é testado, em vez de reescrever a fixture para o caminho não testado.
3. **É a forma que a auto-descrição do ecossistema já pressupõe.** `metric_version`,
   `c1_contract_version`, `analytics_version` e `source_lineage` viajam com o dado; a versão do
   **contrato** ser a única exceção é incoerente.

**Não é breaking change.** `load_c2` já aceita as duas formas (`serve.py:61-68`), então a API
consome o novo documento sem nenhuma alteração. A mudança é aditiva do lado do produtor.

> **Falta para ratificar:** decidir se `contract_version` é a versão do **contrato** (`v1.1.0`)
> declarada pelo produtor, ou se o Analytics deve emitir também `analytics_version` no envelope.
> Recomendação: apenas `contract`/`contract_version` no envelope — `analytics_version` é por
> registro e já existe.

## 4. Consequências

**Positivas**
- `D2_CONTRACT_VERSION` passa a poder **recusar** um documento de versão não suportada.
- `pass_with_warnings (… 2 warning)` → `pass`: os avisos remanescentes voltam a significar algo.
- `c2_contract_version` deixa de ser `null` em toda resposta e em `GET /meta`.

**Negativas / custo aceito**
- `analyze.py` muda a forma de saída — **a primeira mudança de escrita no produtor desde o freeze
  do C2**. Exige reexecução e reconferência do artefato (o conteúdo dos registros não muda; o
  digest do arquivo muda).
- Todo artefato C2 existente passa a ser forma legada. Mitigação: a API continua aceitando array
  puro, então nada quebra — mas o `contracts.md` deve dizer qual é a forma canônica e qual é tolerada.
- `RECONCILIATION.md` e o exemplo do README da API precisam ser reconferidos (o conteúdo servido
  não muda; a linha `C2=None` do exemplo passa a mostrar `v1.1.0`).

## 5. Propagação a jusante

| Alvo | Mudança |
|---|---|
| `docs/ecosystem/contracts.md` §C2 | Nova subseção *Forma do documento*: envelope canônico + array puro tolerado como legado; atualizar a *Nota de escopo* |
| `docs/ecosystem/contracts.md` — Histórico de versões | Avaliar se o envelope é **C2 `v1.2.0`** (aditivo, nível documento) ou refinamento não versionado de `v1.1.0`. **Questão para o Sprint Lead** — a auditoria não decide isso |
| `docs/engineering/engineering-execution-plan.md` §7 | Marcar *"Formato dos conjuntos de dados intermediários"* como decidida (ADR-0001 + ADR-0002) |
| `market-intelligence-analytics/src/analyze.py` | Emitir envelope; `README.md` refletir a nova forma |
| `market-intelligence-api/src/c2_validation.py` | Nenhuma mudança de lógica. Avaliar se `D1`/`D2` sobem de `warning` para `error` quando o envelope passar a ser obrigatório |
| `market-intelligence-api/README.md` | Remover a *Observação para o Analytics*; atualizar `C2=None` no exemplo |
| `docs/engineering/sprints/sprint-01-acceptance.md` §6 | Follow-up *"Envelope de C2"* → resolvido por este ADR (coordenar com **GOV-001**) |

## 6. Rastreabilidade

- Auditoria de integração 2026-07-25 — divergência ⚠1 e inconsistência estrutural S4.
- Issue de governança: **GOV-003**.
- Registro anterior do problema: `api/README.md` (*Observação para o Analytics*) e
  `sprint-01-acceptance.md` §6 — onde estava classificado como *"cosmético"*. Este ADR
  **discorda dessa classificação**: o efeito é a inutilização de um gate de contrato.
- Verificação: `python src/serve.py --input …/c2_punctuality.json --route CGH-SDU --month 2023-06`
  → `C2=None`, `pass_with_warnings`, 2 avisos (`D1`, `D2`).
