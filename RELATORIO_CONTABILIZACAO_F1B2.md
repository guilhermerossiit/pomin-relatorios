# RELATÓRIO — Contabilização Efetiva (#22) · Lote F1-B2

**Data:** 2026-08-09 · **Branch:** `feat/contabilizacao-efetiva` · **Commits:** `f976f5d`, `9beba9c`
**Escopo:** telas — aba Contabilidade do modal com efetivados/estornos e navegação bidirecional com o Diário.

---

## 1. Aba Contabilidade do modal de Escrituração

**Antes:** prévia (não efetivada) + uma tabela alimentada por uma fixture antiga (`LancamentoContabilNota`), que nunca refletia o que a escrituração realmente gerava.

**Agora:** a aba mostra os **lançamentos contábeis reais** do documento.

- `montarDetalheConecta` expõe `lancamentosContabeis` — os `LancamentoContabil` do store compartilhado cujo `origemTipo`/`origemId` apontam para o documento, **incluindo os estornos**.
- Novo tipo `LancamentoContabilDocumento` (projeção enxuta para a aba).
- Cada lançamento vira um cartão com: histórico, **badge de status** (efetivado / estornado / rascunho), competência, autor, total e a tabela de partidas (conta, centro de custo, D/C, valor).
- **"Ver na Contabilidade"** → `/contabilidade/lancamentos?lancamento=<id>`.
- O estado vazio ("escriture o documento para gerar a contabilização") só aparece quando não há **nem** prévia **nem** efetivado.

## 2. Diário de Lançamentos — coluna Origem

Responde "de onde veio este lançamento", e fecha o outro lado da navegação:

| `origemTipo` | Coluna Origem |
|---|---|
| `DocumentoFiscal` | **link "Fiscal"** → `/gestor-fiscal/monitores/conecta?documento=<id>` |
| `LancamentoContabil` | "Estorno" |
| — | "Manual" |

---

## 3. Evidências

```
POST /v1/fiscal/conecta/integdoc-4/escriturar   → 200 · lancamentoContabilId "lanc-…lvoc4y" · pendencias []
GET  /v1/fiscal/conecta/integdoc-4/detalhe      → lancamentosContabeis: 1
                                                   status "lancado" · 8 partidas · R$ 5.300,00

POST /v1/fiscal/conecta/integdoc-4/reverter-regra
GET  /v1/fiscal/conecta/integdoc-4/detalhe      → o MESMO lançamento agora "estornado",
                                                   com estornadoPorLancamentoId preenchido
```

**Tela do Diário** (`/contabilidade/lancamentos`): colunas `Data · Histórico · Competência · Valor · Tipo · Origem · Status`, com o link para o documento renderizado no lançamento gerado pela escrituração.

`tsc --noEmit` limpo · `next lint` sem erros/warnings.

---

## 4. O que falta no F1-B2 (declarado, não silenciado)

A spec previa mais dois itens que **não entraram** neste lote:

- **Filtro/chip de origem** no Diário (a coluna existe; o filtro por Fiscal × Manual, com chip no painel de filtros aplicados, ainda não).
- **Fila de pendências contábeis** — hoje as pendências são retornadas pelo `escriturar` (`pendenciasContabeis`) e registradas no log de escrituração, mas não têm superfície própria no módulo Contabilidade.

Ambos ficam para o fechamento do F1-B2 ou entram no F1-B3, a seu critério.

## 5. Próximo
**F1-B3** — documentação in-app das telas afetadas + E2E da jornada completa (capturar → check-in → escriturar → lançamento efetivado → visível na Contabilidade → estornar → reescriturar).

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
