# RELATÓRIO — Documentação in-app · ETAPA A (mapa de cobertura)

**Data:** 2026-08-09
**Branch:** `feat/docs-cobertura` (a partir da `main`)
**Molde:** piloto do Conecta (`DESIGN_SYSTEM.md §4g`) — 7 seções: *o que é · anatomia com ilustração · ações com passo a passo · conceitos · simulação GPE · FAQ · telas relacionadas* + sinônimos.
**Natureza:** levantamento. **Nenhuma doc foi escrita.** Aguardando aprovação da ordem de produção.

---

## 1. Cobertura hoje

Cruzamento entre **as 202 telas do registry de módulos** e **as 65 definições do registry de documentação**:

| Nível | Telas | % | Critério |
|---|---:|---:|---|
| **PLENA** | **3** | 1,5% | molde completo, **com anatomia ilustrada** |
| **BOA** (incompleta) | **33** | 16,3% | núcleo pronto (o que é + ações + conceitos + FAQ + relacionadas); **falta anatomia** e/ou simulação |
| **MÍNIMA** (placeholder) | **29** | 14,4% | declara propósito e status, **sem listar ação** (fábrica `docPlaceholder`) |
| **SEM DOC** | **137** | 67,8% | não registrada — o painel abre "em construção" |

**Com alguma documentação: 65 de 202 (32%). No molde pleno: 3 (1,5%).**

As 3 plenas são o piloto e seus irmãos diretos: **Integração Conecta**, **Pedido de Compra**, **Captura XML** — todas em Gestor Fiscal › Monitores.

> **Nota de método:** "BOA" e "MÍNIMA" são subdivisões do que o PO chamou de *doc mínima*. Separei porque o custo de conclusão é muito diferente: **BOA** precisa só da ilustração de anatomia (e simulação quando há fluxo); **MÍNIMA/SEM DOC** exigem o conteúdo inteiro.

---

## 2. Mapa por módulo

| Módulo | Telas | Plena | Boa | Mínima | Sem doc |
|---|---:|---:|---:|---:|---:|
| Gestor Fiscal | 41 | **3** | 19 | 17 | 2 |
| Portal do Fornecedor | 12 | 0 | 7 | 5 | 0 |
| Portal do Cliente | 7 | 0 | 7 | 0 | 0 |
| Controle & Compliance | 14 | 0 | 0 | 6 | 8 |
| BPM Engine | 11 | 0 | 0 | 1 | 10 |
| Dashboards | 3 | 0 | 0 | 0 | **3** |
| Contabilidade | 16 | 0 | 0 | 0 | **16** |
| Folha de Pagamento | 18 | 0 | 0 | 0 | **18** |
| CRM Contábil | 19 | 0 | 0 | 0 | **19** |
| Administração | 14 | 0 | 0 | 0 | **14** |
| Ativo Imobilizado | 13 | 0 | 0 | 0 | **13** |
| Núcleo de IA | 11 | 0 | 0 | 0 | **11** |
| Robot Engine | 10 | 0 | 0 | 0 | **10** |
| Integrações | 8 | 0 | 0 | 0 | **8** |
| Central de Alertas | 5 | 0 | 0 | 0 | **5** |

A documentação seguiu os ciclos de produto: onde houve ciclo (Gestor Fiscal, Portais, CND/DTE/Tarefas, Apurações), há doc; os 9 módulos que não passaram por ciclo estão zerados.

---

## 3. 🐛 Achado — duas docs completas que **nunca aparecem** (bug de binding)

O `DocPanel` resolve a doc por `telaId === rota sem a barra`. Duas definições ricas do ciclo BPM-DESIGN foram registradas com um `telaId` que **não corresponde à rota real do menu**:

| Doc registrada (`telaId`) | Verbetes | Rota real no menu | Efeito hoje |
|---|---:|---|---|
| `bpm-engine/fluxo-de-processo` | 5 | `/bpm-engine/processos` | painel abre **"em construção"** |
| `bpm-engine/central-de-processos` | 4 | `/bpm-engine/monitor` | painel abre **"em construção"** |

**Conteúdo pronto, invisível ao usuário.** Correção: alinhar o `telaId` à rota (ou a rota ao `telaId`) — minutos de trabalho, e o BPM salta de "1 mínima" para "2 boas + 1 mínima". Proponho fazer isso como **G0**, antes de qualquer produção nova.

*(As outras duas docs fora do menu — `launcher` e `preferencias` — são rotas legítimas de plataforma, fora do menu de módulos. Não são bug.)*

---

## 4. Ordem de produção proposta — por VALOR DE USO

Critério: **frequência de uso × custo de não ter doc**. Telas de rotina diária/mensal do escritório primeiro; módulos especializados e de persona restrita por último. "Completar" (BOA → PLENA, só anatomia/simulação) é muito mais barato que "criar" (do zero).

| # | Grupo | Telas | Trabalho | Esforço | Por quê nesta posição |
|---|---|---:|---|:--:|---|
| **G0** | **Correção de binding do BPM** | 2 | corrigir `telaId` | **P** | Conteúdo já existe e está invisível. Melhor relação valor/custo do ciclo |
| **G1** | **Núcleo fiscal diário** — Acompanhamento (6), Recebidos/Emitidos, Manifesto CT-e, Manifestação | ~12 | completar (anatomia + simulação) | **M** | A operação diária do analista. Núcleo já escrito |
| **G2** | **Controle & Compliance** — Obrigações + Monitor, Certidões, DTE, Tarefas, Painel, Histórico, Config | 14 | 6 completar · 8 criar | **G** | **Obrigações é a rotina mensal do escritório** e está SEM DOC |
| **G3** | **Apurações + SPED** (Gestor Fiscal restante) | ~19 | completar/criar | **G** | Fechamento mensal; alto valor, vocabulário difícil |
| **G4** | **Dashboards (3) + Central de Alertas (5)** | 8 | criar | **M** | Vistos todo dia por todos os papéis; telas simples de documentar |
| **G5** | **Portais** — Fornecedor (12) + Cliente (7) | 19 | completar (anatomia) | **M** | Únicas telas com **usuário externo**; núcleo pronto e recente |
| **G6** | **Contabilidade** | 16 | criar | **G** | Rotina mensal pesada |
| **G7** | **Folha de Pagamento** | 18 | criar | **G** | Rotina mensal com prazos legais |
| **G8** | **Administração (14) + Integrações (8)** | 22 | criar | **G** | Configuração/onboarding — uso concentrado no admin |
| **G9** | **BPM (10) + Robot (10) + Núcleo de IA (11)** | 31 | criar | **G** | Automação — usuários avançados |
| **G10** | **CRM (19) + Ativo Imobilizado (13)** | 32 | criar | **G** | Persona comercial / rotina periódica |

**Se o objetivo for impacto rápido:** G0 → G1 → G2 → G4 cobre a operação diária e mensal do escritório com esforço majoritariamente de *conclusão*, não de criação — leva a cobertura plena de 3 para ~37 telas mexendo pouco em conteúdo novo.

### Ordem alternativa (se preferir "zerar os módulos vazios" primeiro)
G0 → G4 (Dashboards/Alertas) → G2 → G6/G7 (Contabilidade/Folha) → demais. Fecha buracos visíveis por módulo mais cedo, ao custo de deixar o núcleo fiscal incompleto por mais tempo.

---

## 5. Observações para a Etapa B

- **Simulação GPE só onde há fluxo real** — o molde pede simulação "quando houver fluxo". Telas de consulta/listagem simples não ganham simulação artificial (seria ruído).
- **Anatomia é componente SVG com tokens**, nunca screenshot (`DESIGN_SYSTEM.md §4g`) — acompanha os 6 temas e não envelhece a cada ajuste de layout.
- **Conceitos transversais** (`lib/docs/conceitos-comuns.ts`) devem ser reusados em vez de reescritos por tela.
- **Sinônimos declarados por verbete** alimentam a busca — parte obrigatória, não enfeite.
- **Placeholders** (29 telas) só devem virar doc plena **se a tela tiver funcionalidade real**; para as que continuam placeholder de produto, a doc mínima é a resposta honesta (o próprio molde prevê isso).

---

## PARADA

Aguardo sua aprovação de: **(1)** a ordem de produção (G0→G10 acima ou a alternativa), **(2)** o tamanho do lote por vez, e **(3)** se G0 (correção do binding BPM) pode entrar já junto com o primeiro grupo aprovado.

*Relatório de análise. Não contém credenciais, tokens nem dados de clientes reais.*
