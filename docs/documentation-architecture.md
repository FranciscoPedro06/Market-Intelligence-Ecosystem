# Arquitetura da Documentação — Market Intelligence Ecosystem

> A documentação é tratada como um **sistema**: componentes com papéis definidos, dependências direcionadas, um caminho de leitura único e regras de propagação de mudança.
> Este documento descreve o **sistema da documentação** — não o conteúdo dos documentos.
> Dono: EM. Atualiza quando um documento é adicionado/removido ou quando uma regra de dependência/propagação muda.

---

## 1. Estrutura completa de pastas

```
Market-Intelligence-Ecosystem/
├── docs/                                     # governança do ecossistema (fonte oficial)
│   ├── README.md                             # índice / porta de entrada
│   ├── documentation-architecture.md         # ESTE documento (o sistema da doc)
│   │
│   ├── ecosystem/                            # camada 1 — vale para tudo
│   │   ├── vision.md
│   │   ├── principles.md
│   │   ├── glossary.md
│   │   ├── contracts.md                      # contratos de dados C0–C3
│   │   └── decisions/                        # ADRs (log append-only)
│   │       ├── README.md                     # índice de ADRs + template
│   │       └── NNNN-<titulo>.md              # 1 decisão por arquivo, imutável
│   │
│   ├── engineering/                          # camada-ponte — traduz produto em execução
│   │   ├── engineering-execution-plan.md
│   │   ├── spikes/                           # resultados de spikes (log append-only)
│   │   │   ├── README.md
│   │   │   └── <data>-<premissa>.md
│   │   └── sprints/                          # execução por Sprint (camada-ponte)
│   │       ├── sprint-NN-<nome>.md           # PLANO da Sprint (derivado; editável)
│   │       └── sprint-NN-acceptance.md       # ACEITE da Sprint (log append-only; adendos)
│   │
│   └── product/                              # camada 2 — a visão do produto
│       ├── product-discovery.md
│       ├── roadmap.md
│       └── metrics-definitions.md
│
├── market-intelligence-collector/            # camada 3 — repo se documenta
│   └── README.md
├── market-intelligence-api/
│   └── README.md
└── market-intelligence-analytics/
    └── README.md
```

---

## 2. Papéis dos documentos no sistema

Todo documento tem um **comportamento**, não só um assunto. Quatro papéis:

| Papel | Comportamento no sistema | Documentos |
|---|---|---|
| **Autoridade (fonte)** | Origem da verdade. Muda raramente. Nada acima dele. | `vision.md`, `principles.md`, `product-discovery.md` |
| **Derivado (dependente)** | Deve permanecer consistente com uma autoridade a montante. Muda quando a fonte muda. | `roadmap.md`, `metrics-definitions.md`, `engineering-execution-plan.md`, `contracts.md`, `<repo>/README.md`, `sprints/sprint-NN-<nome>.md` (plano) |
| **Referência (transversal)** | Lido por todos, não deriva regra de negócio. Evolução contínua e incremental. | `glossary.md`, `docs/README.md`, `documentation-architecture.md` |
| **Log (append-only)** | Registro histórico. Entradas são **imutáveis** — nunca reescritas, apenas superadas. | `decisions/` (ADRs), `engineering/spikes/`, `sprints/sprint-NN-acceptance.md` (aceite) |

**Regra do sistema:** mudança flui de **autoridade → derivado → repo**. Referências são consultadas por todos. Logs só recebem novas entradas.

### 2.1 Por que plano e aceite de Sprint têm papéis diferentes

Os dois vivem em `sprints/` e são documentos distintos do sistema:

- O **plano** (`sprint-NN-<nome>.md`) declara o que *será* feito. É **derivado**: se o contrato ou a
  métrica mudam antes do fim da Sprint, o plano se ajusta. Editar é correto.
- O **aceite** (`sprint-NN-acceptance.md`) declara o que *foi verificado e aceito*, numa data, sob as
  versões vigentes **naquele momento**. É **log**. Reescrevê-lo para refletir uma versão posterior
  falsificaria o fato registrado: a Sprint 1 foi aceita sob `pontualidade v1.0.0`, e nenhuma emenda
  posterior torna isso menos verdadeiro. Um registro de aceite continuamente reescrito não prova
  nada — que é exatamente o valor que se espera dele.

> **Consequência operacional (I4):** um aceite obsoleto **não é corrigido no corpo**. Recebe um
> **Adendo** ao final — datado, identificando seção por seção o que foi superado e por qual
> instrumento (emenda de contrato, ADR, nova versão de métrica). O corpo permanece legível como
> o que era verdade na data do aceite; o adendo diz o que é verdade hoje.

*(Decidido em 2026-08-31 — Issue **GOV-006**, item 1.)*

---

## 3. Grafo de dependências (quem precisa estar consistente com quem)

Seta `A → B` lê-se: **B depende de A** (B deve ser consistente com A).

```
              principles ─────────────┐
                                       ▼
vision ──► product-discovery ──► engineering-execution-plan ──► contracts
              │      │                    │                        │
              │      ▼                    ▼                        ▼
              │  metrics-definitions ─────┘                    <repo>/README
              │      │                                             ▲
              ▼      ▼                                             │
           roadmap   └──────────────► contracts (C2) ─────────────┘

  glossary  ── referência transversal: lido por TODOS, deriva de nenhum ──
  decisions/ (ADRs) ── deriva de engineering-execution-plan + principles ──
  spikes/ ── deriva de engineering-execution-plan (valida premissas P1–P7) ──
  sprints/ ── plano deriva de engineering-execution-plan + contracts + metrics ──
             aceite CITA contracts + metrics numa data (log; adendo, não edição) ──
  docs/README.md ── índice: aponta para todos, não é dependido por nenhum ──
```

Leitura das dependências principais:
- **product-discovery** depende de **vision** (o produto realiza o ecossistema).
- **roadmap** e **metrics-definitions** dependem de **product-discovery**.
- **engineering-execution-plan** depende de **product-discovery** + **principles**.
- **contracts** depende de **engineering-execution-plan** (define C0–C3) e de **metrics-definitions** (C2 precisa das definições de KPI).
- **`<repo>/README`** depende de **contracts** (contratos de entrada/saída) + **engineering-execution-plan** (papel do repo na cadeia).
- **glossary** é transversal: fonte de vocabulário para todos; não impõe regra de negócio.
- **ADRs** e **spikes** são logs: derivam do plano, mas não são editados — crescem por acréscimo.
- **Sprints** têm os dois comportamentos: o **plano** depende de `engineering-execution-plan`,
  `contracts` e `metrics-definitions` (e se ajusta quando eles mudam); o **aceite** depende dos
  mesmos documentos no **momento em que é escrito**, e depois congela. A dependência do aceite não
  é "manter-se consistente", é "**declarar sob quais versões foi verificado**" — por isso ele cita
  versões explicitamente e nunca genericamente.

> **Sem dependência circular.** Se um dia dois documentos passarem a depender um do outro, é sinal de que uma responsabilidade está no lugar errado.

---

## 4. Ordem de leitura (onboarding de engenheiro ou IA)

O caminho único para entrar no projeto sem depender de conversas:

1. `docs/README.md` — onde estou e como navegar.
2. `ecosystem/vision.md` — por que o ecossistema existe.
3. `ecosystem/principles.md` — como trabalhamos (regras invioláveis).
4. `ecosystem/glossary.md` — vocabulário (leitura de referência, consultar sempre).
5. `product/product-discovery.md` — o produto e seu MVP.
6. `product/roadmap.md` — a sequência de valor.
7. `product/metrics-definitions.md` — o que cada métrica significa.
8. `engineering/engineering-execution-plan.md` — como a visão vira execução.
9. `ecosystem/contracts.md` — as fronteiras entre os produtos.
10. `ecosystem/decisions/` — por que decidimos o que decidimos (conforme necessário).
11. `<repo>/README.md` — o repositório em que vou atuar.

A ordem de leitura segue o grafo de dependências: **da autoridade para o derivado**. Ninguém lê um derivado antes da sua fonte.

---

## 5. Fluxo de atualização (propagação de mudança)

Quando um documento muda, ele **força a revisão** dos que dependem dele (a jusante). Regras de propagação:

| Se mudar… | Então revisar (a jusante)… |
|---|---|
| `vision.md` | product-discovery → (roadmap, engineering-execution-plan) → cascata completa. |
| `principles.md` | engineering-execution-plan → ADRs vigentes. |
| `product-discovery.md` | metrics-definitions, roadmap, engineering-execution-plan. |
| `metrics-definitions.md` | contracts (C2) → analytics/README, api/README → **registros de sprint que citam a versão alterada** (por adendo). |
| `engineering-execution-plan.md` | contracts, ADRs, todos os `<repo>/README`, planos de Sprint abertos. |
| `contracts.md` | os `<repo>/README` dos produtos afetados (versionar o contrato) → **registros de sprint que citam a versão alterada** (por adendo). |
| `<repo>/README.md` | ninguém a jusante (é folha do grafo). |
| **Qualquer doc criado/movido/removido** | `docs/README.md` (índice) — **obrigatório**. |
| **Uma decisão adiável é tomada** | novo ADR em `decisions/` + marcar como decidida na seção 7 do engineering-execution-plan. |
| **Um spike conclui** | nova entrada em `spikes/` + atualizar premissas/riscos no engineering-execution-plan. |
| **Um contrato ou métrica é versionado** | além da cascata acima: **varrer `sprints/*-acceptance.md` por citações da versão antiga**. Cada registro afetado recebe um **adendo** (nunca edição do corpo — §2.1). Um aceite que cita versão superada sem adendo é violação de I5. |

**Direção da propagação:** sempre montante → jusante. Nunca se altera uma autoridade para "encaixar" um derivado — se o derivado não cabe, ou a fonte muda conscientemente, ou o derivado está errado.

---

## 6. Tiers de estabilidade (frequência esperada de mudança)

O sistema é mais saudável quanto mais estável for o seu núcleo:

| Tier | Frequência | Documentos |
|---|---|---|
| **0 — Bedrock** | Muito raro (mudança estratégica) | vision, principles |
| **1 — Visão de produto** | Por rodada de Discovery aprovada | product-discovery |
| **2 — Planejamento** | Por marco/fase | roadmap, engineering-execution-plan |
| **3 — Fronteiras** | Por revisão de contrato/métrica | contracts, metrics-definitions |
| **4 — Repositório** | Por Sprint que toca o repo | `<repo>/README`, plano de Sprint |
| **Append-only** | Por evento (decisão, spike, aceite) | decisions/, spikes/, `sprint-NN-acceptance.md` |
| **Contínuo** | A cada mudança estrutural | glossary, docs/README, documentation-architecture |

> Sinal de alerta do sistema: se um documento de Tier 0 muda com frequência de Tier 4, há uma decisão instável escondida — investigar antes de continuar.

---

## 7. Invariantes do sistema

Regras que, se violadas, indicam que a documentação parou de funcionar como sistema:

- **I1** — Todo documento tem exatamente um papel (seção 2) e um dono.
- **I2** — O grafo de dependências é acíclico (seção 3).
- **I3** — Propagação só ocorre montante → jusante (seção 5).
- **I4** — Logs (ADRs, spikes, aceites de Sprint) nunca são reescritos, apenas acrescidos.
  Ver **I4.1** para o único momento em que uma entrada ainda pode mudar.
- **I5** — O índice (`docs/README.md`) reflete o estado real dos arquivos.
- **I6** — Nenhum stub vazio: documento existe só com conteúdo real.
- **I7** — Contrato é governança do ecossistema; nenhum repo é dono de contrato compartilhado.
- **I8** — Todo aceite de Sprint declara **explicitamente** as versões de contrato e de métrica sob
  as quais foi verificado. Sem isso, o registro não é auditável e o adendo não tem o que superar.

### I4.1 — Selo no estado final (ratificado 2026-08-31, Issue GOV-006 item 3)

O I4 declarava imutabilidade sem prever que uma entrada pode nascer **aberta**. A regra ratificada:

> **Uma entrada de log é imutável a partir do momento em que atinge seu estado final — o selo.**
> Antes do selo ela é rascunho **dentro** do log; depois dele, nada muda.

| Tipo de entrada | Nasce | Selada em |
|---|---|---|
| ADR | `Proposto` | a aceitação (`Aceito` ou `Rejeitado`) |
| Spike | concluído | a publicação |
| Aceite de Sprint | concluído | a publicação |

**O que a transição `Proposto → Aceito` pode alterar num ADR:** o campo `Status`, a data (que passa
a registrar *proposta* e *ratificação*) e a seção **Decisão** — convertida de *recomendação* em
*decisão*, incluindo as subseções que registram o que o decisor resolveu (ADR-0001 §3.1,
ADR-0002 §3.1–3.3). Isso é deliberado: **um ADR aceito deve ler-se como decisão, não como
sugestão.** Contexto e opções consideradas não são reescritos — o registro do que se sabia ao
propor é justamente o que dá valor ao ADR.

**Depois do selo:** correção de ADR só por **novo ADR** com `Supera: NNNN`; correção de aceite ou
spike só por **adendo** ao final do próprio arquivo. Em nenhum caso se edita o corpo selado.

> Esta subseção fecha a ressalva que vivia em `ecosystem/decisions/README.md` e que descrevia a
> exceção como sendo apenas do campo `Status` — a prática dos dois primeiros ADRs já era mais
> ampla que isso, e a regra agora diz o que de fato se faz.
