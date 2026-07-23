# Sprint 1 — Walking Skeleton

> **Tipo:** fatia vertical única atravessando o ecossistema.
> **Base:** Engineering Execution Plan, seções 8 (critérios do Walking Skeleton) e 9 (critérios de abertura).
> **Dono do plano:** EM (participação encerrada ao final deste documento). **Execução:** Collector / Analytics / API Engineers.
> **Regra-mãe:** esta Sprint entrega **um** insight de ponta a ponta, com dado real, e nada além.

---

## 1. Objetivo da Sprint (único)

> Responder, de ponta a ponta e com dado oficial real, à pergunta-âncora do MVP para **uma única rota**:
> **"Nesta rota, qual companhia é mais confiável?"** — medida pela **pontualidade** de cada companhia, com recorte temporal.

Se ao final a persona conseguir olhar uma rota e comparar a pontualidade de duas companhias com um número reconciliável à fonte, a Sprint cumpriu seu propósito.

---

## 2. Escopo

### Dentro do escopo
- **Uma rota-alvo** de alto volume, para garantir dado suficiente e múltiplas companhias.
  - *Proposta a confirmar na Fase 0:* **CGH (São Paulo/Congonhas) ↔ SDU (Rio/Santos Dumont)** — corredor doméstico mais denso do país, operado por múltiplas companhias. Seleção final depende da validação do dado.
- **Uma métrica obrigatória:** taxa de **pontualidade** por companhia, por mês, na rota-alvo.
- **Comparação entre ≥ 2 companhias** na mesma rota.
- **Recorte temporal mensal** (evolução ao longo do tempo).
- Os **três produtos participam** minimamente: Collector adquire, Analytics calcula, API expõe.

### Fora do escopo (trava explícita)
- ❌ Mais de uma rota.
- ❌ Outras métricas como obrigatórias (cancelamento/atraso médio/concentração ficam para depois — cancelamento é *stretch opcional* só se não tocar o DoD nem ampliar escopo).
- ❌ Preço, demanda, dados internacionais, tempo real, previsão.
- ❌ Qualquer decisão de arquitetura/tecnologia adiável (seção 7 do plano).
- ❌ Cobertura histórica ampla — período mínimo suficiente para mostrar evolução mensal.

---

## 3. Fases da Sprint

A Sprint é **contract-first**: os contratos são congelados cedo para que os três engenheiros trabalhem em paralelo, evitando o gargalo da cadeia linear.

| Fase | O que acontece | Participantes |
|---|---|---|
| **Fase 0 — Gate** | Validar premissas bloqueantes **P1, P2, P4**; congelar a definição de "pontualidade"; escrever contratos **C1** e **C2**; confirmar a rota-alvo. **Se P1/P2/P4 falharem, a Sprint para.** | Os 3 engenheiros + insumo de métrica do PM |
| **Fase 1 — Build paralelo** | Cada engenheiro constrói contra os contratos congelados, usando amostra acordada. | Collector, Analytics, API |
| **Fase 2 — Integração** | Ligar a cadeia real na rota-alvo: Collector → Analytics → API. | Os 3 |
| **Fase 3 — Reconciliação e aceite** | Conferir o número à mão contra a fonte; validar reexecução; revisão cruzada. | Os 3 |

---

## 4. Papéis participantes e responsabilidades

| Papel | Responsabilidade nesta Sprint |
|---|---|
| **Collector Engineer** | Adquirir o dado oficial voo-a-voo da rota-alvo no período; produzir o **dataset bruto conforme contrato C1**; registrar **proveniência** (fonte, período, data de coleta); garantir **idempotência mínima** (reexecutar não duplica). |
| **Analytics Engineer** | Consumir C1; calcular a **pontualidade por companhia × mês** segundo a definição congelada; produzir a **saída conforme contrato C2**; tratar registros inválidos/ausentes de forma **transparente**; produzir a **evidência de reconciliação**. |
| **API Engineer** | Consumir C2; **expor a comparação** de pontualidade entre companhias na rota-alvo; **não conter regra de cálculo** — apenas servir o que Analytics produziu. |
| *(PM — insumo)* | Fornecer a **definição canônica de "pontualidade"** (tolerância) para `product/metrics-definitions.md`. |
| *(EM — gate/handoff)* | Definir esta Sprint e o gate; **participação encerrada** após a entrega deste documento. |

---

## 5. Entregáveis

| ID | Entregável | Responsável |
|---|---|---|
| E1 | Dataset bruto da rota-alvo + registro de proveniência + **contrato C1 escrito**. | Collector |
| E2 | Dataset de pontualidade (companhia × mês) + definição de métrica aplicada + **contrato C2 escrito** + **evidência de reconciliação manual**. | Analytics |
| E3 | Interface/endpoint que devolve a **comparação de pontualidade entre companhias** na rota-alvo. | API |
| E4 | **README mínimo** de cada repo (papel, contrato de entrada/saída, como executar). | Cada engenheiro |
| E5 | Governança: `ecosystem/contracts.md` com **C1/C2 versionados** e `product/metrics-definitions.md` com **pontualidade**; índice `docs/README.md` atualizado. | Todos |

---

## 6. Critérios de Aceite (alinhados à seção 8 do plano)

- **AC1** — Uma rota real é processada de ponta a ponta com **dado oficial real** (não simulado).
- **AC2** — **Os três produtos participam**: Collector adquire, Analytics calcula, API expõe.
- **AC3** — A **pontualidade é calculada por companhia**, comparando **≥ 2 companhias** na mesma rota, com **recorte mensal**.
- **AC4** — O número servido pela **API é idêntico** ao do Analytics e **reconciliável manualmente** com o dado bruto (amostra conferida à mão).
- **AC5** — A cadeia é **reexecutável** e produz o **mesmo resultado**.
- **AC6** — Os contratos **C1 e C2 existem por escrito** e o que roda os respeita.

---

## 7. Definition of Done (barra de qualidade transversal)

Todo entregável só está "pronto" quando **todos** os itens abaixo forem verdadeiros:

- ✅ Todos os Critérios de Aceite (seção 6) atendidos.
- ✅ Definição de "pontualidade" **documentada** em `metrics-definitions.md` e efetivamente usada.
- ✅ Contratos **C1/C2 versionados** em `contracts.md`.
- ✅ Cada repo tem **README mínimo** (papel, contrato in/out, como executar).
- ✅ **Proveniência** do dado registrada (fonte, período, data de coleta).
- ✅ Registros inválidos/ausentes tratados de forma **transparente e documentada** — nunca inventados.
- ✅ **Reconciliação manual registrada** como evidência.
- ✅ **Revisão cruzada** feita entre os três engenheiros.
- ✅ Índice `docs/README.md` atualizado (invariante I5).

---

## 8. Riscos da Sprint

| # | Risco | Mitigação nesta Sprint |
|---|---|---|
| R1 | Fonte não traz o esperado (horário real/situação) — RT1/RT2 | **Fase 0 é gate**: valida antes de qualquer build; falha bloqueia a Sprint. |
| R2 | Definição de "pontualidade" não fechada — RT3 | Congelar em Fase 0; build não começa sem ela. |
| R3 | Acoplamento serial Collector→Analytics→API gera ociosidade | **Contract-first**: C1/C2 congelados cedo → trabalho paralelo contra contratos. |
| R4 | Escopo inchar (mais rotas/métricas) | Escopo travado (seção 2); extras só como *stretch* sem tocar o DoD. |
| R5 | Reexecução corrompe/duplica — RT4 | Coberto por AC5 (reexecução determinística) + idempotência do Collector. |
| R6 | Cálculo vazar para a API — RT5 | Responsabilidade da API restrita a "servir C2" (seção 4). |

---

## 9. Dependências

- **Externa:** disponibilidade e estabilidade da fonte pública voo-a-voo (VRA). Fora do nosso controle — por isso o gate.
- **De produto:** `metrics-definitions.md` (definição de pontualidade) precisa ser fornecido pelo PM antes da Fase 1.
- **Interna (sequência):** contratos **C1/C2 (Fase 0)** desbloqueiam o paralelismo; a **integração (Fase 2)** exige a saída real do Collector para o Analytics, e do Analytics para a API.
- **De governança:** os invariantes do sistema de documentação (I1–I7) permanecem válidos durante a Sprint.

---

## 10. Encerramento da participação do EM

Com este documento, a fundação (Discovery → Plano de Engenharia → Governança documental → Sprint 1) está completa e a Sprint 1 está **pronta para execução**. A partir daqui, a condução passa aos três engenheiros, dentro do escopo, dos contratos e do Definition of Done aqui definidos. **A participação do Engineering Manager nesta Sprint está encerrada.**
