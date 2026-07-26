# ADR-0001 — Artefato canônico do C2

- **Status:** ✅ **Aceito** — ratificado pelo Sprint Lead em 2026-07-26
- **Data:** 2026-07-25 (proposta) · 2026-07-26 (ratificação)
- **Decisor:** Sprint Lead (guardião dos contratos)
- **Proponente:** auditoria de integração do ecossistema (2026-07-25)
- **Decisão adiável correspondente:** `engineering-execution-plan.md` §7 — *"⏳ Formato dos conjuntos de dados intermediários"*
- **Supera:** —

## 1. Contexto

O `contracts.md` §C2 descreve **campos, tipos e garantias**, e sua *Nota de escopo* difere
explicitamente "armazenamento, formato de serialização e framework". Consequência: **nenhum
documento de governança nomeia o arquivo do C2**. Cada repo escolheu por conta própria, e as
escolhas divergiram.

| Fonte | Caminho declarado |
|---|---|
| `analytics/src/analyze.py:275` (default de `--output`) | `output/c2_punctuality.json` |
| `analytics/README.md` (seção *Como executar*) | `output/c2_punctuality.json` |
| `analytics/RECONCILIATION.md:6` | `output/c2_punctuality.json` |
| `sprint-01-acceptance.md` §5 (cadeia da Fase 2) | `output/c2_punctuality.json` |
| **`api/README.md`** (2 comandos + exemplo de saída + item 7 do self-test) | **`output/c2_on_time.csv`** |
| **`api/tests/self_test.py:71`** (`ANALYTICS_C2`) | **`output/c2_on_time.csv`** |

Fatos verificados na auditoria:

- `output/` é **gitignored** nos dois repos; **nenhum** dos dois arquivos é rastreado pelo git.
- Os dois arquivos existem hoje na máquina de desenvolvimento e são **idênticos exceto
  `computed_at_utc`** (29.861 bytes cada; `c2_on_time.csv` = `…T16:16:35Z`,
  `c2_punctuality.json` = `…T00:00:00Z`). São duas execuções do mesmo produtor, não dois artefatos distintos.
- `c2_on_time.csv` contém **JSON** apesar da extensão `.csv`. A própria API registra a
  observação no fim do seu README.
- Um `python src/analyze.py` limpo **nunca** produz `c2_on_time.csv` — o nome só existe porque
  alguém passou `--output` manualmente uma vez.

**Impacto funcional (não apenas cosmético).** Num clone limpo do ecossistema:

1. O comando documentado no README da API falha com `ERROR: C2 input not found`.
2. `TestAgainstRealAnalyticsOutput` (5 testes — a **única** verificação contra C2 v1.1.0 real)
   é pulada pelo `@unittest.skipUnless`, e a suíte reporta **"Ran 54 tests — OK"** sem ter
   validado nada real. A verificação de integração é um falso positivo fora da máquina atual.

## 2. Opções consideradas

| Opção | Prós | Contras |
|---|---|---|
| **A — `c2_punctuality.json`** | Já é o default do produtor; já é o nome em 4 das 5 fontes; extensão condiz com o conteúdo; nome descreve a métrica, não a coluna | Exige alterar 2 pontos no README da API + `self_test.py:71` |
| **B — `c2_on_time.csv`** | Zero mudança na API | Extensão mente sobre o conteúdo; contradiz produtor, aceite e evidência de reconciliação; exige mudar o default do Analytics e 3 documentos |
| **C — `c2_on_time.json`** | Corrige a extensão e aproxima do vocabulário `on_time_rate` | Pior dos dois mundos: muda produtor **e** consumidor; nenhum documento usa esse nome hoje |
| **D — aceitar ambos (alias)** | Nenhuma mudança imediata | Dois nomes para um artefato é exatamente a ambiguidade a eliminar; duplica a superfície de erro |

## 3. Decisão

**Opção A — `c2_punctuality.json` é o artefato canônico do C2.**

O nome é o que o produtor já emite por default e o que a governança (aceite §5) e a evidência
de reconciliação já registram. A correção é local ao consumidor, que é o lado divergente.

`c2_on_time.csv` fica **descontinuado**: não é regenerado e pode ser apagado do diretório de
trabalho sem perda (é reproduzível por `analyze.py`, e seu conteúdo difere do canônico apenas
no campo de auditoria não determinístico).

### 3.1 Onde o nome vive — decisão de sede (ratificada com a Opção A)

O nome canônico é registrado no **`contracts.md` §C2**, em seção própria — ***Artefato de
referência*** — **não** como subseção operacional.

Fundamento ratificado pelo Sprint Lead: **o nome do artefato não é informação meramente
operacional — ele é a identidade externa da interface entre dois produtos.** Quem abrir apenas
o contrato deve conseguir responder *qual contrato*, *qual versão* e *qual artefato o
representa*, sem navegar para o plano de engenharia. Deslocar essa informação para o plano
espalharia conhecimento de integração por quatro documentos e reduziria a autoridade do
contrato.

A separação preservada é **identidade × implementação**: o contrato passa a identificar o
artefato, e continua diferindo explicitamente serialização, armazenamento e framework.

Corolário: uma futura mudança do nome (`c2_punctuality.json` → outro) é **mudança de
interface**, não edição cosmética — exige revisão documental do contrato mesmo com esquema
idêntico, ainda que não implique necessariamente bump semântico maior. O registro desta
decisão é uma **revisão documental do C2 `v1.1.0`**, sem alteração de versão.

## 4. Consequências

**Positivas**
- O comando documentado da API passa a funcionar num clone limpo.
- A cadeia `collector → analytics → api` tem um único nome de artefato ponta a ponta.
- Extensão volta a descrever o conteúdo (JSON em `.json`).
- O `contracts.md` passa a ser autossuficiente quanto à identidade do C2: contrato, versão e
  artefato numa leitura só.

**Negativas / custo aceito**
- O `contracts.md` passa a nomear um **arquivo**, o que tensiona a *Nota de escopo*. A tensão é
  resolvida por **distinção de natureza**, não por marcação de não-normatividade: identidade do
  artefato é normativa; serialização, armazenamento e framework continuam deferidos. A *Nota de
  escopo* foi ajustada para dizer isso explicitamente.
- O contrato ganha uma superfície a mais que exige manutenção: renomear o artefato deixa de ser
  mudança livre no repo produtor e passa a exigir revisão do contrato. Custo aceito
  conscientemente — é o preço de tratar o nome como interface.
- Esta decisão **não** resolve o falso positivo do teste: renomear faz o arquivo existir *nesta*
  máquina, não num clone limpo. O `skipUnless` continua transformando ausência em silêncio.
  Isso é **GOV-005** e é independente deste ADR.

## 5. Propagação a jusante

Conforme `documentation-architecture.md` §5 (`contracts.md` → `<repo>/README` dos produtos afetados):

| Alvo | Mudança |
|---|---|
| `docs/ecosystem/contracts.md` §C2 | Nova seção ***Artefato de referência***; *Nota de escopo* ajustada para distinguir identidade (normativa) de serialização (deferida); histórico de versões registra revisão documental sem bump |
| `docs/engineering/engineering-execution-plan.md` §7 | Marcar como **parcialmente decidida** a linha *"Formato dos conjuntos de dados intermediários"*, referenciando ADR-0001. **Apenas a identidade do artefato foi decidida** — a forma interna do documento segue adiável até ADR-0002 (GOV-003) |
| `market-intelligence-api/README.md` | 2 comandos + exemplo de saída + item 7 do self-test → `c2_punctuality.json`; remover a observação sobre extensão `.csv` |
| `market-intelligence-api/tests/self_test.py:71` | `ANALYTICS_C2` → `c2_punctuality.json` |
| `market-intelligence-analytics/README.md` | Nenhuma (já usa o nome canônico) — confirmar |

> Nenhuma alteração de lógica de cálculo em nenhum repo. O C2 emitido não muda um único campo.

## 6. Rastreabilidade

- Auditoria de integração 2026-07-25 — achado ❌2 (divergência de nome de artefato) e S2.
- Issue de governança: **GOV-002**.
- Verificação: reexecução do `analyze.py` sobre o C1 real produziu saída **byte-idêntica** ao
  artefato commitado (`c2_punctuality.json`), confirmando que os dois arquivos não carregam
  informação distinta.
- Ordem: este ADR **precede** ADR-0002 (o nome do artefato antecede a sua forma interna).
- **Ratificação (2026-07-26, ARB):** Sprint Lead ratificou a Opção A sem ressalvas e decidiu a
  sede do registro contra a proposta original — seção de **identidade** no contrato, e não
  subseção operacional (§3.1). Decidiu também **não** estender essa sede automaticamente ao
  ADR-0002: nome de artefato (identidade externa) e envelope do documento (estrutura interna)
  pertencem a níveis arquiteturais distintos, e amarrá-los restringiria o ADR-0002 sem
  necessidade. **GOV-003 permanece uma decisão independente.**
