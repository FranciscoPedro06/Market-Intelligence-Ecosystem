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

### A única exceção à imutabilidade

O campo **`Status`** é o único mutável, e apenas na transição `Proposto → Aceito` (ou
`Proposto → Rejeitado`). Depois de `Aceito`, nada mais muda: uma reversão gera um **novo**
ADR com `Supera: NNNN`.

> ⚠️ **Questão aberta de governança (GOV-006):** o I4 declara logs imutáveis sem prever
> campo mutável. A exceção acima é a leitura operacional adotada nestes primeiros ADRs e
> precisa de ratificação explícita do Sprint Lead — ou o I4 admite o campo `Status`, ou
> ADRs só nascem já `Aceito` e a fase de proposta vive fora do log.

## Índice

| ADR | Título | Status | Data | Decisão adiável (plano §7) |
|---|---|---|---|---|
| [0001](0001-artefato-canonico-c2.md) | Artefato canônico do C2 | 🟡 **Proposto** | 2026-07-25 | *Formato dos conjuntos de dados intermediários* |
| [0002](0002-envelope-documento-c2.md) | Envelope do documento C2 | 🟡 **Proposto** | 2026-07-25 | *Formato dos conjuntos de dados intermediários* |

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
