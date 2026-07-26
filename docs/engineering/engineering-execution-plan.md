# Engineering Execution Plan — Market Intelligence Ecosystem
### Fundação de engenharia · Deriva da Product Discovery aprovada do Flight Intelligence Platform

> **Escopo deste documento:** transformar a visão de produto em objetivos e critérios de engenharia.
> **Fora do escopo (deliberadamente):** backlog, tarefas, Sprints detalhadas, código, arquitetura concreta e tecnologias específicas.
> **Fonte da verdade de produto:** Product Discovery — Flight Intelligence Platform.
> **Pergunta-âncora do MVP:** *"Nesta rota, qual companhia é mais confiável?"* — respondida com dado público oficial e histórico.

---

## 1. Objetivos de Engenharia (derivados da Discovery)

Cada objetivo existe para servir a pergunta-âncora. Nada aqui é engenharia por engenharia.

| # | Objetivo de engenharia | Deriva de (produto) |
|---|---|---|
| OE1 | Obter, de forma **repetível e rastreável**, o dado público voo-a-voo da aviação doméstica brasileira. | Fonte única do MVP (VRA / ANAC). |
| OE2 | Transformar registros brutos de voo em **indicadores de confiabilidade corretos** por rota e companhia, com histórico. | KPIs congelados: pontualidade, cancelamento, atraso, concentração. |
| OE3 | **Expor** o resultado de forma que a persona consiga comparar companhias na mesma rota. | Decisão recorrente da Ana. |
| OE4 | Garantir **confiabilidade percebida**: todo número deve ser reconciliável com a fonte oficial. | Risco de produto #4 (confiança). |
| OE5 | Manter o ecossistema **simples e adiável** — só adicionar complexidade sob necessidade comprovada. | Princípios de engenharia. |

**Meta única do MVP de engenharia:** provar, de ponta a ponta e com dado real, que o ecossistema consegue responder à pergunta-âncora para *uma* rota.

---

## 2. Capacidades técnicas que o ecossistema precisa possuir

Descritas como **capacidades**, não como implementações ou ferramentas.

### Collector — capacidade de *aquisição confiável*
- Adquirir o dado público a partir da fonte externa de forma repetível.
- Lidar com instabilidade da fonte (indisponibilidade, variação de formato) sem corromper o resultado.
- Produzir um **dado bruto com esquema conhecido e estável**, com registro de proveniência (de onde veio, quando, referente a qual período).
- Ser **idempotente**: reexecutar não deve duplicar nem contaminar o histórico.

### Analytics — capacidade de *transformação correta*
- Interpretar o dado bruto de voo (horários previsto/real, situação do voo).
- Calcular os KPIs do MVP por rota × companhia × período segundo **definições explícitas e versionadas**.
- Tratar dados ausentes ou inválidos de forma **transparente** — nunca inventar valor.
- Produzir um **conjunto analítico consumível** que já responde à pergunta-âncora.

### API — capacidade de *exposição do insight*
- Entregar o resultado analítico ao consumidor recortado por rota.
- Permitir a **comparação entre companhias** na mesma rota.
- Não conter regra de negócio de cálculo — apenas servir o que Analytics produziu.

> Capacidades explicitamente **não requeridas** no MVP: tempo real, previsão, personalização, múltiplas fontes, preço/demanda. (Ver seção 7 e a lista "NÃO faz parte" da Discovery.)

---

## 3. Dependências entre Collector, API e Analytics

O ecossistema é uma **cadeia de valor linear e unidirecional**:

```
[Fonte pública externa (ANAC/VRA)]
            │  (contrato externo — não controlado por nós)
            ▼
      market-intelligence-collector      → produz dado bruto
            │
            ▼
      market-intelligence-analytics      → produz indicadores
            │
            ▼
      market-intelligence-api            → expõe o insight
            │
            ▼
        Consumidor / Persona (Ana)
```

- **Analytics depende de Collector** (do dado bruto e do seu esquema).
- **API depende de Analytics** (do conjunto de indicadores).
- **Collector depende de uma fonte externa** que não controlamos — a dependência mais frágil de todas.
- Não há dependência reversa. Isso é intencional: mantém o Walking Skeleton fino e cada produto substituível.

> **Não** está definido — e não precisa estar agora — se essa cadeia roda como serviços separados, como um único processo, ou de forma agendada. Isso é decisão adiável (seção 7).

---

## 4. Contratos que precisam existir entre os produtos

O que mantém três produtos independentes coerentes são **contratos de dados**, não integrações. Quatro contratos, na ordem da cadeia:

| Contrato | Entre | O que precisa definir | Quem controla |
|---|---|---|---|
| **C0 — Fonte → Collector** | ANAC/VRA → Collector | Campos disponíveis, granularidade (voo-a-voo), cadência de publicação, histórico disponível, semântica dos códigos de situação. | **Externo** (não controlamos) — por isso é o maior risco. |
| **C1 — Collector → Analytics** | Collector → Analytics | Esquema do "registro bruto de voo": companhia, rota (origem→destino), horários previsto/real, situação; garantias de completude, particionamento por período e idempotência. | Nós. |
| **C2 — Analytics → API** | Analytics → API | Esquema do "indicador de confiabilidade" por rota × companhia × período; **definição de cada KPI** (o que é "pontual", tolerância de atraso, como cancelamento entra). | Nós. |
| **C3 — API → Consumidor** | API → Persona | Que perguntas podem ser feitas (por rota) e o que é devolvido (comparação entre companhias). | Nós. |

**Princípio dos contratos:** C1 e C2 devem ser definidos *antes* de qualquer construção significativa, mesmo que mínimos. São eles que permitem os três produtos evoluírem em paralelo sem se quebrarem. O contrato mais importante e mais arriscado — C0 — precisa ser **validado**, não desenhado (ver seções 5 e 6).

---

## 5. Riscos técnicos que podem impedir a entrega do MVP

| # | Risco técnico | Severidade | Efeito no MVP |
|---|---|---|---|
| RT1 | **Fonte externa (C0) diferente do esperado**: sem horário real, sem situação clara, ou indisponível. | 🔴 Crítico | Invalida a fonte única do MVP — sem plano B, não há produto. |
| RT2 | **Qualidade do dado**: registros sem horário real, meses faltantes, correções republicadas pela fonte. | 🔴 Crítico | KPIs incorretos → quebra o objetivo OE4 (confiança). |
| RT3 | **Ambiguidade na definição das métricas**: o que é "pontual"? qual tolerância? fuso dos horários? | 🟠 Alto | Duas leituras do mesmo dado geram números diferentes — erro silencioso. |
| RT4 | **Não-idempotência do Collector**: reprocessamento duplica ou corrompe histórico. | 🟠 Alto | Contamina a série temporal, que é parte do valor. |
| RT5 | **Vazamento de regra de negócio para a API**: cálculo em dois lugares. | 🟡 Médio | Inconsistência entre o que Analytics calcula e o que a API mostra. |
| RT6 | **Overengineering prematuro** (persistência/serviços/orquestração antes da necessidade). | 🟡 Médio | Atrasa o Walking Skeleton; contraria os princípios. |

RT1, RT2 e RT3 são os que efetivamente **impedem** o MVP. Os demais degradam qualidade, mas são gerenciáveis.

---

## 6. Premissas a validar ANTES da Sprint 1

Estas premissas devem ser confirmadas por um **spike de viabilidade** (investigação, não implementação). Enquanto não confirmadas, qualquer plano de execução é especulativo.

- **P1** — A fonte pública (VRA/ANAC) **existe e é obtenível** de forma repetível.
- **P2** — Cada registro traz **horário previsto E real** de partida/chegada e a **situação do voo**.
- **P3** — É possível identificar de forma confiável a **companhia** e a **rota (origem→destino)** em cada registro.
- **P4** — A **semântica dos códigos de situação** (realizado, cancelado, etc.) é conhecida e inequívoca.
- **P5** — Existe **histórico suficiente** para sustentar a análise de evolução temporal.
- **P6** — Há uma **definição acordada de "pontualidade"** (regra de tolerância de atraso) e o **fuso horário** dos horários é conhecido.
- **P7** — A cadência de publicação e o comportamento de **correções/republicações** da fonte são compreendidos (afeta RT4).

> **Regra de gate:** P1, P2 e P4 são bloqueantes absolutos. Sem elas, a Sprint 1 não abre.

---

## 7. Decisões arquiteturais que PODEM (e devem) ser adiadas

Adiar aqui é disciplina, não preguiça. Nenhuma destas aproxima o MVP hoje.

- ⏳ **Modelo de deployment** dos três produtos (serviços separados, processo único, agendado). *Explicitamente não decidir agora.*
- ⏳ **Tecnologia de persistência / armazenamento** dos dados brutos e analíticos.
- 🟡 **Formato dos conjuntos de dados** intermediários — **parcialmente decidida**. A
  **identidade do artefato C2** (`c2_punctuality.json`) foi decidida pelo **ADR-0001** (aceito
  2026-07-26) e vive em `contracts.md` §C2 → *Artefato de referência*. A **forma interna do
  documento** (envelope) segue adiável — ver ADR-0002, ainda `Proposto`. Serialização,
  armazenamento e framework permanecem deferidos.
- ⏳ **Orquestração / agendamento** da cadeia.
- ⏳ **Protocolo e formato da API** além de "responde à pergunta por rota".
- ⏳ **Estratégia de carga histórica** (full vs. incremental) e escala de volume.
- ⏳ **Qualquer camada de BI/visualização** (fora do MVP pela Discovery).
- ⏳ **Múltiplas fontes** (preço, demanda) — são v2.

Cada uma será decidida quando — e somente quando — houver necessidade comprovada, registrada como decisão consciente.

---

## 8. Critérios de conclusão do Walking Skeleton

O Walking Skeleton é a fatia vertical **mais fina possível que atravessa o ecossistema inteiro com dado real**. Considera-se concluído quando **todos** os critérios abaixo forem verdadeiros:

- ✅ **Uma rota real** é processada de ponta a ponta usando **dado oficial real** (não simulado).
- ✅ **Os três produtos participam** minimamente: Collector adquire, Analytics calcula, API expõe.
- ✅ É produzida **ao menos uma métrica de confiabilidade correta** comparando **≥ 2 companhias** na mesma rota.
- ✅ O número é **reconciliável manualmente** com a fonte (satisfaz OE4).
- ✅ A cadeia é **reexecutável** e produz o mesmo resultado (satisfaz idempotência mínima).
- ✅ Os contratos **C1 e C2 existem** de forma escrita, mesmo que mínimos.

> Se qualquer critério falhar, o esqueleto não está pronto — e nenhuma funcionalidade adicional deve ser construída sobre ele.

---

## 9. Critérios para abertura da Sprint 1

A Sprint 1 só abre quando o terreno estiver validado. Critérios de entrada:

- ✅ **Premissas bloqueantes validadas**: P1, P2, P4 confirmadas pelo spike (seção 6).
- ✅ **Definição de "pontualidade" acordada** e documentada (resolve RT3 / P6).
- ✅ **Contratos C1 e C2 rascunhados** — esquema mínimo do dado bruto e do indicador.
- ✅ **Rota-alvo do Walking Skeleton escolhida** (uma rota de alto volume, para ter dado suficiente).
- ✅ **Base de documentação criada** como fonte oficial de conhecimento (Princípio 5).
- ✅ **Objetivo da Sprint 1 = entregar o Walking Skeleton** conforme critérios da seção 8 — nada além.

> Enquanto P1/P2/P4 não estiverem confirmadas, a prioridade **não é** abrir Sprint — é executar o spike de viabilidade da fonte.

---

## Resumo executivo do plano

O ecossistema precisa de exatamente três capacidades — **adquirir, transformar e expor** — encadeadas de forma linear e unidas por quatro contratos de dados, dos quais o mais crítico (a fonte externa) precisa ser **validado antes de qualquer construção**. O primeiro entregável de engenharia não é uma funcionalidade: é o **Walking Skeleton** de uma única rota, com dado real, atravessando os três produtos e reconciliável com a fonte. Toda decisão de arquitetura e tecnologia que não sirva a esse esqueleto está conscientemente adiada.

**Próximo passo natural:** executar o spike de viabilidade das premissas P1–P7. Confirmadas as bloqueantes, os critérios da seção 9 são atendidos e a Sprint 1 pode abrir com um objetivo único: concluir o Walking Skeleton.
