# RELATÓRIO — Portal do Cliente · Lote BC1

**Data:** 2026-07-31
**Branch:** `feat/portal-cliente`
**Commit:** `7543e89`
**Escopo do lote:** infraestrutura do Portal do Cliente (chave **empresa**) + Monitor de saúde fiscal + **checkpoint de isolamento por empresa, incluindo a prova de que as duas chaves não vazam** (D-cliente-1).

---

## 1. O que foi entregue

**Um guard, duas chaves (D-cliente-1).** O helper `portalEmpresaId(request)` é irmão do `portalParticipanteId`: **mesmo header** `X-Portal-Escopo`, resolve o id **só quando o tipo é `empresa`**. O endpoint de notas passou a honrar **as duas chaves sem cruzar** — escopo `empresa` recorta por `clienteId`; escopo `participante`, por `participanteId`. O `notas/:id` nega **403** tanto cross-participante quanto cross-empresa.

**Casca do Portal do Cliente.** Módulo próprio `portal-cliente` (card do launcher apontado; RBAC já existia), `ClienteScopeBar` (Interno × Cliente + seletor de empresa) e o Monitor.

**Reconciliação de ids de empresa.** Os documentos usam `clienteId` (ex.: `empresa-iguassu-resort`) e o Controle & Compliance usa `empresaId` (ex.: `empresa-iguassu`) para a mesma empresa. Um mapa canônico (`fixtures/portal-cliente.ts`: `clienteId ↔ empresaComplianceId`) reconcilia. Unificação registrada no backlog (item 41).

**Monitor de saúde fiscal — linguagem para o CLIENTE LEIGO.** Sem jargão de escrituração; destaques em tom de tranquilidade/alerta e cartões de resumo. Agrega obrigações, CND federal, apuração do mês e documentos, sob escopo empresa. Exemplos reais renderizados: *"Certidão federal (CND) válida até 27/10/2026"*, *"Apuração de abril de 2026: nada a recolher"*.

---

## 2. Checkpoint de isolamento por empresa

**Ambiente:** MSW em `http://localhost:3001`, `X-Mock-Papel: socio`. **Empresa A = `empresa-metalurgica-pomin`**, **Empresa B = `empresa-iguassu-resort`**. Baseline sem escopo: 48 notas.

### (a) Recorte — empresa A só vê A
```
GET /acompanhamento/notas-fiscais  [X-Portal-Escopo: empresa:empresa-metalurgica-pomin] → 200 · 19 notas
GET /acompanhamento/notas-fiscais  [X-Portal-Escopo: empresa:empresa-iguassu-resort]    → 200 · 11 notas
Conjuntos A e B DISJUNTOS (nenhum id em comum).
```

### (b) Acesso direto negado (403)
```
GET /acompanhamento/notas-fiscais/integdoc-1  (nota da empresa B)  [escopo empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
GET /acompanhamento/notas-fiscais/<nota de A> (própria)            [escopo empresa:A] → 200
```

### (c) Não vê interno nem outra organização
```
Baseline (sem escopo): 48   →   escopo empresa:A: 19   (exclui interno + empresa B)
```
Multi-tenant (`organizacaoId`) permanece respeitado por baixo.

### (d) Escopo autoritativo sobre a query
```
GET /portal-cliente/monitor?empresaId=empresa-iguassu-resort  [escopo empresa:empresa-metalurgica-pomin]
   → empresa retornada: empresa-metalurgica-pomin   (o escopo VENCE a query)
```

### (D-cliente-1) As duas chaves NÃO vazam uma na outra
```
Escopo EMPRESA:A  → recorta por clienteId; participantes na resposta: [ part-2, (sem) ]
   → prova que empresa NÃO aplica filtro de participante (traz participantes variados dentro da empresa A)

Escopo PARTICIPANTE:part-2 → 4 notas, todas de part-2
   → prova que participante NÃO aplica filtro de empresa (traz empresas variadas do participante)

Monitor do Cliente sob escopo PARTICIPANTE (participante:part-2) → 400 (empresa vazia)
   → o endpoint do Cliente não responde à chave do Fornecedor; um portal não enxerga o outro
```

✅ Cada chave filtra pela sua dimensão; nenhuma vê os dados da outra. O mesmo endpoint atende os dois portais sem sobreposição.

---

## 3. Evidência de UI

Modo Cliente + empresa Metalúrgica Pomin: título **"Minha Saúde Fiscal"**, selo **"Dados isolados"**, destaques em linguagem para leigo (CND, apuração) e cartões de resumo (vencem esta semana / em atraso / certidão federal / apuração do mês). No modo interno, o analista escolhe a empresa no seletor.

`tsc --noEmit` limpo · `next lint` sem erros/warnings · console sem erros.

---

## 4. Próximos lotes (aprovados)

- **BC2** — Obrigações + Certidões sob escopo (reuso read-only + baixar).
- **BC3** — Apurações (resumo read-only) + Meus Documentos (recebidos/emitidos) sob escopo.
- **BC4** — Comunicação + Tickets sob escopo; docs; fechamento.

Cada lote mantém o guard universal provado por endpoint (recorte + 403 + não-vazamento das chaves).

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`empresa-*`, `integdoc-N`, `part-N`) são fixtures do ambiente de desenvolvimento.*
