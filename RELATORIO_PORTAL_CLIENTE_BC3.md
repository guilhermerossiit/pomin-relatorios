# RELATÓRIO — Portal do Cliente · Lote BC3

**Data:** 2026-07-31
**Branch:** `feat/portal-cliente`
**Commit:** `d17b6c9`
**Escopo do lote:** Apurações (RESUMO read-only) + Meus Documentos (recebidos/emitidos), sob escopo por empresa, linguagem para o cliente leigo.

---

## 1. O que foi entregue

**Meus Impostos (Apurações — RESUMO, D-cliente-2).** O cliente vê **quanto foi apurado** por imposto e competência e baixa a **guia** quando houver. **Nunca** a estação de trabalho, ajustes ou mecânica de escrituração. Linguagem para leigo: imposto × competência × "a recolher"/"nada a recolher" — **sem** CST/CFOP/débito×crédito.

**Meus Documentos.** As notas recebidas/emitidas da empresa, read-only, com **baixar XML/DANFE**. Reusa o **endpoint de duas chaves do BC1** (`/acompanhamento/notas-fiscais`), agora sob escopo empresa (filtra por `clienteId`).

---

## 2. Segurança — endpoints sob o guard universal

| Endpoint | Regra |
|---|---|
| `GET /portal-cliente/apuracoes` | filtra pela empresa em escopo; participante/sem-escopo → 400 |
| `GET /portal-cliente/apuracoes/:id/guia` | 403 se a apuração é de outra empresa; 404 se ainda não há guia |
| `GET /acompanhamento/notas-fiscais` (reuso BC1) | filtra por `clienteId` no escopo empresa; `:id` já nega 403 cross-empresa |

---

## 3. Evidências por endpoint (reais)

**Empresa A = `empresa-metalurgica-pomin`**, **B = `empresa-iguassu-resort`**.

```
Apurações (resumo)
  GET /portal-cliente/apuracoes  [escopo empresa:A] → 200 · 9  (ex.: "CBS · abril de 2026", "COFINS · abril de 2026")
  GET /portal-cliente/apuracoes  [escopo empresa:B] → 200 · 1
  GET /portal-cliente/apuracoes/<apuração de B>/guia  [escopo empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
  GET /portal-cliente/apuracoes/<apuração de A>/guia  [escopo empresa:A] → 404 (sem guia gerada — NÃO 403; o escopo permite)
  GET /portal-cliente/apuracoes  [escopo participante:part-2] → 400  (chave errada rejeitada)

Meus Documentos (reuso do endpoint de duas chaves do BC1)
  GET /acompanhamento/notas-fiscais  [escopo empresa:A] → 200 · 19
```

✅ **Recorte** por empresa; **403** ao acessar guia de outra empresa; **chave errada (participante) → 400**. A guia própria retorna 404 quando ainda não foi gerada (honesto: "baixar guia quando houver"), nunca 403.

---

## 4. Evidência de UI

- **Meus Impostos:** título client-friendly, 9 linhas (empresa A), colunas Imposto/Competência/A recolher/Guia, "nada a recolher" quando zero — **sem jargão** (verificado: nenhum CST/CFOP/débito/crédito na tela).
- **Meus Documentos:** 19 linhas (empresa A), botões XML/DANFE por nota.

`tsc`/`next lint`/console limpos.

---

## 5. Próximo lote

- **BC4** — Comunicação + Tickets sob escopo empresa; docs; **fechamento do Portal do Cliente** (com status de fechamento do ciclo para o PO decidir merge + tag + deploy).

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`empresa-*`, `apuracao-*`, `part-N`) são fixtures do ambiente de desenvolvimento.*
