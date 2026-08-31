# ADRs — Registros de Decisão Arquitetural

> **Responsabilidade deste documento:** índice dos ADRs e template canônico.
> **Dono:** EM / Sprint Lead (guardião).
> **Papel no sistema:** este diretório é um **Log (append-only)** — ver `documentation-architecture.md` §2.

## O que é um ADR aqui

Um ADR registra **uma** decisão arquitetural que era adiável e deixou de ser. A regra que
o cria está em `documentation-architecture.md` §5:

> *"Uma decisão adiável é tomada → novo ADR em `decisions/` + marcar como decidida na seção 7 do engineering-execution-plan."*

Consequências disso:

- **Um arquivo = uma decisão.** Nome: `NNNN-<titulo-em-kebab-case>.md`, numeração sequencial, nunca reaproveitada.
- **Imutável (I4).** O corpo de um ADR aceito **não é reescrito** — é **superado** por um ADR posterior que o referencia.
- **Toda decisão adiável tomada precisa de ADR.** Se a seção 7 do `engineering-execution-plan.md` marca algo como decidido e não existe ADR, a governança está inconsistente.

### Quando um ADR ainda pode mudar — o selo

Regra ratificada em 2026-08-31 (**GOV-006** item 3), especificada em
`documentation-architecture.md` **§7 → I4.1**:

> Um ADR é **selado na aceitação**. Enquanto `Proposto`, é rascunho dentro do log; a partir de
> `Aceito` (ou `Rejeitado`), nada muda.

A transição `Proposto → Aceito` pode alterar:

- o campo **`Status`**;
- a **Data** (passa a registrar *proposta* e *ratificação*);
- a seção **3. Decisão** — convertida de *recomendação* em *decisão*, com as subseções que
  registram o que o decisor resolveu (ver ADR-0001 §3.1, ADR-0002 §3.1–3.3).

**Contexto** e **Opções consideradas** não são reescritos: o registro do que se sabia ao propor é
o que dá valor ao ADR. Depois do selo, uma reversão gera um **novo** ADR com `Supera: NNNN`.

## Índice

| ADR | Título | Status | Data | Decisão adiável (plano §7) |
|---|---|---|---|---|
| [0001](0001-artefato-canonico-c2.md) | Artefato canônico do C2 | ✅ **Aceito** | 2026-07-26 | *Formato dos conjuntos de dados intermediários* (parcial: identidade) |
| [0002](0002-envelope-documento-c2.md) | Envelope do documento C2 | ✅ **Aceito** | 2026-08-31 | *Formato dos conjuntos de dados intermediários* (fecha a decisão: forma do documento) |

Legenda: 🟡 Proposto (aguarda Sprint Lead) · ✅ Aceito · ❌ Rejeitado · ⛔ Superado por outro ADR.

---

## Template

```markdown
# ADR-NNNN — <título>

- **Status:** Proposto | Aceito | Rejeitado | Superado por ADR-MMMM
- **Data:** YYYY-MM-DD
- **Decisor:** Sprint Lead / EM
- **Proponente:** <papel>
- **Decisão adiável correspondente:** <item da seção 7 do engineering-execution-plan>
- **Supera:** — | ADR-MMMM

## 1. Contexto

O que forçou a decisão agora. Fatos verificáveis, com caminho de arquivo e linha.
Nenhuma opinião nesta seção.

## 2. Opções consideradas

| Opção | Prós | Contras |
|---|---|---|

## 3. Decisão

A opção escolhida, em uma frase afirmativa. Enquanto o Status for `Proposto`, esta
seção registra a **recomendação** e o que falta para ratificar.

## 4. Consequências

O que passa a ser verdade, incluindo o que fica pior. Consequência negativa omitida
é ADR incompleto.

## 5. Propagação a jusante

Quais documentos e repositórios precisam mudar quando este ADR for aceito
(`documentation-architecture.md` §5). Sem isso a decisão não chega ao código.

## 6. Rastreabilidade

Evidência: auditoria, Issue GOV, commit, teste.
```
