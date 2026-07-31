# RELATÓRIO — Portal do Fornecedor · Lote B4 (fechamento do ciclo)

**Data:** 2026-07-31
**Branch:** `feat/portal-fornecedor`
**Commit:** `f991b25`
**Escopo do lote:** Configurações estruturadas + Logs (Integração/Atividade) + Tickets de suporte. **Este lote fecha o Portal do Fornecedor.**

---

## 1. O que foi entregue

**Configurações estruturadas (D4).** Catálogo de campos por entidade (Participante e Produto) com atributos **obrigatório** e **visível**. É a *definição estruturada* dos campos — **sem motor de formulário dinâmico** nesta fase. Configurar é ação do analista; o fornecedor vê a tela em leitura.

**Logs sob escopo.** Duas abas:
- **Integração** — status da integração de cada documento com o ERP (protocolo/motivo de falha), derivado do estado real.
- **Atividade** — histórico da jornada do documento.
O fornecedor vê apenas os logs dos próprios documentos; o interno vê todos.

**Tickets de suporte** (entidade nova `TicketSuporte` + `TicketMensagem`). Abrir/acompanhar chamado, conversa em thread, resposta e mudança de status. O fornecedor abre e responde os próprios; **mudar status é exclusivo do suporte interno**.

---

## 2. Segurança — todos os endpoints novos sob o guard

| Endpoint | Regra de escopo |
|---|---|
| `GET /portal/configuracoes/campos` | leitura do catálogo |
| `PUT /portal/configuracoes/campos/:id` | INTERNO — 403 para escopo de participante |
| `GET /portal/logs/integracao` | filtra pelos documentos do escopo |
| `GET /portal/logs/atividade` | filtra pelos documentos do escopo |
| `GET /portal/tickets` | participante vê só os seus; interno vê todos |
| `POST /portal/tickets` | `participanteId` vem do ESCOPO |
| `POST /portal/tickets/:id/mensagens` | 403 em chamado de outro participante |
| `POST /portal/tickets/:id/status` | INTERNO — 403 para escopo de participante |

---

## 3. Evidências por endpoint (chamadas e retornos reais)

**Ambiente:** MSW em `http://localhost:3001`, `X-Mock-Papel: socio`. **A = `part-1`**, **B = `part-2`**.

```
Configurações
  GET  /portal/configuracoes/campos                                  → 200 · 8 campos
  PUT  /portal/configuracoes/campos/cfg-prod-unidade  [interno]      → 200
  PUT  /portal/configuracoes/campos/cfg-prod-unidade  [escopo A]     → 403 "Apenas o analista interno pode configurar."

Logs (sob escopo)
  GET  /portal/logs/integracao   [interno]                           → 200 · 48
  GET  /portal/logs/integracao   [escopo participante:part-1]        → 200 · 2  (só part-1)
  GET  /portal/logs/atividade    [escopo participante:part-1]        → 200 · 7  (só part-1)

Tickets
  GET  /portal/tickets  [escopo A] → tkt-1, tkt-2      |  [escopo B] → tkt-3   |  [interno] → 3
  POST /portal/tickets  [escopo A] { assunto, categoria, descricao } → 201 · participanteId "part-1" · protocolo gerado
  POST /portal/tickets/tkt-3/mensagens  [escopo A]  (tkt-3 é de B)   → 403
  POST /portal/tickets/tkt-1/status     [escopo A]                   → 403
  POST /portal/tickets/tkt-1/status     [interno] { status:'resolvido' } → 200 · status "resolvido"
```

✅ Cada leitura recorta para o escopo; ações internas (configurar, mudar status) e escrita em recurso de outro participante são negadas (403) no backend.

---

## 4. Evidências de UI

- **Configurações** (interno): aviso de "definição estruturada, sem construtor de formulários"; alternadores Obrigatório/Visível por campo, agrupados por entidade. No modo participante, somente leitura.
- **Logs**: abas Integração/Atividade; tabela com quando/documento/evento/detalhe/nível.
- **Tickets**: lista + "Abrir chamado" (dialog) + conversa em thread + resposta; seletor de status só no modo interno.

Menu do Portal fechado: Monitor · Acompanhamento (5) · Solicitação de Cadastro · Cadastros (2) · Tickets · Logs · Configurações. Console sem erros; `tsc` e `next lint` limpos.

---

## 5. Fechamento do ciclo — Portal do Fornecedor

### Pronto (nesta feature branch)
- **Infra compartilhada de escopo** (B1): header genérico `X-Portal-Escopo` (`<tipo>:<id>`), guard real no backend, casca do Portal (módulo, sidebar, barra Interno × Fornecedor, Monitor).
- **Acompanhamento reusado** (B2): Dashboard, Notas Fiscais, Pagamentos, Pendências, Comunicação — sob escopo, ações menores no modo participante.
- **Solicitação de Cadastro** (B3): self-service com fila de aprovação (entidade nova), aprovar→cadastro real, recusar com motivo; Cadastros read-only (Meus Dados, Meus Produtos).
- **Configurações + Logs + Tickets** (B4): este lote.
- **Guard universal**: todos os endpoints do portal (leitura e escrita) sob o mesmo guard — provado por endpoint em cada lote (recorte + 403).
- **Docs in-app** de todas as telas do Portal.

### Foi para o backlog (não entrou por decisão de fase)
- **Portal — Autenticação Externa (Keycloak)** — item 38. Hoje o escopo é escolhido internamente; a identidade externa real usa o mesmo guard.
- **Motor de formulário dinâmico** nas Configurações (criar campos/forms arbitrários) — D4 limitou a esta fase a definição estruturada.
- **Logs/Integração reais** — hoje derivados do estado mock; quando `apps/api` virar backend real, passam a vir de eventos reais.
- **Portal do Cliente** — módulo SEPARADO (escopo por `empresa`), lote próprio; **antes de qualquer código, checklist específico para aprovação** (conteúdo ainda não especificado).

### Pronto para o Product Owner decidir
Merge de `feat/portal-fornecedor` na `main` (--no-ff), tag (ex.: `portal-fornecedor-concluido`) e deploy — a critério do PO.

---

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`part-1`, `part-2`, `tkt-N`, `cfg-*`) são fixtures do ambiente de desenvolvimento.*
