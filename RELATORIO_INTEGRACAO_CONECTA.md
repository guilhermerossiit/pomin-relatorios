# RELATÓRIO — Integração Conecta (portabilidade fiel + estação de Escrituração)

> **Ciclo:** Gestor Fiscal › Monitores › Integração Conecta
> **Branch:** `feat/gestor-fiscal-conecta`
> **Referência:** tela de Escrituração Fiscal do Invoice-Management (`InvoiceDashboard` + `InvoiceTable` + `InvoiceFilters` + `AdvancedFilterModal` + `StatusBadge` + `IntegrationModal` + `EscrituracaoModal`)
> **Data:** 2026-07-17

## 1. Objetivo

Reimplementar a Integração Conecta como **portabilidade fiel** da tela de Escrituração Fiscal do Invoice, elevada aos tokens do design system do Pomin — herdando 100% da estrutura e das funcionalidades da referência, com dados reais por trás de cada elemento.

## 2. Ajuste de decisão (D2) aprovado pelo PO

O Conecta **não é read-only** nos estágios anteriores: é a **estação de trabalho da Escrituração**. Criação/importação da estrutura da nota permanece na Captura (deep-link); mas **complementar a nota** (CFOP, destinação, finalidade, conta contábil, código vinculado) e **aplicar/reverter regras tributárias** (Central de Regras Fiscais) que fazem a nota avançar na Escrituração são **funcionais**, com handler real, RBAC próprio e auditoria em toda mutação. As ações de Integração (envio/reenvio/manual) permanecem.

## 3. Checklist de fidelidade — cobertura

### A. Pipeline de 3 estágios (herói)
| Item | Status |
|---|---|
| A1 3 seções ligadas por setas (Check-in → Escrituração → Integração) | ✅ |
| A2 Check-in: 4 sub-etapas (Registro de Entrada, PO/Item, Info Financeira, Confirmação) c/ ícone e contador | ✅ |
| A3 Escrituração: 2 sub-etapas (Não Convertidos, Convertidos) | ✅ |
| A4 Integração: 3 sub-etapas (Aguardando Aprovação, Rejeitados, Integrados) | ✅ |
| A5 Badge de total por seção | ✅ |
| A6 Barra multi-segmento (verde/âmbar + vermelho na Integração) + rodapé "X concluídos · Y pendentes · Z rejeitados" | ✅ |

### B. Cabeçalho
| Item | Status |
|---|---|
| B1 Título "Notas Fiscais no Ciclo" + "Última atualização HH:MM:SS" | ✅ |
| B2 Botão Documentação (modal de ajuda) | ✅ |
| B3 Nova Nota | ✅ deep-link → Captura Manual (D2 ajustado) |
| B4 Importar Nota | ✅ deep-link → Captura XML (D2 ajustado) |

### C. Filtros
| Item | Status |
|---|---|
| C1 Busca (nº / cliente / CNPJ) | ✅ |
| C2 Filtro Avançado — árvore hierárquica Organização→Contrato→Empresa (contexto real) + Status + Número + Tipo + Participante | ✅ |
| C4 Pills de status com contador | ✅ |
| C5 Chips de filtros aplicados (removíveis) | ✅ |
| C6 Limpar Filtros | ✅ |
| C7 Exportar | ✅ |

### D. Tabela
| Item | Status |
|---|---|
| D1 Colunas Status / Documento-Ações / Cliente / Tipo / Data Emissão / Data Entrada + **ERP Destino** / Valor | ✅ (colunas da ref + ERP) |
| D2 Filtro por coluna (funil → input) | ✅ |
| D3 StatusBadge + 3 step-boxes com tooltip por documento (por-doc: Concluído/Rejeitado/Em Progresso/Pendente) | ✅ |
| D4 Número + chave de acesso + botão Copiar | ✅ |
| D5 Botões de linha (Escrituração / Integração) | ✅ (integração + escrituração operáveis) |
| D6 Tipo com tooltip descritivo | ✅ |
| D7 Data Entrada + responsável | ✅ |
| D8 Participante: avatar iniciais + nome + CNPJ | ✅ |
| D10 Estado vazio "Nenhuma nota fiscal encontrada…" | ✅ |
| D11 Rodapé de paginação "Mostrando X de Y" + Anterior/Próximo | ✅ |

### E. Modal de Integração
| Item | Status |
|---|---|
| E1 Abas Status / Log de Integração | ✅ |
| E2 Ícone grande de status + título + descrição | ✅ |
| E3 Mensagem de erro em destaque | ✅ |
| E4 Detalhes (Documento, Data Transmissão, Chave) | ✅ |
| E5 Ver/Baixar JSON | ✅ (Envio + Retorno do ERP — supera a ref, D8) |
| E6 Ações Fechar + Tentar Novamente/Marcar Manual | ✅ RBAC-gated |
| E7 Aba Log tabular (Data/Hora, Usuário, Status, Mensagem) | ✅ |

### F. Estação de Escrituração (workstation — D2 ajustado)
| Item | Status |
|---|---|
| Abas Dados da Nota / Itens e Impostos / Totais / Financeiro / Contabilidade / Log | ✅ |
| Itens com CFOP, Destinação, Finalidade, Conta Contábil, PO + impostos ICMS/IPI/PIS/COFINS/IBS-CBS | ✅ |
| Complementar item (edição inline dos campos fiscais) | ✅ funcional (PATCH + auditoria) |
| Importar Regra / Reverter Regra (Central de Regras Fiscais) | ✅ funcional |
| Confirmar Escrituração → avança pipeline + dispara auto-envio ao ERP | ✅ funcional |

## 4. Arquivos alterados

| Arquivo | Conteúdo |
|---|---|
| `types/fiscal.ts` | Sub-etapas do pipeline em `IntegracaoDocumento`; `ItemNotaFiscal`, `ParcelaFinanceira`, `LancamentoContabilNota`, `LogEscrituracao`, `RegraTributaria`, `TotalImpostoNota`, `ConectaDetalhe`; `LinhaConecta`/`ConectaSummary` estendidos. |
| `mocks/fixtures/fiscal.ts` | CNPJs; doc aguardando escrituração (000315); campos de pipeline nas integrações; catálogo de regras tributárias; itens/parcelas/lançamentos/logs. |
| `mocks/handlers-fiscal.ts` | `montarDetalheConecta` + totais; endpoints `/regras-tributarias`, `/:id/detalhe`, PATCH item, `/aplicar-regra`, `/reverter-regra`, `/escriturar` (auto-envio ERP); auditoria em toda mutação; summary com sub-etapas. |
| `mocks/fixtures/admin-concessoes.ts` | `concessoesEscrituracaoConecta` (ação `escriturar`, papéis Analista/Admin/Master). |
| `app/(app)/escrituracao-fiscal/monitores/conecta/page.tsx` | Reescrita completa: pipeline com sub-etapas, filtros (busca/avançado hierárquico/pills/chips), tabela fiel + step-boxes/tooltips, EscrituracaoModal (6 abas) e IntegracaoModal (Status/Log). |

`apps/api` intocado. Nenhuma entidade duplicada (itens/regras são satélites e projeção da Central de Regras Fiscais).

## 5. Verificação

- `npm run typecheck` — **limpo**; `npm run lint` — **limpo**.
- **Navegador (dev server fresco), percorrendo o checklist:** pipeline com as sub-etapas e barras multi-segmento; KPIs; filtros com contadores; tabela com todas as colunas da referência + step-boxes exibindo o estado por documento (Concluído/Rejeitado/Em Progresso/Pendente), chave+copiar, avatar+CNPJ, responsável, paginação, deep-links preservando o contexto (`?organizacao=`).
- **Modal de Escrituração:** renderiza as 6 abas (Dados da Nota, Itens e Impostos [2], Totais, Financeiro [1], Contabilidade, Log) com cabeçalho de contexto e rodapé "Confirmar Escrituração".
- **Motor end-to-end (API):** complementar item (destinação) → aplicar regra RT-001 (CFOP 1101, ICMS R$ 1.530, conta 1.01.03, conversão `em_andamento`) → **Confirmar Escrituração** → documento `escriturado`, conversão `concluído`, **auto-envio ao ERP** (`enviando`) e 3 logs de escrituração. Integração `reenviar`/`marcar-manual` revalidados.

## 6. Fora de escopo (mantido)

- Envio real a ERPs (simulado no handler MSW, padrão do repo).
- Geração automática de lançamentos contábeis ao escriturar um documento novo (a aba Contabilidade mostra os lançamentos existentes; documentos recém-escriturados exibem estado vazio até a contabilização) — candidato a backlog.
