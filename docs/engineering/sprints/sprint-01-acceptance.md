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
