# RELATÓRIO — Portal do Fornecedor · Lote B2

**Data:** 2026-07-31
**Branch:** `feat/portal-fornecedor`
**Commit:** `b1a9b6e`
**Escopo do lote:** reuso das telas do Acompanhamento dentro do Portal do Fornecedor, sob o escopo por participante — modo participante (só o seu, ações menores) × modo interno (todos, com seletor). Requisito de segurança: **todo endpoint que o portal expõe passa pelo mesmo guard de escopo**.

---

## 1. O que foi entregue

**Reuso, não duplicação.** As 5 telas do Acompanhamento passam a existir também sob o Portal do Fornecedor, reusando os mesmos content components internos:

| Rota do Portal | Componente reusado |
|---|---|
| `/portal-fornecedor/acompanhamento/dashboard` | `AcompanhamentoDashboardContent` |
| `/portal-fornecedor/acompanhamento/notas-fiscais` | `AcompanhamentoNotasContent` |
| `/portal-fornecedor/acompanhamento/pagamentos` | `AcompanhamentoPagamentosContent` |
| `/portal-fornecedor/acompanhamento/pendencias` | `AcompanhamentoPendenciasContent` |
| `/portal-fornecedor/acompanhamento/comunicacao` | `AcompanhamentoComunicacaoContent` |

Cada componente recebe um prop `portalModo`, derivado do escopo pela página:
- **participante** (fornecedor no escopo): oculta o seletor de participante (dado já travado no backend) e as **ações internas** (Efetivar pagamento, Resolver no Conecta). Permanecem: leitura, baixar comprovantes e comentar/abrir ticket.
- **interno** (analista, sem escopo) ou ausência do prop: comportamento idêntico ao Acompanhamento interno (seletor visível, ações completas). O Acompanhamento interno de `/gestor-fiscal` fica **inalterado**.

---

## 2. Segurança — todo endpoint do portal sob o guard

No B1, o guard de escopo (`X-Portal-Escopo`) já cobria `dashboard`, `notas-fiscais` (+`/:id`) e `pagamentos` (+`/:id/efetivar`). **B2 estende o MESMO guard aos endpoints restantes que as telas do portal consomem** — nenhuma rota do portal escapa do isolamento:

| Endpoint | Tratamento adicionado |
|---|---|
| `GET /captura-automatica` | filtra pelos documentos do participante em escopo |
| `GET /pendencias` | `participanteId` em `PendenciaDerivada` + filtro por escopo |
| `GET /valores/:integracaoId` | 403 quando o documento é de outro participante |
| `GET /comunicacao` (lista) | `participanteId` em `ThreadResumo` + filtro por escopo |
| `GET /comunicacao/:documentoFiscalId` (detalhe) | 403 quando o documento é de outro participante |
| `POST /comunicacao/:documentoFiscalId/mensagens` | 403 cross-participante (antes do RBAC) |

A regra de resolução do escopo é a mesma do B1: o escopo vem do request (`X-Portal-Escopo: participante:<id>`) e é autoritativo; parâmetros de query não o sobrepõem.

---

## 3. Evidências por endpoint (chamadas e retornos reais)

**Ambiente:** backend MSW em `http://localhost:3001`, `X-Mock-Papel: socio`. **A = `part-1`**, **B = `part-2`**.

### Filtro de leitura sob escopo do fornecedor A

```
GET /captura-automatica                         (sem escopo)          → 200 · 74 itens
GET /captura-automatica  [escopo participante:part-1]                 → 200 · 2 itens

GET /pendencias                                 (sem escopo)          → 200 · 22 (part-1, interno, part-2)
GET /pendencias          [escopo participante:part-1]                 → 200 · 2  (só part-1)

GET /comunicacao                                (sem escopo)          → 200 · 3  (part-1, part-2)
GET /comunicacao         [escopo participante:part-1]                 → 200 · 2  (só part-1)
```

✅ Cada endpoint recorta para o participante em escopo — não vaza documento interno nem de outro fornecedor.

### Bloqueio de recurso de outro participante (403)

```
GET  /valores/integdoc-3            (integ. de B)  [escopo participante:part-1] → 403 "Acesso negado ao recurso de outro participante."
GET  /comunicacao/doc-fiscal-8      (doc de B)     [escopo participante:part-1] → 403 "Acesso negado ao recurso de outro participante."
POST /comunicacao/doc-fiscal-8/mensagens (doc de B)[escopo participante:part-1] → 403 "Acesso negado ao recurso de outro participante."
```

### Controles (acesso próprio permitido)

```
GET /valores/integdoc-1        (integ. de A) [escopo participante:part-1] → 200
GET /comunicacao/<doc de A>    (doc de A)    [escopo participante:part-1] → 200
```

✅ O fornecedor acessa e comenta os próprios documentos; qualquer recurso de outro participante é negado no backend, não na tela.

---

## 4. Evidências de UI (modo participante × interno)

Tela **Pagamentos** do Portal, mesma URL, alternando o escopo pela barra do topo:

| Sinal | Modo participante (escopo A) | Modo interno (sem escopo) |
|---|---|---|
| Seletor "Todos os participantes" | **oculto** | visível |
| Botão **Efetivar** (ação interna) | **oculto** | conforme RBAC/status |
| Selo "Dados isolados" | presente | ausente |
| Linhas na tabela | **2** (só de A) | 5 (todos) |

✅ No modo participante as ações são menores (leitura + baixar comprovantes + comentar); as ações internas somem. No modo interno a tela é a de sempre.

---

## 5. Documentação in-app

- 5 telas reusadas documentadas (`portal-fornecedor/acompanhamento/*`): doc curta e honesta de "reuso sob escopo", com o conceito de escopo por participante e a explicação de por que as ações são menores. Não são placeholders (as telas existem).
- Doc do **Monitor** do Portal atualizada: a FAQ "onde vejo o detalhe" agora aponta o grupo Acompanhamento (deixou de ser "próximo lote").

---

## 6. Verificações

- `tsc --noEmit` limpo · `next lint` sem erros/warnings · console sem erros.
- Zero regressão no Acompanhamento interno (sem o prop, o comportamento é idêntico).

## 7. Próximos lotes

- **B3** — Solicitação de Cadastro (self-service com fila de aprovação, entidade nova) + Cadastros reusados.
- **B4** — Configurações estruturadas + Logs + Tickets.
- **Portal do Cliente** — lote próprio (escopo por `empresa`), com checklist apresentado antes.

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`part-1`, `part-2`, `integdoc-N`, `doc-fiscal-N`) são fixtures do ambiente de desenvolvimento.*
