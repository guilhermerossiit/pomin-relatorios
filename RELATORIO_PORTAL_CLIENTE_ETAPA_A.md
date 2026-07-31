# RELATÓRIO — Portal do Cliente · ETAPA A (checklist + escopo)

**Data:** 2026-07-31
**Branch:** `feat/portal-cliente` (a partir da `main`)
**Natureza:** análise e proposta. **Nenhum código foi escrito.** Aguardando aprovação do PO para implementar.

O Portal do Cliente é um **módulo SEPARADO** do Portal do Fornecedor, reusando a mesma infraestrutura de escopo — **a chave aqui é `empresa`**, não `participante`. A persona é a **empresa-cliente do escritório**, que LÊ a própria saúde fiscal e o trabalho do escritório e se comunica; **não opera o trabalho interno**.

---

## 1. Como a infra de escopo se aplica com a chave `empresa`

A infra do ciclo anterior já foi desenhada em B1 para os dois portais. O que é **reuso direto** e o que precisa de **adaptação**:

### Reuso direto (sem tocar)
- **`lib/portal/portal-scope.ts`** — já define `EscopoPortal { tipo: 'participante' | 'empresa'; id }` e o header único **`X-Portal-Escopo: <tipo>:<id>`**. Para o cliente: `empresa:<empresaId>`. Zero mudança.
- **`lib/portal/use-portal-scope.ts`** — hook genérico ao escopo; serve aos dois portais.
- **`apiFetch`** — anexa `X-Portal-Escopo` automaticamente. Zero mudança.
- **Padrão de casca** — módulo no registry + `layout.tsx` com `<ModuleShell>` + barra de escopo no topo + Monitor. Mesmo molde do Fornecedor.
- **Modo interno × externo** — conceito idêntico: interno = analista com visão plena (seletor de empresa); externo = travado numa empresa.

### Adaptação (novo, pequeno e localizado)
- **Helper de guard `portalEmpresaId(request)`** — irmão do `portalParticipanteId`. Lê o mesmo header e resolve o id **só quando `tipo === 'empresa'`**. Em B1 já ficou provado que o guard do Fornecedor **ignora** um escopo `empresa:` (discriminação por tipo) — então os dois portais coexistem sem interferência. Este helper é o núcleo do B1 do cliente.
- **Aplicar o guard aos endpoints do cliente** — os endpoints de Obrigações, Certidões, Apurações e Documentos **já têm a chave de filtro** (`empresaId`/`clienteId` são campos first-class); falta honrarem o escopo `empresa` (filtro + 403 cross-empresa), exatamente como o B2 fez no Acompanhamento.
- **Módulo próprio `portal-cliente`** — o card já existe no launcher (hoje "Em breve") e o RBAC `portal-cliente` já está no catálogo. Vira módulo com rota `/portal-cliente`.
- **`ClienteScopeBar`** — barra Interno × Cliente com seletor de **empresa** (alimentado pela lista de empresas do tenant — fixture `admin-empresas`). Análoga à `FornecedorScopeBar`.

### Observação importante (não confundir dois mecanismos)
O app já tem um **seletor de empresa interno** (multi-tenant / workspace switcher — `context-switcher.tsx`) para o analista. O **escopo do Portal** (`X-Portal-Escopo: empresa:<id>`) é **outro mecanismo**, autoritativo e independente: representa o cliente externo travado na própria empresa. O guard do portal não depende do switcher interno.

---

## 2. Levantamento — o que o Pomin já tem na ótica do CLIENTE

A chave `empresa` é **first-class** em quase tudo (`empresaId`/`clienteId`), o que torna o recorte **mais direto que o do participante** (que dependia de `participanteDoDocumento`). Para cada área:

| Área (origem interna) | Chave de escopo | Reuso ou nova? |
|---|---|---|
| **Obrigações** (Controle & Compliance › Obrigações) | `Obrigacao.empresaId` | **Reuso** de tela sob novo escopo (read-only) |
| **Certidões (CND)** (Controle & Compliance › Certidões) | `Certidao.empresaId` | **Reuso** (read-only + baixar) |
| **Apurações** (Gestor Fiscal › Apurações) | `Apuracao.clienteId` | **Reuso** — versão resumo read-only (ver D-cliente-2) |
| **Documentos recebidos/emitidos** (Jornada/Acompanhamento) | `DocumentoFiscal.clienteId` | **Reuso** dos endpoints, agora scope-aware por empresa (ver D-cliente-1) |
| **Comunicação / Suporte** | por empresa | **Reuso** dos Tickets do Fornecedor (B4) sob chave empresa + comunicação por documento |
| **Monitor de saúde fiscal** | agrega o acima | **Nova composição** (reusa KPIs existentes) sob escopo empresa |

Nada disso é "tela nova de negócio": é **reuso das telas internas sob o escopo empresa**, no mesmo padrão que funcionou no Fornecedor (prop de modo + guard no backend). A única superfície realmente nova é a **composição** do Monitor de saúde fiscal do cliente.

---

## 3. Menu proposto do Portal do Cliente

Tudo **filtrado pela empresa do cliente logado**. Ações do modo externo: **LER** a situação, **BAIXAR** (certidões, guias, documentos) e **COMUNICAR** — nunca operar trabalho interno (não escritura, não apura, não dá baixa, não altera obrigação/certidão).

| Tela | Escopo (externo) | Ações permitidas (modo cliente) |
|---|---|---|
| **Monitor** (saúde fiscal) | empresa | Ler KPIs: obrigações pendentes/atrasadas, certidões a vencer, apuração do mês, documentos recentes |
| **Obrigações** | empresa | Ler status/prazo/competência; baixar comprovante de entrega |
| **Certidões** | empresa | Ler validade/status; **baixar** a certidão (PDF) |
| **Apurações** | empresa | Ler o apurado por tributo/competência; baixar guia (quando houver) |
| **Meus Documentos** (recebidos/emitidos) | empresa | Ler a lista/jornada; baixar XML/DANFE |
| **Comunicação** | empresa | Ler e responder a conversa por documento (comentar) |
| **Tickets** | empresa | Abrir/acompanhar chamado; responder |

Fora do escopo do cliente (permanecem internos): escriturar, apurar, fechar competência, editar obrigação/certidão, aprovar solicitações, configurar. No **modo interno** (analista) o portal mostra tudo com seletor de empresa, para visualizar "como o cliente vê".

### Decisões que dependem do PO (antes de eu codar)
- **D-cliente-1 (Documentos):** reusar os **mesmos** endpoints do Acompanhamento tornando-os scope-aware para as duas chaves (participante *e* empresa), **ou** endpoints dedicados do cliente? *Recomendação:* um guard, duas chaves — o endpoint honra `participante:` ou `empresa:` conforme o header.
- **D-cliente-2 (Apurações):** expor o **resumo read-only** (quanto foi apurado + guias) — **não** a estação de trabalho de apuração. *Recomendação:* resumo, não workstation.
- **D-cliente-3 (Comunicação):** Tickets (reuso do B4, por empresa) como canal principal **+** comunicação por documento? *Recomendação:* ambos, ambos read/responder.
- **D-cliente-4 (Solicitações):** o cliente precisa de um "Solicitar 2ª via / documento" (fila de aprovação, como o Fornecedor)? *Recomendação:* fora do v1; avaliar depois.
- **D-cliente-5 (Auth):** mesma linha do Fornecedor — "ver como empresa" interno agora; login externo real (Keycloak) no item 38 do backlog (cobre os dois portais).

---

## 4. Cenários de isolamento a blindar (os mesmos 4 tipos, agora por empresa)

Serão provados por endpoint em cada lote, como no Fornecedor. Com empresas **A** e **B** (dois clientes distintos):

- **(a) Recorte** — escopo `empresa:A` retorna só dados da empresa A (obrigações, certidões, apurações, documentos filtrados por `empresaId/clienteId === A`); nada de B nem interno.
- **(b) Acesso direto negado** — `GET` de um recurso da empresa B (id de B) sob escopo A → **403**; recurso próprio → 200; inexistente → 404.
- **(c) Não vê interno nem outra organização** — o recorte por empresa exclui a visão plena e qualquer outra empresa; e o **tipo** `empresa` não é confundido com `participante` (discriminação de tipo, já provada no Fornecedor). Multi-tenant (`organizacaoId`) permanece respeitado por baixo.
- **(d) Escopo autoritativo** — `?empresaId=B` na query enquanto o escopo é A → **ignorado**; a resposta traz só A. O escopo vence o parâmetro do cliente (mesmo padrão `escopo ?? query` do Fornecedor).

---

## 5. Plano de lotes proposto (para sua validação)

- **BC1** — Infra do cliente: helper `portalEmpresaId` + módulo `portal-cliente` + `ClienteScopeBar` (Interno × Cliente) + Monitor de saúde fiscal + checkpoint de isolamento (4 cenários, provado por endpoint).
- **BC2** — Obrigações + Certidões sob escopo (reuso read-only + baixar).
- **BC3** — Apurações (resumo) + Meus Documentos (recebidos/emitidos) sob escopo.
- **BC4** — Comunicação + Tickets sob escopo; docs in-app; fechamento.

Cada lote: reuso sob escopo, **guard universal provado por endpoint**, typecheck/lint, docs in-app, relatório no repo público com evidências, mini-resumo com URL raw e PARADA para validação.

---

## PARADA

Este é o checklist da Etapa A. **O conteúdo do Portal do Cliente é decisão do PO.** Não implemento nada até sua aprovação — em especial das decisões **D-cliente-1 a D-cliente-5** e do **menu/plano de lotes** acima.

*Relatório de análise. Não contém credenciais, tokens nem dados de clientes reais — nomes de empresa/identificadores citados (A, B, `empresaId`, `clienteId`) são referências de modelo/fixtures do ambiente de desenvolvimento.*
