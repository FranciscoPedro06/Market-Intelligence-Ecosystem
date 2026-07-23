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
│   │   └── sprints/                          # planos de Sprint (execução, camada-ponte)
│   │       └── sprint-NN-<nome>.md
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
| **Derivado (dependente)** | Deve permanecer consistente com uma autoridade a montante. Muda quando a fonte muda. | `roadmap.md`, `metrics-definitions.md`, `engineering-execution-plan.md`, `contracts.md`, `<repo>/README.md` |
| **Referência (transversal)** | Lido por todos, não deriva regra de negócio. Evolução contínua e incremental. | `glossary.md`, `docs/README.md`, `documentation-architecture.md` |
| **Log (append-only)** | Registro histórico. Entradas são **imutáveis** — nunca reescritas, apenas superadas. | `decisions/` (ADRs), `engineering/spikes/` |

**Regra do sistema:** mudança flui de **autoridade → derivado → repo**. Referências são consultadas por todos. Logs só recebem novas entradas.

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
| `metrics-definitions.md` | contracts (C2) → analytics/README, api/README. |
| `engineering-execution-plan.md` | contracts, ADRs, todos os `<repo>/README`. |
| `contracts.md` | os `<repo>/README` dos produtos afetados (versionar o contrato). |
| `<repo>/README.md` | ninguém a jusante (é folha do grafo). |
| **Qualquer doc criado/movido/removido** | `docs/README.md` (índice) — **obrigatório**. |
| **Uma decisão adiável é tomada** | novo ADR em `decisions/` + marcar como decidida na seção 7 do engineering-execution-plan. |
| **Um spike conclui** | nova entrada em `spikes/` + atualizar premissas/riscos no engineering-execution-plan. |

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
| **4 — Repositório** | Por Sprint que toca o repo | `<repo>/README` |
| **Append-only** | Por evento (decisão, spike) | decisions/, spikes/ |
| **Contínuo** | A cada mudança estrutural | glossary, docs/README, documentation-architecture |

> Sinal de alerta do sistema: se um documento de Tier 0 muda com frequência de Tier 4, há uma decisão instável escondida — investigar antes de continuar.

---

## 7. Invariantes do sistema

Regras que, se violadas, indicam que a documentação parou de funcionar como sistema:

- **I1** — Todo documento tem exatamente um papel (seção 2) e um dono.
- **I2** — O grafo de dependências é acíclico (seção 3).
- **I3** — Propagação só ocorre montante → jusante (seção 5).
- **I4** — Logs (ADRs, spikes) nunca são reescritos, apenas acrescidos.
- **I5** — O índice (`docs/README.md`) reflete o estado real dos arquivos.
- **I6** — Nenhum stub vazio: documento existe só com conteúdo real.
- **I7** — Contrato é governança do ecossistema; nenhum repo é dono de contrato compartilhado.
