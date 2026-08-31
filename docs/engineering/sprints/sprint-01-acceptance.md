# Sprint 1 — Registro de Aceite (Walking Skeleton)

> **Papel deste documento:** evidência de que a Sprint 1 atingiu todos os Critérios de Aceite (§6) e o Definition of Done (§7) do plano `sprint-01-walking-skeleton.md`.
> **Conduzido por:** Sprint Lead (execução). EM encerrado após o plano.
> **Data do aceite:** 2026-07-24.

---

## 1. Resultado — a pergunta-âncora respondida com dado real

**"Na rota CGH↔SDU, qual companhia é mais confiável?"** — por pontualidade (`pontualidade v1.0.0`: chegada, ≤15 min), dado oficial ANAC/VRA, 3 meses.

Comparação combinada (soma de contagens das duas direções), melhor → pior:

| Mês | 1º | 2º | 3º |
|---|---|---|---|
| 2023-04 | **TAM 82,14%** | AZU 80,23% | GLO 77,37% |
| 2023-05 | **GLO 85,39%** | TAM 84,96% | AZU 83,91% |
| 2023-06 | **TAM 87,58%** | GLO 85,35% | AZU 84,18% |

(ACN/PTB aparecem com 0% marginal — ver nota de transparência §4.)

---

## 2. Critérios de Aceite

| AC | Critério | Evidência | Status |
|---|---|---|---|
| **AC1** | Uma rota real, ponta a ponta, dado oficial real | VRA/ANAC `VRA_2023_{04,05,06}.csv` baixados do SIROS; 9.527 linhas C1 CGH↔SDU | ✅ |
| **AC2** | Três produtos participam | Collector (C1) → Analytics (C2) → API (comparação), cadeia real executada | ✅ |
| **AC3** | Pontualidade por companhia, ≥2 companhias, mensal | TAM/GLO/AZU comparadas em 3 meses | ✅ |
| **AC4** | API idêntica ao Analytics e reconciliável ao bruto | Triangulação exata (ver §3) | ✅ |
| **AC5** | Reexecutável, mesmo resultado | C2 re-gerado **byte-idêntico** (exceto `computed_at_utc`); Collector idempotente (chave natural; re-run idêntico exceto `ingested_at_utc`) | ✅ |
| **AC6** | C1 e C2 escritos e respeitados | `contracts.md` C1/C2 `v1.0.0`; a cadeia os respeita | ✅ |

---

## 3. Reconciliação manual (AC4) — triangulação RAW = Analytics = API

Recomputação **independente** direto do VRA bruto (script de auditoria do Sprint Lead, sem usar o código do Analytics), grupo **TAM · SBSP→SBRJ · 2023-06**:

| Medida | RAW (auditoria independente) | C2 (Analytics) | API (servido) |
|---|---|---|---|
| `flights_operated` | 653 | 653 | 653 |
| `flights_on_time` | 579 | 579 | 579 |
| `on_time_rate` | `0.886676875957121` | `0.886676875957121` | `0.886676875957121` |
| `flights_cancelled` | 22 | 22 | — |

Valor idêntico ao último dígito nas três camadas. Cross-check adicional: total 2023-06 = 3.173 registros, **igual ao spike da Fase 0**.

---

## 4. Definition of Done

| Item DoD | Evidência | Status |
|---|---|---|
| Todos os AC atendidos | §2 | ✅ |
| Pontualidade documentada e usada | `metrics-definitions.md` `v1.0.0`, aplicada no Analytics | ✅ |
| C1/C2 versionados | `contracts.md` `v1.0.0` | ✅ |
| README mínimo por repo | Collector / Analytics / API têm README (papel, in/out, como executar) | ✅ |
| Proveniência registrada | `PROVENANCE.md` (sha256 por mês); `source_lineage` em cada registro C2 | ✅ |
| Inválidos/ausentes transparentes | contadores `flights_cancelled`/`not_reported`/`missing_arrival`; nunca inventados | ✅ |
| Reconciliação registrada | §3 (este documento) | ✅ |
| Revisão cruzada | Sprint Lead auditou cada entrega contra o contrato adjacente + triangulação RAW=C2=API | ✅ |
| Índice `docs/README.md` atualizado (I5) | atualizado nesta entrega | ✅ |

### Nota de transparência — `0%` de ACN/PTB
Os voos da Azul Conecta (ACN) e PTB nesta rota são `REALIZADO` **sem Chegada Prevista** na fonte VRA. Pela definição congelada entram no denominador (têm chegada real) mas não há como provar pontualidade → 0 pontuais. **O número é honesto, nunca inventado.** Não afeta a comparação relevante (TAM/GLO/AZU têm 0 casos). `NÃO INFORMADO` não ocorre nesta rota/amostra (0 linhas).

---

## 5. Repositórios entregues

| Repo | Papel | Executar |
|---|---|---|
| `market-intelligence-collector` | Adquire VRA → C1 | `python src/collect.py` |
| `market-intelligence-analytics` | C1 → C2 (pontualidade) | `python src/analyze.py --input <c1.csv> --output <c2.json>` |
| `market-intelligence-api` | Serve comparação (C2) | `python src/serve.py --input <c2.json> --route CGH-SDU --month 2023-06 [--combine]` |

Cadeia (Fase 2): `collector/output/c1_flights.csv` → `analytics` → `analytics/output/c2_punctuality.json` → `api`.

---

## 6. Follow-ups (fora do escopo desta Sprint — não bloqueiam o aceite)

- **`pontualidade v1.1.0` (candidato):** tratar `REALIZADO` **sem Chegada Prevista** como bucket de transparência próprio, em vez de no denominador (hoje derruba a taxa de operadores marginais). Requer nova versão da métrica.
- **Envelope de C2 (cosmético):** Analytics emite array JSON puro → a API mostra `c2_contract_version: null`. Considerar um envelope `{contract_version, records[]}` numa iteração futura.
- **C3 formal:** a API entrega um rascunho `C3-draft v0.1.0` (per-direção + `--combine`). Congelar C3 quando o consumidor validar a forma da resposta.

---

**Veredito do Sprint Lead:** Walking Skeleton **concluído e aceito**. Todos os AC e o DoD satisfeitos com dado real, número reconciliável e cadeia determinística.

---

# Adendo A — reconciliação com `pontualidade v1.1.0` e `C2 v1.2.0`

- **Data:** 2026-08-31 · **Autor:** Sprint Lead
- **Forma:** adendo, conforme `documentation-architecture.md` §2.1 e §7 → I4.1 — este registro é
  **log append-only**; o corpo (§§1–6 acima) **não foi alterado** e continua legível como o que
  era verdade em 2026-07-24.
- **Instrumentos:** Issue **GOV-006** (decidiu a forma) · Issue **GOV-001** (levantou o conteúdo).

## A.1 Por que este adendo existe

O aceite foi escrito sob `pontualidade v1.0.0` e `C2 v1.0.0`. Depois dele:

| Data | Mudança | Instrumento |
|---|---|---|
| 2026-07-25 | `pontualidade v1.1.0` / `C2 v1.1.0` — voo `REALIZADO` sem chegada prevista sai do denominador | CCR do Analytics |
| 2026-07-26 | `c2_punctuality.json` fixado como artefato canônico | ADR-0001 |
| 2026-08-31 | Envelope do documento — `C2 v1.2.0` | ADR-0002 |

Nada disso alcançou este arquivo, porque o sistema de documentação não tinha regra que o
alcançasse — a lacuna corrigida por **GOV-006**. O adendo repara o efeito; a regra nova impede a
repetição.

## A.2 O que foi superado

| Seção | O corpo afirma | Verdade sob `v1.1.0` | Instrumento |
|---|---|---|---|
| §1 cabeçalho | `pontualidade v1.0.0` | `pontualidade v1.1.0` | CCR 2026-07-25 |
| §1 · 2023-04 | AZU **80,23%** | AZU **81,05%** | recontagem `v1.1.0` |
| §1 · 2023-05 | TAM **84,96%** | TAM **85,03%** | recontagem `v1.1.0` |
| §1 · 2023-06 | TAM **87,58%** · AZU **84,18%** | TAM **87,71%** · AZU **85,20%** | recontagem `v1.1.0` |
| §1 nota | *"ACN/PTB aparecem com 0% marginal"* | aparecem como **`n/a`** (denominador 0) | `v1.1.0` |
| §2 · AC6 | `contracts.md` C1/C2 `v1.0.0` | C1 `v1.0.0` · C2 **`v1.2.0`** | ADR-0002 |
| §4 DoD | `metrics-definitions.md v1.0.0`; `contracts.md v1.0.0` | ambos `v1.1.0`+ | CCR / ADRs |
| §4 DoD | **3** contadores de transparência | **5** contadores | `C2 v1.1.0` |
| §4 nota ACN/PTB | contar esses voos é honesto | **revogada** — ver A.3 | CCR 2026-07-25 |
| §6 follow-up 1 | `pontualidade v1.1.0` como *"candidato"* | **aplicado** em 2026-07-25 | CCR |
| §6 follow-up 2 | envelope de C2, *"cosmético"* | **resolvido**; a classificação estava errada — ver A.3 | ADR-0002 |
| §6 follow-up 3 | C3 formal | **ainda aberto** — Fase 1 | — |

Percentuais obtidos por `serve.py --combine` sobre o C2 real, por mês, reconferidos em 2026-08-31.

## A.3 As duas afirmações revogadas — e por quê

**1. A nota de transparência do §4 se contradiz com `v1.1.0`.** O corpo diz que voos `REALIZADO`
sem chegada prevista *"entram no denominador (têm chegada real) mas não há como provar
pontualidade → 0 pontuais"*, e chama isso de **honesto**. A `v1.1.0` afirma o oposto, e é a
posição vigente: **mantê-los no denominador os tornava inalcançáveis pelo numerador, ou seja,
contados como atrasados** — afirmar atraso onde a fonte não diz nada é inventar um fato, e viola
a garantia *"nulos nunca inventados"* do próprio C2. A intenção do corpo era certa (não descartar
o dado); o mecanismo escolhido é que produzia a afirmação falsa. Hoje esses voos aparecem em
`flights_operated_missing_schedule` e a taxa é `null`.

**2. O follow-up "Envelope de C2 (cosmético)" estava mal classificado.** Não era cosmético: sem
envelope, o gate `D2_CONTRACT_VERSION` — a única defesa contra consumir um C2 de versão não
suportada — nunca podia disparar. O ADR-0002 discordou explicitamente da classificação e a
corrigiu.

Registrar isto importa mais que corrigir os números: **a governança sustentou por 37 dias duas
proposições contrárias sobre o mesmo fato**, cada uma num documento diferente do repositório que
é fonte única da verdade.

## A.4 O que **não** mudou

- **A triangulação do §3 permanece exata.** O grupo auditado (TAM · SBSP→SBRJ · 2023-06) tem
  **zero** voos sem chegada prevista, então a emenda não o toca: `flights_operated` 653,
  `flights_on_time` 579, `on_time_rate` `0.886676875957121` — idênticos ao último dígito nas três
  camadas, reconferidos contra o C2 `v1.2.0`. **AC4 continua provado pela mesma evidência.**
- **A resposta à pergunta-âncora é a mesma.** A ordenação por mês não se altera: 2023-04 TAM ▸ AZU ▸
  GLO · 2023-05 GLO ▸ TAM ▸ AZU · 2023-06 TAM ▸ GLO ▸ AZU. As taxas se movem, o veredito não.
- **AC1–AC5 e o DoD seguem satisfeitos.** Nenhum critério de aceite depende das versões corrigidas
  aqui; o que mudou foi a precisão dos números e a honestidade da nota do §4.

## A.5 Efeito sobre o aceite

**Nenhum. A Sprint 1 permanece aceita.** Este adendo corrige o que o registro *afirma hoje*, não o
que foi *verificado então* — e nada do que foi verificado deixou de valer. Um aceite que precisasse
ser reaberto a cada emenda de contrato não seria um aceite.

*(Fecha **GOV-001**.)*
