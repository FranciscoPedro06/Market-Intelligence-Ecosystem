# ADR-0003 — Núcleo normativo do C3

- **Status:** ✅ **Aceito** — ratificado pelo Sprint Lead em 2026-09-20
- **Data:** 2026-09-20 (proposta e ratificação)
- **Decisor:** Sprint Lead (guardião dos contratos)
- **Proponente:** auditoria de governança GOV-009
- **Decisão adiável correspondente:** `engineering-execution-plan.md` §7 — *"⏳ Protocolo e formato da API além de 'responde à pergunta por rota'"*
- **Supera:** —
- **Depende de:** ADR-0002 (convenção de envelope do ecossistema)

## 1. Contexto

O `contracts.md` reserva uma seção para o C3 e **não a preenche**:

> `contracts.md:240-242` — *"## C3 — API → Consumidor · 🕓 Fase 1 · Perguntas por rota e a
> comparação de pontualidade entre companhias devolvida. **A definir na Fase 1.**"*

Enquanto isso, a forma da resposta **já existe, já é servida e já é versionada** — em outro lugar:

| Onde | O que está escrito lá |
|---|---|
| `api/README.md:45-59` | Tabela normativa *"Saída — C3-draft (Fase 1)"*: quais campos saem **verbatim do C2** e quais são **derivados** |
| `api/README.md:72-80` | Tabela normativa das **recusas**: `conclusive: false`, empate, `excluded_no_denominator` |
| `api/src/serve.py:34` | `C3_RESPONSE_VERSION = "0.1.0"` — o contrato compartilhado tem número de versão **dentro de um produto** |
| `api/src/serve.py:293-294` | `"response_contract": "C3-draft"`, `"response_version": C3_RESPONSE_VERSION` |
| `api/src/serve.py:539` | `GET /meta` publica `output_contract: {"name": "C3-draft", …}` ao consumidor |
| `api/tests/self_test.py:525` | Um teste **fixa** o nome `C3-draft` — a API impõe o contrato a si mesma |

### 1.1 O achado: violação de I7

A invariante **I7** diz: *"Contrato é governança do ecossistema; **nenhum repo é dono de contrato
compartilhado**."* Hoje o `market-intelligence-api` **define, versiona, publica e testa** o C3
sozinho, e o documento de governança que deveria ser a sede diz *"a definir"*. Não é omissão de
fato — é fato registrado no lugar errado. Um consumidor futuro que leia `contracts.md` para saber
o que vai receber não encontra nada; quem quiser saber precisa ler o código da API.

### 1.2 Duas convenções de auto-descrição no mesmo ecossistema

O ADR-0002 ratificou, há sete semanas, que **o documento declara o que carrega**, com a forma
`{"contract": "C2", "contract_version": "v1.2.0"}` (`contracts.md` §C2 → *Forma do documento*).
A resposta da API declara a mesma coisa com **chaves diferentes e formato de valor diferente**:

| | Documento C2 (ratificado) | Resposta da API (hoje) |
|---|---|---|
| Chave do nome | `contract` | `response_contract` |
| Chave da versão | `contract_version` | `response_version` |
| Formato do valor | `"v1.2.0"` — com prefixo | `"0.1.0"` — **sem** prefixo |

Nenhuma das duas está errada isoladamente. Juntas, obrigam todo consumidor do ecossistema a
aprender duas convenções para o mesmo fato — e a segunda nasceu **antes** de a primeira ser
ratificada, então é divergência por antiguidade, não por discordância.

### 1.3 Uma responsabilidade atribuída a um contrato inexistente

`api/README.md:263-264` afirma:

> *"o JSON carrega o `on_time_rate` em precisão plena do C2 — **arredondamento é responsabilidade
> do C3**."*

Mas **o JSON servido *é* o documento C3**. A frase delega a si mesma uma responsabilidade, e só
pôde ser escrita porque ninguém jamais escreveu o que o C3 é. Enquanto o C3 não existir como
contrato, a frase não é verificável nem falsificável.

### 1.4 A tensão real: congelar sem consumidor

O `sprint-01-acceptance.md:88` condiciona explicitamente: *"Congelar C3 quando o consumidor
validar a forma da resposta."* **Não existe consumidor.** E o próprio ADR-0002 §4 registrou o
risco com todas as letras:

> *"…se o C2 bumpar de novo antes da Fase 1, o problema não são os bumps, é o contrato ter sido
> congelado antes de existir consumidor."*

Ignorar isso para congelar o C3 inteiro repetiria conscientemente o erro que o ecossistema acabou
de documentar. Esta é a restrição que molda a decisão — não um detalhe a contornar.

## 2. Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| **A — Congelar o C3 completo** (envelope + campos + superfície de endpoints + protocolo) | Um documento responde tudo; o consumidor futuro encontra URL, *status codes* e forma no mesmo lugar | Congela **ergonomia que nenhum consumidor validou** — exatamente o risco de ADR-0002 §4; invade dois itens do §7 deliberadamente adiados (protocolo, superfície); a primeira demanda de um consumidor real forçaria `v2.0.0` |
| **B — Congelar apenas o núcleo normativo** (auto-descrição, fronteira verbatim × derivado, proveniência obrigatória, semântica de recusa, nulos, avisos) e **declarar explicitamente o que fica adiado** | Encerra a violação de I7 **hoje**; tudo que se congela **deriva de regra já ratificada a montante** (RT5, `pontualidade v1.1.0`, *nulos nunca inventados*, ADR-0002) — não precisa de consumidor para ser validado; deixa o §7 honestamente **parcial**, como o ADR-0001 fez | Um "contrato congelado" que não diz qual URL chamar é objeto incomum — exige que a nota de escopo seja lida; não fecha o follow-up 3 do aceite, apenas o estreita |
| **C — Continuar adiando até existir consumidor** | Esforço zero; leitura literal da condição escrita no aceite | I7 segue violada por tempo indeterminado; e é **ficção**: a forma já está decidida, versionada e publicada — só está registrada no lugar errado. As duas convenções de envelope seguem divergindo |

## 3. Decisão

**Opção B — congela-se o núcleo normativo do C3 como `v1.0.0`; a superfície fica adiada.**

O princípio que resolve a tensão do §1.4:

> **Estreita-se o contrato até o que é estável, em vez de enfraquecer a versão para cobrir o que
> não é.** Um `v0.x` perpétuo diria "tudo aqui pode quebrar" sobre garantias que **não podem**
> quebrar, porque decorrem de regras já ratificadas a montante.

### 3.1 O que o C3 `v1.0.0` torna normativo

Seis garantias. Cada uma é **derivada de autoridade já ratificada** — nenhuma depende de
preferência de consumidor:

| # | Garantia | Deriva de |
|---|---|---|
| **G1** | **Auto-descrição.** A resposta declara qual contrato e qual versão carrega. | ADR-0002 |
| **G2** | **Fronteira verbatim × derivado.** Medidas e contadores saem do C2 **sem recálculo, sem arredondamento e sem reescrita de tipo**; todo campo derivado é rotulado, e o conjunto de derivados é **fechado** nesta versão. | RT5 (`engineering-execution-plan.md`) |
| **G3** | **Proveniência obrigatória.** Toda resposta carrega `provenance` (versão da métrica, base, limiar, linhagem) e `validation` (resultado do gate do C2). | AC4 · `contracts.md` §C2 *Garantias* |
| **G4** | **Recusa explícita.** A resposta **declara** a resposta-âncora ou **recusa-se a respondê-la** — nunca a fabrica. `conclusive` é obrigatório. | AC3 · *nulos nunca inventados* |
| **G5** | **Nulos nunca inventados.** `on_time_rate` nulo é servido nulo, jamais `0`; contador ausente numa versão antiga do C2 é servido `null`, jamais `0`. | `pontualidade v1.1.0` · Adendo A.3 do aceite |
| **G6** | **Avisos visíveis.** Insumo sintético, registro em quarentena e publicação de versão legada aparecem em `warnings`. **O silêncio é o que se proíbe.** | GOV-005 |

A especificação campo a campo vive em `contracts.md` §C3 — este ADR decide, o contrato descreve.

### 3.2 O envelope do C3 adota a convenção do C2

`response_contract` / `response_version` passam a **`contract` / `contract_version`**, e o valor
da versão passa a levar o prefixo `v`:

```json
{
  "contract": "C3",
  "contract_version": "v1.0.0",
  "query":      { "…": "…" },
  "validation": { "…": "…" },
  "provenance": { "…": "…" },
  "months":     [ { "…": "…" } ],
  "warnings":   []
}
```

É **breaking change de forma** — e é o momento mais barato em que ela pode ocorrer. O que se
quebra é um rascunho auto-declarado `v0.1.0`, cujos únicos consumidores são o renderizador de
texto da própria API e um caso de teste (`self_test.py:525`). Adiar esta unificação até existir
consumidor externo significaria pagá-la quando ela **tiver** custo.

> Corolário: **uma convenção de auto-descrição para todo o ecossistema.** Quem aprende a ler um
> documento do ecossistema aprendeu a ler todos.

### 3.3 O que o C3 `v1.0.0` **não** fixa — e por quê

Fica fora do contrato, deliberadamente, tudo sobre o que um consumidor real teria opinião legítima
e hoje não tem quem a dê:

| Fora do C3 `v1.0.0` | Onde vive hoje | Por que fica fora |
|---|---|---|
| **Protocolo de transporte** (HTTP, CLI, arquivo) | `api/README.md` | Item do §7 adiado por decisão consciente |
| **Superfície de endpoints** (URLs, *status codes*, cabeçalhos) | `api/README.md` | Idem — e é a parte que um consumidor reprojeta primeiro |
| **Ergonomia de nomes e aninhamento** (inclusive a divisão `answer` × `answers`) | `serve.py` | Nenhuma autoridade a montante a determina; só uso real a valida |
| **Paginação e filtros** além de par de rota + mês | — | Não existe volume que os justifique |
| **Arredondamento e apresentação** | `render_text()` | Resolvido em §3.4 |

Consequência para o `engineering-execution-plan.md` §7: o item *"Protocolo e formato da API"* fica
**parcialmente decidido** — o **formato do documento de resposta** está decidido; **protocolo e
superfície permanecem adiados**. É o mesmo tratamento que o ADR-0001 deu a *"Formato dos conjuntos
de dados intermediários"*, e pela mesma razão: decidir só o que a evidência sustenta.

### 3.4 Arredondamento é do consumidor, não do C3

Resolvida a frase de `api/README.md:263-264` (§1.3): **o C3 transporta o `on_time_rate` em precisão
plena do C2** (G2). Arredondar é **apresentação**, e apresentação é de quem exibe. O renderizador
de texto da CLI (`render_text` / `_fmt_rate`) é **um consumidor do C3**, não parte dele — é ele
quem formata `86.75%`. Um C3 que arredondasse destruiria, na fronteira de saída, a precisão que o
AC4 exige para reconciliar.

### 3.5 Campos informativos não são esquema

A resposta carrega duas notas em prosa: `_value_source` (*"read verbatim from C2"*, nas entradas
por direção) e `on_time_rate_note` (*"PURE AGGREGATION…"*, nas entradas combinadas). São
**informativas**: podem mudar de texto sem emenda de contrato.

A assimetria entre elas foi verificada e **está correta, não é defeito**: uma entrada combinada
**não** é verbatim — sua taxa é reexpressa a partir de somas de inteiros do C2 — então afirmar
`_value_source` ali seria falso. Cada caminho carrega a nota verdadeira para ele.

## 4. Consequências

**Positivas**

- **I7 deixa de ser violada:** o contrato compartilhado passa a ter sede em `contracts.md`; o
  `api/README.md` passa a **derivar** dele, como todo `<repo>/README` (§2 da arquitetura).
- **Uma convenção de envelope** no ecossistema inteiro (§3.2).
- A **semântica de recusa** — a propriedade mais valiosa da camada de exposição, hoje sustentada
  só por teste e prosa de README — vira **garantia de contrato**. Um substituto da API que
  respondesse `most_reliable` num empate passaria a violar o C3, não apenas a divergir do README.
- `api/README.md:263-264` deixa de afirmar o inverificável (§3.4).

**Negativas / custo aceito**

- **Breaking change na forma da resposta.** Mitigado por ser `v0.1.0` rascunho, com dois
  consumidores internos (§3.2). Ainda assim é quebra, e está registrada como tal.
- **Um contrato congelado que não diz qual URL chamar** é objeto incomum e pode ser
  **super-interpretado**: alguém pode ler "C3 `v1.0.0` congelado" e supor que a superfície HTTP
  também está. Mitigação: *Nota de escopo* explícita na seção do contrato, mesmo dispositivo que o
  C2 usa para armazenamento e framework.
- **A condição escrita no aceite não foi satisfeita.** `sprint-01-acceptance.md:88` pede validação
  por consumidor; não há consumidor. Este ADR **não fecha** aquele follow-up — ele o **estreita**
  para *"validar a ergonomia e congelar a superfície"*. Afirmar fechamento seria exatamente o tipo
  de falsificação que o Adendo A existiu para reparar.
- **Risco registrado:** o primeiro consumidor real pode exigir mudança na forma do bloco `answer`.
  Se for aditiva → `v1.1.0`; se exigir reestruturar o bloco → `v2.0.0`. Estreitar o escopo (§3.3)
  reduz a superfície desse risco, **não o elimina**.
- **Quarto evento de versionamento de contrato em ~2 meses**, em documento de Tier 3. Diferente
  dos três anteriores, este **cria** um contrato em vez de emendar um existente — mas o sinal de
  `documentation-architecture.md` §6 continua valendo e segue sob observação.

## 5. Propagação a jusante

| Alvo | Mudança |
|---|---|
| `docs/ecosystem/contracts.md` §C3 | ✅ Seção escrita: envelope, garantias G1–G6, semântica de recusa, *Nota de escopo* |
| `docs/ecosystem/contracts.md` — Histórico de versões | ✅ Linha **C3 `v1.0.0`** (congelamento inicial, núcleo normativo) |
| `docs/ecosystem/contracts.md` §C2 → `on_time_rate` | ✅ O parêntese *"(arredondamento é do C3)"* apontava para um responsável que §3.4 acaba de recusar. Corrigido para *"nem o C2 nem o C3 arredondam"*. **Não é emenda de esquema:** nenhum campo, tipo, medida ou garantia muda — a garantia de precisão plena é a mesma —, e o C2 permanece em `v1.2.0` |
| `docs/ecosystem/decisions/README.md` | ✅ Índice: ADR-0003 → `Aceito` |
| `docs/README.md` | ✅ Índice (I5): C3 deixa de ser 🕓 |
| `docs/engineering/engineering-execution-plan.md` §7 | ✅ *"Protocolo e formato da API"* → **parcialmente decidida** (formato sim; protocolo e superfície não) |
| `docs/engineering/sprints/sprint-01-acceptance.md` | ✅ **Adendo B** — o follow-up 3 é estreitado, não fechado (§4). Adendo, nunca edição do corpo (I4 · §2.1) |
| `market-intelligence-api/src/serve.py` | ✅ `contract`/`contract_version` = `C3`/`v1.0.0` em `build_comparison`, `build_answer_only`, `build_meta` e no cabeçalho `Server` |
| `market-intelligence-api/tests/self_test.py` | ✅ Classe `TestC3ResponseContract` — as seis garantias viram teste executável |
| `market-intelligence-api/README.md` | ✅ Passa a apontar `contracts.md` §C3 como sede; frase do arredondamento corrigida (§3.4) |

## 6. Rastreabilidade

- Issue de governança: **GOV-009**.
- Registro anterior do problema: `sprint-01-acceptance.md:88` (follow-up 3) e o Adendo A.2, que em
  2026-08-31 classificou o item como *"ainda aberto — Fase 1"*.
- Aviso que moldou o escopo: **ADR-0002 §4** — *"o problema é o contrato ter sido congelado antes
  de existir consumidor"*.
- Verificação **antes** (2026-09-20):
  `python src/serve.py --input input/sample_c2.json --route CGH-SDU --month 2023-06 --combine --json`
  → `"response_contract": "C3-draft"`, `"response_version": "0.1.0"`.

### Ratificação — 2026-09-20 (Sprint Lead)

Decidido: **Opção B**, versionada como **C3 `v1.0.0`** (§3.1), envelope alinhado ao C2 (§3.2),
protocolo e superfície de endpoints **permanecem adiados** (§3.3).

| Verificação | Resultado |
|---|---|
| Mesmo comando de antes, após a mudança | `"contract": "C3"` · `"contract_version": "v1.0.0"` (antes: `"response_contract": "C3-draft"`, `"response_version": "0.1.0"`) |
| **Nenhum número se moveu** | Resposta ao **C2 real** (`--route CGH-SDU --combine --json`) comparada com a do `serve.py` anterior, estrutura inteira: **idêntica** fora das duas chaves do envelope. Os 5 campos de topo restantes são os mesmos |
| `GET /meta` publica a versão vigente | `output_contract: {"name": "C3", "version": "v1.0.0"}` |
| As seis garantias viram teste | `TestC3ResponseContract` — **20 casos** cobrindo G1…G6 |
| Suíte da API | 62 → **82 casos unitários** (88 com os 6 de integração), todos verdes |
| Integração contra o C2 real | `python tests/self_test.py --integration` → `INTEGRATION: VERIFIED` |

A última linha é a que justifica o ADR: a garantia de recusa deixa de depender de um README e
passa a ser verificável contra o contrato.
