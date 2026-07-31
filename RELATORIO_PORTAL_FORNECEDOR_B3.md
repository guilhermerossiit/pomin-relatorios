# RELATÓRIO — Portal do Fornecedor · Lote B3

**Data:** 2026-07-31
**Branch:** `feat/portal-fornecedor`
**Commit:** `20e1ef1`
**Escopo do lote:** Solicitação de Cadastro (self-service com fila de aprovação — entidade nova) + Cadastros reusados em somente leitura. Requisito mantido: **todo endpoint novo passa pelo mesmo guard de escopo**.

---

## 1. O que foi entregue

**Solicitação de Cadastro (self-service com aprovação humana).** O fornecedor PROPÕE o cadastro de um produto ou a atualização dos próprios dados; a proposta entra numa **fila** e **nada vira cadastro real sem aprovação do analista** (padrão da casa: automação propõe, humano decide).

- **Entidade nova** `SolicitacaoCadastro` (tipo `participante`/`produto`, operação `criacao`/`atualizacao`, status `pendente`/`aprovada`/`recusada`, `dados` propostos, `motivoRecusa`, `registroGeradoId`).
- **Aprovar** gera o registro real reusando o estado de **Participante/Produto** (+ vínculo Fornecedor×Produto), com log/auditoria; **recusar** grava o motivo (visível ao fornecedor).

**Cadastros reusados (somente leitura, sob escopo):** o fornecedor vê os próprios dados de Participante (Meus Dados) e os Produtos vinculados a ele (Meus Produtos). Para alterar, abre uma Solicitação.

**Telas:** `Solicitação de Cadastro` (form + fila própria no modo participante; fila + Aprovar/Recusar no modo interno), `Cadastros › Meus Dados`, `Cadastros › Meus Produtos`.

---

## 2. Segurança — todos os endpoints novos sob o guard

| Endpoint | Regra de escopo |
|---|---|
| `GET /v1/fiscal/portal/solicitacoes` | participante vê só as suas; interno vê a fila toda |
| `POST /v1/fiscal/portal/solicitacoes` | `participanteId` vem do **ESCOPO**, nunca do body — não dá para solicitar em nome de outro |
| `POST /v1/fiscal/portal/solicitacoes/:id/aprovar` | ação **INTERNA** — 403 quando há escopo de participante |
| `POST /v1/fiscal/portal/solicitacoes/:id/recusar` | ação **INTERNA** — 403 quando há escopo de participante |
| `GET /v1/fiscal/portal/cadastro/participante` | retorna só o participante em escopo |
| `GET /v1/fiscal/portal/cadastro/produtos` | retorna só os produtos vinculados ao escopo |

---

## 3. Evidências por endpoint (chamadas e retornos reais)

**Ambiente:** backend MSW em `http://localhost:3001`, `X-Mock-Papel: socio`. **A = `part-1`**, **B = `part-2`**.

### Fila isolada por escopo

```
GET /portal/solicitacoes  [escopo participante:part-1] → sol-1 (part-1), sol-3 (part-1)
GET /portal/solicitacoes  [escopo participante:part-2] → sol-2 (part-2), sol-4 (part-2)
GET /portal/solicitacoes  [sem escopo / interno]       → 4 (fila completa)
```

### Aprovar/recusar são internos (403 para o fornecedor)

```
POST /portal/solicitacoes/sol-1/aprovar  [escopo participante:part-1]
  → 403 "Apenas o analista interno pode aprovar."
```

### Fluxo completo: solicitar → aprovar → vira cadastro real

```
1) POST /portal/solicitacoes  [escopo participante:part-1]
   body: { tipo:'produto', operacao:'criacao', dados:{codigo:'TEC-NOVO', descricao:'Fio 40/1', ncm:'52051100', unidade:'KG', precoUnitario:33.5} }
   → 201 · status: "pendente" · participanteId: "part-1"   (participante veio do ESCOPO, não do body)

2) POST /portal/solicitacoes/<id>/aprovar  [interno]
   → 200 · status: "aprovada" · registroGeradoId: "prod-…" · decididoPor: "socio"

3) GET /portal/cadastro/produtos  [escopo participante:part-1]
   → 200 · contém o produto gerado (codigo "TEC-NOVO", descricao "Fio 40/1")

4) GET /portal/cadastro/produtos  [escopo participante:part-2]
   → 200 · NÃO contém o produto de A (total 0)
```

### Recusar (interno) grava o motivo

```
POST /portal/solicitacoes/sol-2/recusar  [interno]  body: { motivo:'Falta comprovante' }
  → 200 · status: "recusada" · motivoRecusa: "Falta comprovante"
```

### Cadastros read-only isolados

```
GET /portal/cadastro/participante [escopo participante:part-1] → Têxtil Cataratas (part-1)
GET /portal/cadastro/participante [escopo participante:part-2] → Suprimentos TF (part-2)
```

✅ O fornecedor só solicita e só vê o que é dele; aprovar/recusar são exclusivos do analista (403 no backend); a aprovação materializa o registro real e ele passa a aparecer no cadastro do fornecedor certo — e só dele.

---

## 4. Evidências de UI

| Sinal | Modo participante (escopo A) | Modo interno (sem escopo) |
|---|---|---|
| Formulário "Nova solicitação" | **presente** | ausente |
| Título da lista | "Minhas solicitações" | "Fila de aprovação" |
| Coluna Fornecedor | ausente | **presente** |
| Botões Aprovar / Recusar | **ausentes** | **presentes** |
| Linhas | 2 (só de A) | 4 (fila completa) |

`Meus Dados` renderiza o participante do escopo (somente leitura); `Meus Produtos` lista os vinculados (vazio na base; populado por uma solicitação aprovada, conforme o fluxo acima).

---

## 5. Documentação in-app

3 telas novas documentadas: `portal-fornecedor/solicitacoes`, `.../cadastros/meus-dados`, `.../cadastros/meus-produtos` — com o conceito "solicitação → aprovação → cadastro real" e as ações de cada modo.

---

## 6. Verificações

- `tsc --noEmit` limpo · `next lint` sem erros/warnings · console sem erros.
- Nenhuma entidade duplicada: a aprovação reusa Participante/Produto/Vínculo existentes.

## 7. Próximo lote

- **B4** — Configurações estruturadas + Logs + Tickets.
- **Portal do Cliente** — lote próprio (escopo por `empresa`), com checklist apresentado antes.

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`part-1`, `part-2`, `sol-N`, `prod-…`) são fixtures do ambiente de desenvolvimento.*
