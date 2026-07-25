# Definições Canônicas de Métricas — Flight Intelligence Platform

> **Responsabilidade deste documento:** fixar, de forma única e versionada, o significado exato de cada KPI do produto.
> **Fonte de valor:** PM. **Consistência:** EM/Sprint Lead.
> **Princípio:** duas leituras do mesmo dado não podem gerar números diferentes (mitiga RT3). Toda métrica aqui é reconciliável à mão com o dado bruto.

---

## Pontualidade (on-time performance) — `v1.1.0`

**Congelada na Fase 0 da Sprint 1; emendada na Fase 2 (CCR do Analytics).** Definição usada de ponta a ponta pelo Analytics e servida pela API.

### Definição

> Um voo é **pontual** quando sua **chegada real** ocorre em até **15 minutos** após a **chegada prevista**.

Formalmente, para um voo *v*:

```
pontual(v)  ⟺  (chegada_real(v) − chegada_prevista(v)) ≤ 15 minutos
```

- Chegadas **antecipadas** (diferença negativa) contam como **pontuais**.
- O limiar é **inclusivo**: exatamente +15 min ainda é pontual; +16 min não é.

### Taxa de pontualidade (o número comparado entre companhias)

Para uma **companhia × rota × mês**:

```
pontualidade = voos_pontuais / voos_no_denominador
```

- **Denominador** = voos **operados** (`Situação Voo = REALIZADO`) que possuem **chegada real E chegada prevista** registradas. *(v1.1.0: sem chegada prevista, `pontual(v)` é indefinido — o voo é **não mensurável**, ver Exclusões.)*
- **Numerador** = subconjunto do denominador que satisfaz `pontual(v)`.
- Resultado expresso como fração/percentual, com o denominador sempre reportado junto (para reconciliação).

### Regras de recorte (congeladas)

| Regra | Decisão | Justificativa |
|---|---|---|
| **Horário-base** | **Chegada** (destino) | É o que a persona sente ao viajar. |
| **Tolerância** | **≤ 15 minutos** | Padrão de mercado (ANAC/DOT). |
| **Recorte temporal** | **Mensal** (por `mês de referência`) | Mostra evolução ao longo do tempo. |
| **Fuso horário** | **America/Sao_Paulo** (horário de Brasília) | A fonte VRA publica horários locais sem offset. |

### Exclusões (fora do denominador)

- **`CANCELADO`** — cancelamento está **fora do escopo desta Sprint**; não entra no denominador nem é reportado como métrica separada agora. Registrado de forma transparente, nunca como voo pontual/atrasado.
- **`NÃO INFORMADO`** — status desconhecido; **excluído** do denominador e contabilizado à parte como "não reportado". Nunca descartado silenciosamente nem tratado como operado.
- Voos `REALIZADO` **sem chegada real** — excluídos do denominador e registrados como dado ausente transparente (`flights_operated_missing_arrival`).
- Voos `REALIZADO` **sem chegada prevista** — classificados como **não mensuráveis**: `pontual(v)` é indefinido sem o horário de comparação. **Permanecem fora da métrica de pontualidade** (fora do denominador e do numerador) e aparecem **apenas em contadores de transparência** (`flights_operated_missing_schedule`). **Nunca** contados como atrasados nem inventados. *(v1.1.0)*

### Reconciliação

O número servido pela API para (companhia, rota, mês) deve ser reproduzível manualmente: filtrar o dataset bruto por essa companhia/rota/mês, contar voos com `Situação Voo = REALIZADO` que possuam chegada real **e** chegada prevista, aplicar a regra dos 15 min, dividir. O resultado deve bater com o do Analytics (AC4).

---

## Métricas fora do escopo da Sprint 1

Cancelamento, atraso médio e concentração de mercado **não** são definidos nesta versão. Serão adicionados aqui — versionados — quando entrarem em escopo, nunca antes (sem stub vazio).

---

### Histórico de versões

| Versão | Data | Mudança |
|---|---|---|
| `v1.0.0` | 2026-07-24 | Congelamento inicial da pontualidade (Sprint 1, Fase 0): chegada, ≤15 min, mensal, denominador = REALIZADO com chegada real; cancelamento fora de escopo. |
| `v1.1.0` | 2026-07-25 | Emenda (CCR do Analytics, Fase 2): voos `REALIZADO` sem chegada prevista são **não mensuráveis** — excluídos do denominador, apenas em contador de transparência. Corrige a violação de "nulos nunca inventados" do denominador `v1.0.0` (voos indefinidos eram contados como atrasados). |
