# Market Intelligence Ecosystem — Documentação Raiz

> **Comece por aqui.** Este é o ponto de entrada oficial do ecossistema.
> Qualquer engenheiro ou IA deve conseguir se orientar lendo apenas a documentação — sem depender do histórico de conversas.
> **Princípio 5:** a documentação é a única fonte oficial de conhecimento.

---

## Como ler esta documentação

A governança é organizada em **três camadas**, do geral para o específico:

1. **Ecossistema** — regras e conhecimento válidos para tudo (`docs/ecosystem/`).
2. **Produto** — a visão do Flight Intelligence Platform (`docs/product/`).
3. **Repositórios** — cada produto de código documenta a si mesmo, no seu próprio diretório (`<repo>/README.md`).

Entre elas existe uma camada-ponte de **Engenharia** (`docs/engineering/`), que traduz produto em execução.

Ordem de leitura recomendada para quem chega:
`ecosystem/vision` → `ecosystem/principles` → `product/product-discovery` → `engineering/engineering-execution-plan` → `ecosystem/contracts` → repositório em que vai atuar.

> Para entender a documentação **como sistema** (papéis, dependências, propagação de mudança), ver `documentation-architecture.md`.

---

## Mapa dos documentos

Legenda de status: ✅ existe · 🕓 a criar quando houver conteúdo real (nunca stub vazio).

### Camada 1 — Ecossistema (`docs/ecosystem/`)
| Documento | Status | Responsável |
|---|---|---|
| `vision.md` — visão do Market Intelligence Ecosystem | 🕓 | PM + EM |
| `principles.md` — princípios de produto e engenharia | 🕓 | EM |
| `glossary.md` — vocabulário canônico (domínio + técnico) | 🕓 | EM (curadoria) |
| `contracts.md` — contratos de dados C0–C3 entre os produtos | ✅ (C0 validado, C1 `v1.0.0`, C2 `v1.2.0`; C3 🕓) | EM (guardião) |
| `decisions/` — ADRs (registros de decisão arquitetural) | ✅ (ADR-0001 e ADR-0002 aceitos) | EM |

### Camada-ponte — Engenharia (`docs/engineering/`)
| Documento | Status | Responsável |
|---|---|---|
| `engineering-execution-plan.md` — ponte visão → execução | ✅ | EM |
| `sprints/sprint-01-walking-skeleton.md` — plano da Sprint 1 | ✅ | EM (plano) / Engenheiros (execução) |
| `sprints/sprint-01-acceptance.md` — registro de aceite da Sprint 1 | ✅ | Sprint Lead |

### Camada 2 — Produto (`docs/product/`)
| Documento | Status | Responsável |
|---|---|---|
| `product-discovery.md` — Discovery aprovada (fonte da visão) | 🕓 | PM |
| `roadmap.md` — sequência de entrega de valor (MVP → v2 → v3) | 🕓 | PM |
| `metrics-definitions.md` — definições canônicas dos KPIs | ✅ (pontualidade `v1.1.0`) | PM (valor) + EM (consistência) |

### Camada 3 — Repositórios (dentro de cada repo, **não** em `docs/`)
| Documento | Status | Responsável |
|---|---|---|
| `market-intelligence-collector/README.md` | ✅ | Eng. do repo |
| `market-intelligence-api/README.md` | ✅ | Eng. do repo |
| `market-intelligence-analytics/README.md` | ✅ | Eng. do repo |

---

## Regras de governança da documentação

- **Uma responsabilidade por documento.** Se dois documentos disputam o mesmo assunto, um deles está errado.
- **Contratos de dados vivem no ecossistema**, não nos repositórios — nenhum repo é dono de um contrato que compartilha.
- **Decisões adiáveis viram ADR quando tomadas.** A seção 7 do Engineering Execution Plan lista as decisões hoje adiadas; cada uma, ao ser decidida, gera um ADR em `ecosystem/decisions/`.
- **Sem stub vazio.** Um documento só é criado quando tem conteúdo real. Até lá, fica marcado 🕓 neste mapa.
- **Este índice é atualizado sempre que um documento é criado, movido ou removido.** É o mapa; se ele mentir, a governança falha.
