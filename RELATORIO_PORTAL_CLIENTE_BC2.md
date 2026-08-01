# RELATÓRIO — Portal do Cliente · Lote BC2

**Data:** 2026-07-31
**Branch:** `feat/portal-cliente`
**Commit:** `ddf0dc1`
**Escopo do lote:** Obrigações + Certidões (CND) na ótica do cliente — read-only, sob escopo por empresa, linguagem para o cliente leigo.

---

## 1. O que foi entregue

Duas telas do cliente, **read-only**, filtradas pela empresa em escopo via o **mapa canônico** (`clienteId → empresaComplianceId`, regra do item 41 — todo lote que toca empresa usa o mapa):

- **Minhas Obrigações** — o que o escritório entrega ao governo pela empresa, com prazo e situação; **baixar** o comprovante das entregues.
- **Minhas Certidões** — as CNDs com validade e situação; **baixar** o PDF das válidas.

**Ações do modo cliente:** LER (status/prazo/competência/validade) e BAIXAR (comprovante/PDF). **Nunca** editar, dar baixa ou renovar — essas permanecem internas ao escritório.

**Linguagem para o cliente leigo:** situações traduzidas ("A entregar", "Entregue", "Em atraso", "Válida", "Vence em breve", "Vencida — em renovação"), sem jargão de obrigação acessória.

---

## 2. Segurança — endpoints novos sob o guard universal

| Endpoint | Regra |
|---|---|
| `GET /portal-cliente/obrigacoes` (lista) | filtra pela empresa em escopo (via mapa); participante/sem-escopo → 400 |
| `GET /portal-cliente/obrigacoes/:id` | 403 se a obrigação é de outra empresa |
| `GET /portal-cliente/certidoes` (lista) | idem |
| `GET /portal-cliente/certidoes/:id` | 403 se a certidão é de outra empresa |

---

## 3. Evidências por endpoint (chamadas e retornos reais)

**Ambiente:** MSW em `http://localhost:3001`, `X-Mock-Papel: socio`. **Empresa A = `empresa-metalurgica-pomin`**, **B = `empresa-iguassu-resort`**.

```
Obrigações
  GET /portal-cliente/obrigacoes  [escopo empresa:A] → 200 · 2 · ids [obrigacao-das-metalurgica, obrigacao-dirf-metalurgica]
  GET /portal-cliente/obrigacoes  [escopo empresa:B] → 200 · 2 · id0 obrigacao-iss-iguassu
  GET /portal-cliente/obrigacoes/obrigacao-iss-iguassu  (de B)  [escopo empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
  GET /portal-cliente/obrigacoes/<de A>                 (própria)[escopo empresa:A] → 200 (podeBaixar: true)
  GET /portal-cliente/obrigacoes  [escopo participante:part-2]   → 400  (empresa não vaza para participante)

Certidões
  GET /portal-cliente/certidoes  [escopo empresa:A] → 200 · 12
  GET /portal-cliente/certidoes  [escopo empresa:B] → 200 · 12
  GET /portal-cliente/certidoes/<de B>  [escopo empresa:A] → 403 "Acesso negado ao recurso de outra empresa."
  GET /portal-cliente/certidoes/<válida de A>  [escopo empresa:A] → 200 · arquivo "certidao_certidao-cnd-24.pdf"
  GET /portal-cliente/certidoes  [escopo participante:part-2]   → 400
```

✅ **Recorte** por empresa; **403** ao acessar recurso de outra empresa; **empresa não vaza para participante** (chave errada → 400). Baixar o próprio documento → 200 com o nome do arquivo.

---

## 4. Evidência de UI

Modo Cliente + empresa Metalúrgica Pomin, tela **Minhas Obrigações**: título client-friendly, 2 linhas (só da empresa A), botão **Baixar** nas entregues e linguagem de leigo ("entrega ao governo por você", "A entregar/Entregue/Em atraso"). Certidões no mesmo padrão. `tsc`/`next lint`/console limpos.

---

## 5. Próximos lotes

- **BC3** — Apurações (resumo read-only: apurado por tributo/competência + baixar guia) + Meus Documentos (recebidos/emitidos) sob escopo.
- **BC4** — Comunicação + Tickets sob escopo; fechamento do Portal do Cliente.

*Relatório de evidências técnicas. Não contém credenciais, tokens nem dados de clientes reais — os identificadores (`empresa-*`, `obrigacao-*`, `certidao-*`, `part-N`) são fixtures do ambiente de desenvolvimento.*
