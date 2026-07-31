# RELATÓRIO — Portal do Fornecedor · Lote B1

**Data:** 2026-07-30
**Branch:** `feat/portal-fornecedor`
**Commits:** `97dc178` (B1 inicial) → `6aed297` (ajuste D3 — dois portais separados)
**Escopo do lote:** infraestrutura compartilhada de escopo dos Portais + casca do Portal do Fornecedor (módulo, sidebar, barra Interno × Fornecedor, Monitor) + **guard de isolamento REAL no backend, provado por endpoint**.

---

## 1. O que foi entregue

- **Infra compartilhada de escopo** (`lib/portal/portal-scope.ts`): tipo `EscopoPortal { tipo: 'participante' | 'empresa'; id }` e header único **`X-Portal-Escopo`** no formato `<tipo>:<id>` (ex.: `participante:part-1`). Mesma mecânica para os dois portais; muda só a chave (`participante` no Fornecedor; `empresa` no futuro Portal do Cliente).
- **Envio automático** (`lib/api/client.ts`): o `apiFetch` anexa `X-Portal-Escopo` a toda requisição quando há escopo ativo.
- **Guard no backend** (`mocks/handlers-fiscal.ts`): os endpoints do Acompanhamento (`/participantes`, `/dashboard`, `/notas-fiscais` e `/notas-fiscais/:id`, `/pagamentos` e `/pagamentos/:id/efetivar`) filtram pelo participante em escopo e **negam (403)** o acesso a recurso de outro participante.
- **Casca do Portal do Fornecedor**: módulo próprio `portal-fornecedor` (rota `/portal-fornecedor`), card no launcher, RBAC, barra **Interno × Fornecedor** e **Monitor** (KPIs sensíveis ao escopo).

### Mecanismo do isolamento (por que a query não vaza)

O escopo é **autoritativo** e vence qualquer `participanteId` recebido por query. No handler:

```ts
// O escopo (do request) tem precedência; o query participanteId só vale quando NÃO há escopo.
const participanteId = portalParticipanteId(request) ?? (url.searchParams.get('participanteId') ?? '');
```

Quando há escopo, o `??` curto-circuita e o `participanteId` da query **nunca é consultado**. Hoje o escopo chega pelo header `X-Portal-Escopo` (definido internamente pelo analista, pois ainda não há login externo — ver backlog item 38 "Portal — Autenticação Externa"). Quando a sessão externa real existir, o escopo passa a ser lido da sessão no servidor, com **o mesmo guard** — a propriedade provada abaixo (cliente não sobrepõe o escopo) permanece.

---

## 2. Checkpoint de isolamento — evidências por endpoint

**Ambiente:** backend MSW em `http://localhost:3001`, sessão `X-Mock-Papel: socio`.
**Massa de teste:** dois fornecedores vinculados a documentos reais do Acompanhamento — **A = `part-1`** (Têxtil Cataratas, 2 notas) e **B = `part-2`** (Suprimentos TF, 4 notas). Baseline interno (sem escopo): **48 notas** no total, distribuídas entre `part-1`, `part-2` e documentos internos sem fornecedor vinculado.

```
Baseline (sem escopo):
GET /v1/fiscal/acompanhamento/notas-fiscais            → 200 · 48 notas
  participantes distintos: [ part-1, part-2, (interno/sem-fornecedor) ]
```

### (a) Escopo do fornecedor A retorna só dados de A

```
Requisição:
  GET /v1/fiscal/acompanhamento/notas-fiscais
  X-Portal-Escopo: participante:part-1

Resposta:
  200 · 2 notas
  participantes distintos: [ part-1 ]
```

✅ Sob escopo de A, a listagem cai de 48 → **2**, todas de `part-1`. Nenhuma nota de B nem interna aparece.

### (b) GET de recurso do fornecedor B (id de B) sob escopo A → 403 (e 404 para inexistente)

```
Requisição (recurso de B, sob escopo de A):
  GET /v1/fiscal/acompanhamento/notas-fiscais/integdoc-3   (integdoc-3 é nota de part-2 = B)
  X-Portal-Escopo: participante:part-1

Resposta:
  403 · { "message": "Acesso negado ao recurso de outro participante." }

Controle — recurso inexistente, sob escopo de A:
  GET /v1/fiscal/acompanhamento/notas-fiscais/doc-inexistente-999
  X-Portal-Escopo: participante:part-1
  → 404

Controle — recurso PRÓPRIO (nota de A), sob escopo de A:
  GET /v1/fiscal/acompanhamento/notas-fiscais/integdoc-1   (integdoc-1 é nota de part-1 = A)
  X-Portal-Escopo: participante:part-1
  → 200
```

✅ Acesso direto por id ao recurso de outro fornecedor é **negado (403)**; id inexistente devolve **404**; o próprio recurso continua acessível (**200**). O 403 é decisão de backend, não de UI.

### (c) Escopo do fornecedor não vê dados internos nem de outra organização

```
Requisição:
  GET /v1/fiscal/acompanhamento/dashboard?periodo=mensal
  X-Portal-Escopo: participante:part-1

Resposta:
  200 · 2 documento(s) no escopo
  participantes distintos: [ part-1 ]
  kpis: { recebimento:0, escrituracao:0, pgtoProgramado:1, pgtoRealizado:0, rejeitadas:1, aguardandoCorrecao:0 }
```

✅ O painel de A enxerga só os **2** documentos de A. Ficam de fora tanto os documentos **internos** (sem fornecedor) quanto os do **outro fornecedor** (B, `part-2`) — do universo de 48, restam 2. Os KPIs refletem apenas o recorte de A.

**Discriminação por tipo de escopo** (o guard do Fornecedor não age sobre escopo de outro tipo):

```
GET /v1/fiscal/acompanhamento/notas-fiscais
X-Portal-Escopo: empresa:org-outra
  → 200 · 48 notas (não filtra)
```

✅ Um escopo de **tipo `empresa`** (a chave que o futuro Portal do Cliente usará) **não** é interpretado como participante — o guard do Fornecedor só atua sobre `participante:`. Isso garante que cada portal filtra pela sua própria chave, sem sobreposição.

### (d) participanteId de B enviado enquanto o escopo é A → ignorado (o escopo vence), sem vazamento

```
Requisição (query tenta forçar B, mas o escopo é A):
  GET /v1/fiscal/acompanhamento/notas-fiscais?participanteId=part-2
  X-Portal-Escopo: participante:part-1

Resposta:
  200 · 2 notas
  participantes distintos: [ part-1 ]
  vazou dados de B? NÃO

Contraste — a MESMA query, mas SEM escopo (para provar que o parâmetro funciona quando permitido):
  GET /v1/fiscal/acompanhamento/notas-fiscais?participanteId=part-2
  (sem X-Portal-Escopo)
  → 200 · 4 notas · participantes distintos: [ part-2 ]
```

✅ Com escopo de A ativo, o `participanteId=part-2` da query é **ignorado**: a resposta traz apenas as 2 notas de A. O escopo é autoritativo e **não pode ser sobreposto** por parâmetro do cliente. O contraste confirma que a query só recorta quando **não** há escopo (aí, 4 notas de B) — ou seja, o filtro existe e funciona; o que o escopo faz é **travá-lo** no participante correto.

---

## 3. Conclusão do checkpoint

| Cenário | Requisição-chave | Esperado | Obtido |
|---|---|---|---|
| (a) escopo A só vê A | `GET /notas-fiscais` + escopo A | só `part-1` | ✅ 2 notas, só `part-1` |
| (b) recurso de B por id sob escopo A | `GET /notas-fiscais/integdoc-3` + escopo A | 403 | ✅ 403 (e 404 p/ inexistente, 200 p/ próprio) |
| (c) escopo A não vê interno/outra org | `GET /dashboard` + escopo A | só `part-1` | ✅ 2 de 48, só `part-1` |
| (d) query B sob escopo A | `GET /notas-fiscais?participanteId=part-2` + escopo A | ignora query | ✅ 2 notas de A, sem vazar B |

O isolamento de dados do Portal do Fornecedor é **real no backend** e resistente a parâmetro malicioso do cliente. A ausência de login externo (fase atual) **não** relaxa o isolamento — o escopo é a fronteira; a identidade externa (Keycloak) é evolução registrada, com o mesmo guard.

---

## 4. Próximos lotes

- **B2** — reuso das telas do Acompanhamento dentro do Portal, sob o mesmo escopo por participante.
- **B3** — Solicitação de Cadastro (self-service com aprovação) + Cadastros.
- **B4** — Configurações estruturadas + Logs + Tickets.
- **Portal do Cliente** — lote próprio (escopo por `empresa`), reusando esta infra; checklist a apresentar antes.

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`part-1`, `part-2`, `integdoc-N`) são fixtures do ambiente de desenvolvimento.*
