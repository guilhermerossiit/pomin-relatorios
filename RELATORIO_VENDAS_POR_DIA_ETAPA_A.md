# RELATÓRIO — Vendas por Dia · ETAPA A (proposta de conteúdo)

**Data:** 2026-08-09
**Branch:** `feat/vendas-por-dia` (a partir da `main`)
**Contexto:** último placeholder do menu **Gestor Fiscal › Monitores**. O menu previu a tela; o conteúdo nunca foi especificado.
**Natureza:** proposta. **Nenhum código de tela foi escrito.** Aguardando seu aval do conteúdo.

---

## 1. Referência de design — não existe

Procurei nas duas fontes possíveis:

- **Invoice-Management / `design-reference/`** — *nenhuma tela de vendas/faturamento*. Os módulos portados (CndControl, ControleEntregas, TaxCalculation, Admin/Auditor) não têm equivalente. Busca por "venda/sales/faturamento" no `design-reference/`: **zero ocorrências**.
- **`PROPOSTA_GESTOR_FISCAL.md`** (fonte do menu) — cita a tela apenas na lista de placeholders, **declarando explicitamente que não correspondia a nenhuma entidade existente** e pedindo esclarecimento antes de desenhar.

A única definição escrita é a do próprio placeholder em tela:
> *"Painel de vendas diárias por empresa — volume e valor emitido no dia, cruzado com os documentos de saída escriturados."*

**Portanto a proposta abaixo é original**, derivada dos dados que o sistema já produz — não é port de referência visual. Sigo o padrão visual das telas irmãs de Monitores (Conecta, Pedido de Compra, Captura XML).

---

## 2. Matéria-prima disponível (o que já existe, sem inventar)

A tela se alimenta de **Documentos Emitidos** — a entidade criada no ciclo homônimo:

| Fonte | O que oferece |
|---|---|
| `LinhaDocumentoEmitido` | `dataEmissao`, `valorTotal`, `tipo` (NF-e/NFS-e), `destinatarioNome/Cnpj`, `statusAutorizacao`, `rejeicao`, `pedidoVenda`, `qtdFontes` |
| `DocumentoFiscal` (vinculado) | `clienteId` (empresa emitente), `naturezaOperacao`, `competencia`, `status` de escrituração |
| `valoresDocumento(doc, itens)` | tributos destacados por documento: **IBS, CBS, PIS, COFINS, ISS, IRRF, CSLL, INSS** (derivados da regra aplicada) |
| `GET /v1/fiscal/emitidos/dashboard` | já agrega **volume por dia (30 dias)** — mas **só contagem, não valor** |

**A lacuna que a tela preenche:** hoje o sistema sabe *quantos* documentos saíram por dia; **não sabe quanto foi faturado por dia**, nem a quebra por empresa/tipo, nem a comparação entre períodos.

### Massa de dados atual — atenção
São **15 documentos emitidos** (e01–e15), distribuídos de 1 a ~23 dias atrás, em **3 empresas**, valores de R$ 2.400 a R$ 26.000, com status variados (autorizada, rejeitada, cancelada, contingência) e uma devolução. **É real, mas esparso** para um gráfico diário de 30/90 dias — vários dias ficam zerados. Ver decisão **D-vd-3**.

---

## 3. Proposta da tela

**Rota:** `/gestor-fiscal/monitores/vendas-por-dia` (já existe, hoje placeholder).
**Escopo:** multi-tenant por organização + filtro por empresa — o mesmo guard das telas fiscais.

### 3.1 Barra de filtros + chips (padrão obrigatório)
Período (padrão: **últimos 30 dias**), Empresa, Tipo de documento (NF-e/NFS-e), Status. Todos viram **chips removíveis** no painel de filtros aplicados, com contador `X de Y` — incluindo os de **default não-neutro** (o período de 30 dias vira chip, pela regra do `CLAUDE.md`: recorte que o usuário não vê é recorte que ele não sabe desfazer).

### 3.2 KPIs (4 cartões)
| KPI | Conteúdo |
|---|---|
| **Faturado no período** | soma do `valorTotal` das vendas válidas + variação % vs. período anterior |
| **Documentos emitidos** | contagem + ticket médio |
| **Melhor dia** | data + valor do pico |
| **Tributos destacados** | soma dos tributos das vendas do período (composição no tooltip) |

### 3.3 Gráfico de vendas diárias (peça central)
Barras por dia (valor faturado) + **linha de média móvel de 7 dias**, no período filtrado. Todos os dias do intervalo aparecem, **inclusive os zerados** (mesma regra já usada no dashboard de emitidos — dia sem venda é informação, não ausência). Clicar num dia **filtra a listagem** abaixo.

### 3.4 Quebras (dois painéis lado a lado)
- **Por empresa** — barras horizontais: valor, nº de documentos e % do total.
- **Por tipo de documento** — NF-e × NFS-e (produto × serviço): valor e participação.

### 3.5 Comparativo de período
Faixa comparando o período filtrado com o **período anterior de mesmo tamanho**: faturamento, documentos e ticket médio, com variação (▲/▼ %). Sem projeção/estimativa — só o que já aconteceu.

### 3.6 Listagem drill-down
Tabela dos documentos que compõem o número, com: data, número/série, empresa, destinatário, tipo, status, **valor**, pedido de venda vinculado. Ações: abrir o detalhe do documento (reuso do drawer de Emitidos) e **Exportar .xlsx** (padrão das telas irmãs). Respeita o dia clicado no gráfico e os chips ativos.

### 3.7 Documentação in-app (mesmo ciclo)
Doc nova no molde: *o que é · anatomia · ações · conceitos* (o que conta como venda, dia zerado, média móvel, tributos destacados) *· FAQ · relacionadas*, com sinônimos (`faturamento`, `vendas diárias`, `saídas`, `emissão`).

---

## 4. Decisões que preciso de você

**D-vd-1 — O que conta como "venda"?** *(recomendação: **a**)*
- **(a)** Só documentos **autorizados** (exclui rejeitada, cancelada e contingência não confirmada) — é o faturamento efetivo.
- **(b)** Todos os emitidos, com quebra por status no gráfico.
> Impacto: com a massa atual, (a) reduz de 15 para ~8 documentos no gráfico.

**D-vd-2 — Devolução de venda abate o faturamento?** *(recomendação: **a**)*
- **(a)** Sim, entra como **valor negativo** (visão de faturamento líquido) — há uma devolução na massa (e08).
- **(b)** Não, fica fora do gráfico e aparece só como indicador separado.

**D-vd-3 — Massa de dados** *(recomendação: **b**)*
- **(a)** Usar os 15 emitidos atuais — honesto, mas gráfico esparso (muitos dias zerados).
- **(b)** **Enriquecer as fixtures de emitidos** no lote B (ex.: ~60–80 documentos ao longo de 90 dias, nas 4 empresas, com sazonalidade leve) — dado de desenvolvimento, mantém tudo derivado do mesmo gerador, e faz a tela mostrar o que ela realmente é.

**D-vd-4 — Tributos destacados no KPI** *(recomendação: **a**)*
- **(a)** Somar os tributos via `valoresDocumento` (derivado da regra aplicada — coerente com Apurações).
- **(b)** Deixar tributos fora desta tela (mantê-la puramente comercial) e remeter às Apurações.

---

## 5. Escopo do lote B (após aprovação)

1. Endpoint `GET /v1/fiscal/vendas-por-dia` — série diária + KPIs + quebras + comparativo, sob o guard multi-tenant, com filtros de período/empresa/tipo/status (nada calculado no cliente).
2. (Se D-vd-3=b) enriquecimento do gerador de emitidos nas fixtures.
3. Tela com filtros + chips, 4 KPIs, gráfico com média móvel, 2 quebras, comparativo e listagem drill-down + Exportar.
4. Doc in-app no molde + registro no registry.
5. Validação: números na ponta do lápis (soma da série = KPI = soma da listagem), 6 temas, typecheck/lint, relatório com evidências.

---

## PARADA

Aguardo seu aval do **conteúdo da tela** (§3) e das decisões **D-vd-1 a D-vd-4**.

*Relatório de análise. Não contém credenciais, tokens nem dados de clientes reais — os identificadores citados são fixtures do ambiente de desenvolvimento.*
