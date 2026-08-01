# RELATÓRIO — Portal do Cliente · Lote BC4 (fechamento do ciclo)

**Data:** 2026-07-31
**Branch:** `feat/portal-cliente`
**Commit:** `c1dcac6`
**Escopo do lote:** Comunicação por documento + Tickets de suporte, sob escopo empresa. **Este lote fecha o Portal do Cliente.**

---

## 1. O que foi entregue

**Comunicação por documento** (reuso da tela do Acompanhamento): o cliente lê e responde as conversas dos documentos da sua empresa. Os endpoints de comunicação passaram a honrar a chave `empresa` (thread.empresaId = doc.clienteId) na lista, no detalhe e na escrita.

**Tickets de suporte** (reuso do `TicketsContent`, generalizado para externo = fornecedor OU cliente): abrir/acompanhar/responder chamados. `TicketSuporte` ganhou `empresaId`; os endpoints honram as duas chaves.

**Escrita amarrada ao escopo (requisito central do BC4).** Ao abrir um ticket ou responder, o dono (empresa/participante) vem do **escopo da sessão**, nunca do corpo da requisição. **Mudar o status** de um ticket é ação **interna** — negada (403) ao cliente.

**Linguagem para leigo.** "Falar com o escritório", "Abrir chamado", status como etiqueta legível.

---

## 2. Evidências por endpoint (empresa A = `empresa-metalurgica-pomin`, B = `empresa-iguassu-resort`)

```
Tickets — recorte por chave (um guard, duas chaves)
  GET /portal/tickets  [empresa:A]          → [tkt-c1]
  GET /portal/tickets  [empresa:B]          → [tkt-c2]
  GET /portal/tickets  [participante:part-1]→ [tkt-1, tkt-2]   (fornecedor — não vê tickets de cliente)

Escrita amarrada ao ESCOPO (não ao body) — o requisito central
  POST /portal/tickets  [empresa:A]  body { empresaId: B, participanteId: part-2, ... }
    → 201 · empresaId "empresa-metalurgica-pomin" (A) · participanteId "" · BODY IGNORADO

  POST /portal/tickets/tkt-c2/mensagens  (ticket de B)  [empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
  POST /portal/tickets/tkt-c1/status     [empresa:A]  → 403 "Apenas o suporte interno altera o status."
  POST /portal/tickets/tkt-c1/status     [interno]     → 200

Comunicação por documento
  GET  /acompanhamento/comunicacao        [interno]   → 3
  GET  /acompanhamento/comunicacao        [empresa:A] → 1   (só empresa-metalurgica-pomin)
  GET  /acompanhamento/comunicacao/<doc de outra empresa>  [empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
```

✅ Recorte por empresa; 403 cross-empresa na leitura e na escrita; **o vínculo da escrita vem do escopo, não do body**; status de ticket é privilégio interno.

---

## 3. Evidência de UI

Tickets do cliente (modo Cliente + Metalúrgica Pomin): "Tickets de Suporte", "Abrir chamado", o chamado da empresa (protocolo 2026-000201) na lista; o cliente vê o status como etiqueta (não troca). Comunicação reusa a tela do Acompanhamento sob escopo. `tsc`/`next lint`/console limpos.

---

## 4. 🏁 Fechamento do ciclo — Portal do Cliente

### Pronto (na branch `feat/portal-cliente`)
- **BC1** — infra `portalEmpresaId` (um guard, duas chaves) + módulo `portal-cliente` + `ClienteScopeBar` + **Monitor de saúde fiscal** em linguagem para leigo; checkpoint de isolamento por empresa + prova de que as duas chaves não vazam.
- **BC2** — **Obrigações** + **Certidões** read-only (ler + baixar comprovante/PDF), via o mapa canônico de ids.
- **BC3** — **Apurações (resumo)** (quanto a recolher + baixar guia, sem jargão) + **Meus Documentos** (recebidos/emitidos, baixar XML/DANFE), reusando o endpoint de duas chaves.
- **BC4** — **Comunicação** + **Tickets**; escrita amarrada ao escopo; status interno.
- **Guard universal** — todo endpoint (leitura e escrita) sob o mesmo guard, provado por endpoint em cada lote (recorte + 403 + não-vazamento das chaves + escrita amarrada ao escopo).
- **Linguagem para o cliente leigo** em todas as telas. **Docs in-app** de todas.

### Foi para o backlog (por decisão de fase)
- **Item 38** — Autenticação Externa (Keycloak) — cobre os dois portais.
- **Item 41 (MÉDIA)** — unificar os ids de empresa entre módulos (hoje reconciliados pelo mapa canônico; regra: todo lote que toca empresa usa o mapa).
- **Item 42 (BAIXA)** — "Solicitar 2ª via / documento" self-service (D-cliente-4, fora do v1).

### Pronto para o Product Owner decidir
Merge de `feat/portal-cliente` na `main` (--no-ff), tag (ex.: `portal-cliente-concluido`) e deploy — a critério do PO. Com isso, **os dois portais (Fornecedor e Cliente) ficam completos**.

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`empresa-*`, `tkt-*`, `part-N`) são fixtures do ambiente de desenvolvimento.*
