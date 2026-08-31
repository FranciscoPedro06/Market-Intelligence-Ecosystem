# ADR-0002 — Envelope do documento C2

- **Status:** ✅ **Aceito** — ratificado pelo Sprint Lead em 2026-08-31
- **Data:** 2026-07-25 (proposta) · 2026-08-31 (ratificação)
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

**Opção A — o documento C2 carrega envelope.**

```json
{
  "contract": "C2",
  "contract_version": "v1.2.0",
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

### 3.1 O envelope constitui **C2 `v1.2.0`** (decisão de versionamento ratificada)

A questão escalada em §5 — *bump semântico ou refinamento não versionado de `v1.1.0`* — foi
decidida pelo Sprint Lead: **o envelope é a versão `v1.2.0` do contrato C2**, bump **minor**
(aditivo, retrocompatível).

Fundamento, e a distinção contra o precedente do ADR-0001:

- O ADR-0001 **não alterou nenhum byte emitido** — nomeou o que já existia. Por isso foi
  registrado como *revisão documental sem bump*.
- Este ADR **muda o que o produtor escreve**. Um consumidor passa a poder confiar numa
  garantia que antes não existia (*"o documento declara qual contrato e qual versão carrega"*).
  Garantia nova + compatibilidade preservada = **minor**, pela leitura semver que o
  ecossistema já aplicou ao promover `v1.0.0 → v1.1.0`.

Corolário: **versão de contrato ≠ versão de métrica.** Um documento `C2 v1.2.0` carrega
registros com `metric_version: v1.1.0`, e isso é correto — o envelope não tocou na
pontualidade. Era exatamente a confusão que o `metric_version` como proxy de versão de
contrato produzia (§1).

### 3.2 Conteúdo do envelope: apenas `contract` e `contract_version`

Ratificada a recomendação: o envelope declara **somente** a identidade do contrato.
`analytics_version`, `metric_version`, `c1_contract_version` e `source_lineage` continuam
**por registro**. Motivo: são proveniência do *cálculo daquele registro*, não do documento;
duplicá-los no topo criaria duas fontes para o mesmo fato e um modo de falha novo
(divergirem entre si).

### 3.3 `D1`/`D2` permanecem `warning` enquanto o array puro for tolerado

O ADR previa avaliar a promoção para `error`. Decisão: **permanecem `warning`**. Elevar agora
recusaria todo artefato legado no mesmo ato que estabelece a forma canônica — punir o passado
para inaugurar a regra. O gatilho para a promoção é explícito e fica registrado: **quando o
array puro deixar de ser tolerado** (uma decisão futura, com seu próprio ADR).

## 4. Consequências

**Positivas**
- `D2_CONTRACT_VERSION` passa a poder **recusar** um documento de versão não suportada.
- `pass_with_warnings (… 2 warning)` → `pass`: os avisos remanescentes voltam a significar algo.
- `c2_contract_version` deixa de ser `null` em toda resposta e em `GET /meta`.

**Negativas / custo aceito**
- `analyze.py` muda a forma de saída — **a primeira mudança de escrita no produtor desde o freeze
  do C2**. Exige reexecução e reconferência do artefato (nenhuma medida muda; o digest do arquivo
  muda).
- **`analytics_version` sobe `1.1.0 → 1.2.0`, e isso altera um campo de cada registro.** Não é
  cosmético e é obrigatório: a garantia de determinismo do C2 é *"mesmo input C1 + mesmo
  `analytics_version` → C2 idêntico exceto `computed_at_utc`"*. Manter `1.1.0` faria a mesma
  versão da lógica produzir dois documentos diferentes (array puro e envelope) — a garantia
  passaria a ser falsa. Consequência prática: o registro emitido hoje **não** é byte-idêntico ao
  de 2026-07-25 em `analytics_version`; é idêntico em **todas as 8 medidas e todas as
  dimensões** (verificado — §6).
- **Terceira versão do C2 em ~5 semanas.** É movimento em documento de Tier 3
  (`documentation-architecture.md` §6), que deveria mudar por revisão de contrato, não por
  rodada. Aceito porque cada bump veio de um defeito real encontrado contra dado real
  (`v1.1.0`: voo não mensurável no denominador; `v1.2.0`: gate de versão inerte) — mas é sinal
  a observar: se o C2 bumpar de novo antes da Fase 1, o problema não são os bumps, é o contrato
  ter sido congelado antes de existir consumidor.
- Todo artefato C2 existente passa a ser forma legada. Mitigação: a API continua aceitando array
  puro, então nada quebra — mas o `contracts.md` deve dizer qual é a forma canônica e qual é tolerada.
- `RECONCILIATION.md` e o exemplo do README da API precisam ser reconferidos (o conteúdo servido
  não muda; a linha `C2=None` do exemplo passa a mostrar `v1.1.0`).

## 5. Propagação a jusante

| Alvo | Mudança |
|---|---|
| `docs/ecosystem/contracts.md` §C2 | ✅ Nova seção *Forma do documento*: envelope canônico + array puro tolerado como legado; *Nota de escopo* ajustada |
| `docs/ecosystem/contracts.md` — Histórico de versões | ✅ Decidido em §3.1: **C2 `v1.2.0`** (bump minor, aditivo, nível documento). Linha registrada no histórico |
| `docs/ecosystem/decisions/README.md` | ✅ Índice: ADR-0002 → `Aceito` |
| `docs/README.md` | ✅ Índice (I5): C2 `v1.2.0` |
| `docs/engineering/engineering-execution-plan.md` §7 | ✅ *"Formato dos conjuntos de dados intermediários"* → **decidida** (ADR-0001 identidade + ADR-0002 forma). Armazenamento e framework seguem adiados |
| `market-intelligence-analytics/src/analyze.py` | ✅ `build_document()` emite o envelope; `ANALYTICS_VERSION → 1.2.0` (§4); `README.md` reflete a nova forma |
| `market-intelligence-api/src/c2_validation.py` | ✅ `SUPPORTED_C2_VERSIONS` inclui `v1.2.0`. Severidade de `D1`/`D2` **inalterada** — ver §3.3 |
| `market-intelligence-api/tests/self_test.py` | ✅ Nova classe `TestC2DocumentEnvelope` (5 casos) + o teste contra o C2 real passa a exigir envelope declarado. 54 → 60 casos |
| `market-intelligence-api/README.md` | ✅ *Observação para o Analytics* removida (resolvida); exemplo mostra `C2=v1.2.0` / `pass` |
| `docs/engineering/sprints/sprint-01-acceptance.md` §6 | ⏳ **Não tocado aqui.** O follow-up *"Envelope de C2"* está resolvido de fato, mas a forma admissível de editar um registro de aceite depende de **GOV-006** (é log append-only ou derivado?), que precede **GOV-001**. Registrar em ADR e não editar o log é a leitura conservadora de **I4** |

## 6. Rastreabilidade

- Auditoria de integração 2026-07-25 — divergência ⚠1 e inconsistência estrutural S4.
- Issue de governança: **GOV-003**.
- Registro anterior do problema: `api/README.md` (*Observação para o Analytics*) e
  `sprint-01-acceptance.md` §6 — onde estava classificado como *"cosmético"*. Este ADR
  **discorda dessa classificação**: o efeito é a inutilização de um gate de contrato.
- Verificação **antes** (2026-07-25): `python src/serve.py --input …/c2_punctuality.json --route
  CGH-SDU --month 2023-06` → `C2=None`, `pass_with_warnings`, 2 avisos (`D1`, `D2`).

### Ratificação — 2026-08-31 (Sprint Lead)

Decidido: **Opção A**, versionada como **C2 `v1.2.0`** (§3.1), envelope restrito a
`contract`/`contract_version` (§3.2), `D1`/`D2` seguem `warning` (§3.3).

Evidência da execução, contra o C1 real (9.527 linhas, CGH↔SDU, abr–jun/2023):

| Verificação | Resultado |
|---|---|
| Mesmo comando de antes, após a mudança | `C2=v1.2.0` · `pass (25 valid, 0 quarantined, 0 error, **0 warning**)` |
| Nenhum número se moveu | 25 registros comparados campo a campo com o artefato de 2026-07-25: **0 diferenças** fora de `analytics_version` e `computed_at_utc` |
| Determinismo (AC5) preservado | duas execuções com o mesmo `--computed-at` → sha256 idêntico (`8d912cdb…`) |
| Suíte da API | 60/60 casos (era 54/54); os 6 novos exercitam o envelope, o legado e a recusa por versão desconhecida |
| Gate `D2` deixa de ser inerte | documento com `contract_version: v9.9.9` dentro do envelope → `refused`, 0 registros servidos (teste `test_unsupported_version_in_an_envelope_refuses_the_document`) |

O último item é o que justificava o ADR: o gate existia e nunca podia disparar em produção.

**Efeito colateral registrado:** a correção fecha parte da **GOV-004** (as referências a
`C2 v1.0.0` em `analyze.py`, `c2_validation.py` e `serve.py:520` — esta última visível ao
consumidor em `GET /` e `GET /meta`). A GOV-004 **permanece aberta** para o restante
(`collector/EVIDENCE.md`, `CONTRACT-CHANGE-REQUEST.md` §1, `hashlib` no README do Analytics,
timestamps de `RECONCILIATION.md` e `PROVENANCE.md`).
