# Issues de Governança — registro aberto

> ⚠️ **ARTEFATO TEMPORÁRIO DE BOOTSTRAP — não faz parte da arquitetura do ecossistema.**
>
> **Por que existe:** o `gh` CLI não está instalado/autenticado no ambiente atual, então as
> pendências de governança da auditoria de 2026-07-25 não puderam ser abertas como Issues. Este
> arquivo as preserva até que isso seja possível. Uma limitação de ferramenta **não** deve alterar a
> arquitetura documental.
>
> **Fonte de verdade pretendida:** Issues do GitHub em
> `FranciscoPedro06/Market-Intelligence-Ecosystem`.
>
> **Princípio que justifica isso:** documentos versionados (`vision`, `contracts`, `metrics-definitions`,
> ADRs) representam **conhecimento**; Issues representam **trabalho e decisões pendentes**. Misturar
> as duas responsabilidades cria duplicação e sincronização manual.
>
> **Fim de vida:** ao abrir GOV-001…006 como Issues reais (preservando o conteúdo de cada uma),
> **este arquivo é removido**. Ele não é migrado, arquivado nem mantido em paralelo.
>
> **Deliberadamente não indexado:** ausente de `docs/README.md` e da estrutura em
> `documentation-architecture.md` §1/§2. É uma **exceção consciente e transitória** ao invariante
> **I5** — o índice reflete a arquitetura estável, não um mecanismo temporário de ambiente.
>
> **Dono:** Sprint Lead.
>
> **Regra que sobrevive à remoção:** uma Issue de governança **não é** tarefa de engenharia. Ela
> registra uma pendência de decisão. Somente após a decisão (ADR, quando aplicável) é que Collector,
> Analytics e API recebem tarefas específicas.

Legenda de severidade: 🔴 bloqueador · 🟠 alto · 🟡 médio.
Legenda de status: 🟡 **Aberta** · 🔵 **Em decisão** · ✅ **Fechada** · ⛔ **Descartada**.

---

## Origem

**Auditoria de integração do ecossistema — 2026-07-25**, cobrindo os quatro repositórios após o
commit `7217002` (*promover C2 v1.0.0 → v1.1.0*).

Resultado da auditoria: **o pipeline está íntegro.** O Collector produz exatamente o C1
especificado (27 colunas, nomes e ordem idênticos, chaves naturais únicas), o Analytics implementa
corretamente o C2 `v1.1.0`, a API consome essa versão, os 54 testes passam, a reconciliação fecha
ponta a ponta (`9.054 + 39 + 434 + 0 = 9.527`) e a reexecução é byte-idêntica. **Nenhuma divergência
de lógica de negócio, nome de campo, tipo ou métrica foi encontrada entre os repositórios.**

As pendências abaixo são de **governança e consistência documental**, com duas exceções de efeito
funcional sinalizadas em GOV-005.

---

## Ordem de execução

A ordem não é a numérica — é a de dependência:

```
GOV-006 ──► GOV-001          (corrigir o mecanismo antes do sintoma)
ADR-0001 ──► ADR-0002        (o nome do artefato antecede a sua forma)
     └──────────┴──────────► GOV-004, GOV-005
```

- **GOV-006 antes de GOV-001:** sem fechar a lacuna de propagação, o próximo contrato emendado
  reproduz exatamente o GOV-001.
- **GOV-004 e GOV-005 depois dos ADRs:** ambos dependem do nome e da forma do artefato decididos.

---

## GOV-001 🔴 — Reconciliar `sprint-01-acceptance.md` com `pontualidade v1.1.0`

- **Status:** 🟡 Aberta · **Dono:** Sprint Lead · **Repo:** Market-Intelligence-Ecosystem
- **Bloqueador:** sim — é a única pendência classificada como bloqueadora.

### Problema

O commit `7217002` atualizou apenas `docs/README.md`, `contracts.md` e `metrics-definitions.md`.
O registro de aceite **nunca foi tocado após `3c6c4ef`** e hoje contradiz o contrato vigente — no
repositório que é a fonte única da verdade.

O ponto mais grave é o §4, *Nota de transparência — `0%` de ACN/PTB*:

> *"Pela definição congelada entram no denominador (têm chegada real) mas não há como provar
> pontualidade → 0 pontuais. **O número é honesto, nunca inventado.**"*

`metrics-definitions.md v1.1.0` e o CCR do Analytics afirmam **o oposto**: contar esses voos era
inventar um fato e violava a garantia *"nulos nunca inventados"*. A governança sustenta hoje as
duas proposições contrárias simultaneamente.

### Itens obsoletos

| Local | Estado atual | Correto sob `v1.1.0` |
|---|---|---|
| §1 — cabeçalho | `pontualidade v1.0.0` | `v1.1.0` |
| §1 — 2023-04 | AZU **80,23%** | AZU **81,05%** |
| §1 — 2023-05 | TAM **84,96%** | TAM **85,03%** |
| §1 — 2023-06 | TAM **87,58%** · AZU **84,18%** | TAM **87,71%** · AZU **85,20%** |
| §1 — nota | *"ACN/PTB aparecem com 0% marginal"* | aparecem como `n/a` (denominador 0) |
| §2 — AC6 | `contracts.md` C1/C2 `v1.0.0` | C1 `v1.0.0`, C2 `v1.1.0` |
| §4 — DoD | `metrics-definitions.md v1.0.0`; `contracts.md v1.0.0` | `v1.1.0` |
| §4 — DoD | 3 contadores de transparência | **5** contadores |
| §4 — Nota ACN/PTB | afirma que contar é honesto | contradiz `v1.1.0` — reescrever ou marcar superado |
| §6 — follow-up | `pontualidade v1.1.0` como *"candidato"* | já aplicado em 2026-07-25 |

> Os percentuais corretos foram obtidos executando `serve.py --combine` sobre o C2 real, por mês.
> **A ordenação das companhias não muda em nenhum mês** — a conclusão da Sprint 1 permanece válida.
> O que não é mais reproduzível são os números publicados.

### Decisão pedida

Adendo datado de emenda (preserva o aceite como registro histórico) **vs.** edição in-place com
nota de supersessão. A escolha depende de GOV-006: se o aceite for classificado como log
append-only, só o adendo é admissível.

### Critério de fechamento

§4 não afirma mais que contar voos não mensuráveis é honesto; os 5 percentuais corrigidos; §1, AC6,
DoD e §6 refletem `v1.1.0`; o documento declara explicitamente sob qual versão da métrica o aceite
foi concedido.

---

## GOV-002 🟠 — Definir o artefato canônico do C2

- **Status:** 🔵 **Em decisão** → **[ADR-0001](../ecosystem/decisions/0001-artefato-canonico-c2.md)** (Proposto)
- **Dono:** Sprint Lead · **Repos afetados:** analytics, api, ecosystem

### Problema

Coexistem `c2_punctuality.json` (default do produtor, citado por 4 fontes) e `c2_on_time.csv`
(citado pelo README da API e por `self_test.py:71`, e que contém **JSON** apesar da extensão).
Ambos são gitignored e nenhum é rastreado. Num clone limpo, o comando documentado da API falha e
5 testes de dado real são pulados em silêncio.

Reclassificado como **decisão adiável tomada** (`engineering-execution-plan.md` §7 — *formato dos
conjuntos de dados intermediários*), portanto gera ADR, não apenas Issue.

### Critério de fechamento

ADR-0001 aceito; nome único em `contracts.md`; README da API e `self_test.py:71` alinhados;
`c2_on_time.csv` descontinuado.

---

## GOV-003 🟠 — Formalizar o envelope do documento C2

- **Status:** 🔵 **Em decisão** → **[ADR-0002](../ecosystem/decisions/0002-envelope-documento-c2.md)** (Proposto)
- **Dono:** Sprint Lead · **Repos afetados:** analytics, api, ecosystem
- **Depende de:** ADR-0001

### Problema

O produtor emite array JSON puro; a fixture da API usa envelope
`{contract, contract_version, records[]}`. Produção e teste percorrem caminhos diferentes.
Efeito: `c2_contract_version: null` em toda resposta, 2 avisos permanentes, e o gate
`D2_CONTRACT_VERSION` — a única defesa contra um C2 de versão não suportada — nunca dispara.

O aceite §6 classificou isso como *"cosmético"*. **ADR-0002 discorda:** o efeito é a inutilização
de um gate de contrato.

### Critério de fechamento

ADR-0002 aceito; `contracts.md` especifica a forma do documento; decidido se o envelope constitui
**C2 `v1.2.0`** ou refinamento não versionado; `c2_contract_version` deixa de ser `null`.

---

## GOV-004 🟡 — Limpeza de referências residuais a `v1.0.0`

- **Status:** 🟡 Aberta · **Dono:** engenheiro de cada repo, após os ADRs
- **Natureza:** execução, sem decisão arquitetural.

| Arquivo | Linha | Texto residual |
|---|---|---|
| `api/src/serve.py` | 520 | *"full **C2 v1.0.0** contract validation report"* — **visível ao consumidor** em `GET /` e `GET /meta` |
| `api/src/serve.py` | 8 | *"validate it against the frozen **C2 v1.0.0** contract"* |
| `api/src/c2_validation.py` | 2, 21 | docstring e *"Reference: … C2 — Analytics → API — **v1.0.0**"* |
| `analytics/src/analyze.py` | 9-10 | *"C2 **v1.0.0** (output)"* e *"pontualidade **v1.0.0**"* — o código implementa `v1.1.0` |
| `collector/EVIDENCE.md` | 5 | *"antes do Analytics aplicar `pontualidade **v1.0.0**`"* |
| `analytics/docs/CONTRACT-CHANGE-REQUEST.md` | §1 | *"21 de seus **22** voos"* → ACN tem **21** voos, todos os 21 indefinidos |

Menores, no mesmo lote: `analyze.py:13` e README do Analytics listam `hashlib` entre as stdlib
usadas (não está importado); `RECONCILIATION.md:7` documenta `--computed-at 2026-07-24T00:00:00Z`
mas o artefato carrega `2026-07-25T00:00:00Z`; `PROVENANCE.md`, `stats.json` e `c1_flights.csv`
citam três `ingested_at_utc` distintos (`01:48:25`, `01:55:01`, `01:56:08`), ou seja, a evidência
do Collector não vem de uma execução única.

> **Não incluir aqui** as ocorrências de `v1.0.0` nas fixtures (`api/input/sample_c2.json`,
> `TEST_META`, `make_record`) nem em `SUPPORTED_C2_VERSIONS`, no histórico de versões dos contratos
> e no campo `c1_contract_version`. Todas são **deliberadas e corretas** — cobrem
> retrocompatibilidade e registro histórico.

### Critério de fechamento

Nenhuma referência a `v1.0.0` descrevendo o comportamento **vigente**; o endpoint `/validation`
anuncia a versão correta; correções menores aplicadas.

---

## GOV-005 🟠 — Estratégia de teste de integração com dado real

- **Status:** 🟡 Aberta · **Dono:** Sprint Lead + API Engineer · **Depende de:** ADR-0001
- **Severidade elevada de 🟡 para 🟠** pela auditoria: há efeito funcional, não apenas documental.

### Problema

Dois achados com consequência real:

1. **Falso positivo na suíte.** `api/tests/self_test.py:637` usa
   `@unittest.skipUnless(os.path.exists(ANALYTICS_C2))`. Num clone limpo o arquivo não existe
   (`output/` é gitignored), os 5 testes contra C2 v1.1.0 real são pulados e a suíte reporta
   **"Ran 54 tests — OK"**. Ausência de dado vira silêncio, não falha. A verificação de integração
   só é real nesta máquina.
2. **Default obsoleto.** `api/input/c2_punctuality.json` é um artefato **C2 `v1.0.0`**
   (`metric_version: v1.0.0`, `analytics_version: 1.0.0`, sem
   `flights_operated_missing_schedule`) e é o `DEFAULT_INPUT` de `serve.py:30`. Rodar
   `python src/serve.py` sem argumentos publica hoje **`ACN 0.00%`** — exatamente o fato fabricado
   que a emenda `v1.1.0` existe para eliminar — e a validação **aprova**, porque `v1.0.0` é versão
   suportada. Não é rastreado pelo git, portanto é armadilha local, não do repositório.

### Decisão pedida

Opções não exclusivas: (a) versionar uma fixture C2 **`v1.1.0`** via force-add, como já foi feito
com `sample_c2.json`; (b) trocar o `skipUnless` por falha explícita quando o dado real é esperado
mas ausente; (c) gerar o C1/C2 real no CI antes da suíte; (d) fazer `serve.py` recusar-se a rodar
com default ausente em vez de servir o que estiver no diretório.

### Critério de fechamento

A suíte não pode passar verde sem ter validado um C2 `v1.1.0`; o caminho default não pode servir
artefato de versão anterior sem sinalização inequívoca.

---

## GOV-006 🟡 — Incluir documentos de sprint na propagação de mudança

- **Status:** 🟡 Aberta · **Dono:** EM (dono do `documentation-architecture.md`)
- **Precede:** GOV-001 — **é a causa-raiz, não um item paralelo.**

### Problema

A tabela de propagação (`documentation-architecture.md` §5) mapeia
`metrics-definitions → contracts → analytics/README, api/README` e
`contracts.md → <repo>/README dos produtos afetados`. **Nenhuma linha inclui os documentos de
sprint.** O registro de aceite também não aparece no grafo de dependências (§3), não está entre os
papéis da §2, e não está entre os logs append-only protegidos por **I4** (que lista apenas
`decisions/` e `spikes/`).

Foi exatamente por essa lacuna que o `sprint-01-acceptance.md` escapou da promoção `v1.1.0`.
Corrigir apenas GOV-001 trata o sintoma: **a próxima emenda de contrato reproduz o mesmo defeito.**

Lacuna correlata detectada ao criar os primeiros ADRs: **I4 declara logs imutáveis sem prever campo
mutável**, o que torna ambígua a transição `Proposto → Aceito` de um ADR. Ver a ressalva em
`decisions/README.md`.

### Decisão pedida

1. O registro de aceite de sprint é **log append-only** (protegido por I4, corrigível apenas por
   adendo) ou **derivado** (sujeito a propagação e edição)? A resposta determina a forma admissível
   de GOV-001.
2. Acrescentar à §5 a linha: *mudança em `contracts.md` ou `metrics-definitions.md` → revisar os
   registros de sprint que citam a versão alterada.*
3. Ratificar (ou rejeitar) o campo `Status` como única exceção admitida a I4.

### Critério de fechamento

`documentation-architecture.md` §2, §3 e §5 contemplam os documentos de sprint; a transição de
status de ADR está ratificada; uma emenda de contrato futura força, por regra escrita, a revisão
dos registros de sprint afetados.

---

## Estado consolidado

| ID | Sev. | Título | Status | Instrumento |
|---|---|---|---|---|
| GOV-006 | 🟡 | Propagação para documentos de sprint | 🟡 Aberta | Issue (precede GOV-001) |
| GOV-001 | 🔴 | Reconciliar `sprint-01-acceptance.md` | 🟡 Aberta | Issue |
| GOV-002 | 🟠 | Artefato canônico do C2 | 🔵 Em decisão | ADR-0001 |
| GOV-003 | 🟠 | Envelope do documento C2 | 🔵 Em decisão | ADR-0002 |
| GOV-004 | 🟡 | Referências residuais a `v1.0.0` | 🟡 Aberta | Issue |
| GOV-005 | 🟠 | Teste de integração com dado real | 🟡 Aberta | Issue |

**Nenhuma tarefa foi distribuída aos repositórios.** Collector, Analytics e API permanecem sem ação
pendente até que o Sprint Lead ratifique ADR-0001, ADR-0002 e decida GOV-006 e GOV-001.
