# RELATÓRIO — Consolidação: IDs de Empresa (item 41) + G0 das Docs

**Data:** 2026-08-09 · **Branch:** `fix/unificacao-ids-empresa` · **Commits:** `6dcd3b9`, `c9e3879`
**Escopo:** execução de duas pendências acumuladas — a Etapa B do ciclo 1/3 (unificação de ids) e o G0 do ciclo 2/3 (binding das docs do BPM).

---

## PARTE 1 — Item 41: id único de empresa (Etapa B)

### O que mudou
As 3 grafias divergentes foram unificadas na **grafia do cadastro de Empresa** (fonte de verdade, já usada por 80% do sistema e por todas as telas):

| Antes (mundo fiscal) | Agora (id único) |
|---|---|
| `empresa-iguassu-resort` | `empresa-iguassu` |
| `empresa-metalurgica-pomin` | `empresa-metalurgica` |
| `empresa-cliente-auditto-1` | `empresa-auditto-cliente1` |

**63 ocorrências em 4 arquivos**, todos em `mocks/` — **zero telas afetadas**, como previsto no levantamento.

**Inventário final: 8 grafias → 4** (uma por empresa) + `empresa-manual` (id sintético do upload, tratado como D-ids-1 = manter e registrar).

### Mapa canônico DELETADO
- `EmpresaPortalCliente.empresaComplianceId` removido do tipo e da fixture;
- os 5 usos `empresaPortalPorId(id)?.empresaComplianceId ?? id` viraram o **id direto**;
- a regra *"todo lote que toca empresa usa o mapa"* **deixa de existir** — não há mais o que esquecer.

### ⚠️ Trava de tenant — o risco central do levantamento
O perigo era o **fallback silencioso** de `ORG_POR_CLIENTE` (`?? 'org-pomin'`): se os `clienteId` dos documentos migrassem sem as **chaves** do mapa, os documentos da Auditto cairiam no tenant errado **sem erro nenhum**.

Prova determinística (chaves × clientes realmente usados nos documentos):

```
ORG_POR_CLIENTE: {"empresa-iguassu":"org-pomin","empresa-varejo-sul":"org-pomin",
                  "empresa-metalurgica":"org-pomin","empresa-auditto-cliente1":"org-auditto"}
UF_POR_CLIENTE : {"empresa-iguassu":"PR","empresa-varejo-sul":"PR",
                  "empresa-metalurgica":"PR","empresa-auditto-cliente1":"SP"}

clienteId usados em documentos: empresa-iguassu, empresa-varejo-sul, empresa-metalurgica, empresa-auditto-cliente1

CAEM NO FALLBACK de organização: NENHUM
CAEM NO FALLBACK de UF        : NENHUM

TRAVA DE TENANT — auditto => org-auditto  (esperado org-auditto)  OK
TRAVA DE UF     — auditto => SP           (esperado SP)           OK
```

### Portais íntegros sem o mapa (por endpoint)
```
GET /portal-cliente/obrigacoes [empresa:metalurgica] → 2      (BC2: 2)
GET /portal-cliente/certidoes  [empresa:metalurgica] → 12     (BC2: 12)
GET /portal-cliente/obrigacoes [empresa:iguassu]     → 2      (BC2: 2)
GET /portal-cliente/obrigacoes/<de outra empresa> [empresa:metalurgica] → 403
```
Mesmos números dos lotes BC2/BC3 — o recorte não mudou, só a reconciliação sumiu.

`tsc` e `next lint` limpos. **Item 41 marcado como RESOLVIDO no BACKLOG.**

---

## PARTE 2 — G0: duas docs completas saíam invisíveis

### Causa raiz confirmada
O painel resolve a documentação por **rota**: `documentacaoDaTela(pathname)` em `components/docs/doc-button.tsx`. Duas definições ricas do ciclo BPM-DESIGN tinham `telaId` que **não correspondia à rota do menu** — o painel nunca as encontrava e abria "em construção":

| `telaId` antigo | Rota real do menu | Agora |
|---|---|---|
| `bpm-engine/fluxo-de-processo` | `/bpm-engine/processos` | `bpm-engine/processos` |
| `bpm-engine/central-de-processos` | `/bpm-engine/monitor` | `bpm-engine/monitor` |

### Grep preventivo (a pergunta que você fez)
Procurei a mesma classe de problema no resto do sistema:
- **Docs órfãs restantes: 2** — `launcher` e `preferencias`, que são rotas de plataforma **fora do menu de módulos**. **Não são bug**; existem e são alcançáveis.
- Nenhuma outra doc registrada deixou de casar com uma rota do registry.
- Ajustados também o marcador `data-tela-id` das duas páginas e o `processoTelaId` da simulação GPE, que apontavam para os ids antigos.

### Efeito na cobertura
BPM Engine: de **1 tela documentada · 10 sem doc** → **3 documentadas · 8 sem doc**.

> **Nota honesta de medição:** o inventário classifica essas 3 como "mínima" apenas porque meu parser estático não reconhece `conceitos: conceitosFluxo` (uma variável, não um literal). As docs têm 7 conceitos ricos — a classificação é falso-negativo do medidor, não das docs. Vou corrigir o parser no lote G1.

---

## 3. Situação das pendências

| Pendência | Estado |
|---|---|
| **Item 41** (ciclo 1/3) | ✅ **concluído** — branch `fix/unificacao-ids-empresa`, pronta para merge |
| **G0** (ciclo 2/3) | ✅ **concluído** — na mesma branch |
| **G1** (núcleo fiscal diário, ~12 telas) | ⏳ não iniciado — trabalho substancial (anatomia SVG por tela) |
| **Vendas por Dia — Etapa B** | ⏳ não iniciado — aguardava D-vd-1..4, que sigo assumindo pelas recomendações |
| **F1-B2** (telas da contabilização) | ⏳ próximo do ciclo ativo |

**Nota de branches:** o item 41 e o G0 saíram da `main`; o F1-B1 (contabilização) está na `feat/contabilizacao-efetiva`, também a partir da `main`. As duas tocam `handlers-fiscal.ts` em pontos diferentes — recomendo **mergear o item 41 primeiro** e depois integrar a branch da contabilização, para que ela já nasça sobre os ids unificados.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
