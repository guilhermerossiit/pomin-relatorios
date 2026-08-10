# RELATÓRIO — Fase 3 · Etapa A (Apurações Modernizadas)

**Branch:** `feat/apuracoes-modernas` · **Nenhuma linha de código de produção alterada.**

---

## 1. Claude Design — verificado, e NÃO existe tela de Apuração

Você pediu para conferir antes de propor. Fiz a verificação **renderizando**, não pela memória.

`Artifact action:list scope:all` → 4 artifacts. Três são relatórios (Relatório de Atualizações, Revisão Completa, Relatório de Dados Financeiros do Check-in). O único protótipo é **"Controle de Entrega"**.

Como o artifact é um bundle auto-descompactável (gzip+base64 — `grep` não enxerga nada), servi o HTML numa porta local e abri no navegador. Menu efetivamente renderizado:

```
Início · CND · Domicílio Eletrônico · Formulário Dinâmico · Administração
```

Conteúdo do Início: *Painel de Recebimento* — Check-in, Escrituração, Integração. Busca por `apura|tribut|imposto|guia|ICMS|ISS|PIS|COFINS|IPI|IRPJ|CSLL` no texto renderizado: **nenhuma ocorrência**.

**Conclusão: não há especificação visual de Apuração no Claude Design.** A proposta abaixo é pelos padrões da casa.

---

## 2. Levantamento do que existe hoje

| Fato | Situação |
|---|---|
| Telas de apuração | **11** — IBS, CBS, ICMS, IPI, PIS/COFINS, ISS, Retenções Federais, IRPJ, CSLL + 2 de configuração |
| Porta de entrada | **não existe** — o menu lista 11 itens irmãos; para saber como está a competência, abre-se uma por uma |
| Motor de cálculo | `lib/fiscal/motor-apuracao.ts` · `derivarApuracao(apuracao, itens, saldoAnterior, ajustes)` |
| Decomposição já disponível | `base · debitos · creditos · saldoCredorAnterior · ajustesDebito · ajustesCredito · saldoDevedor · saldoCredorTransportado` |
| Endpoints | `GET /apuracoes` (todas, enriquecidas) · `GET /apuracoes/:id` · fechar · reabrir · transmitir · ajustes · guia |

**A boa notícia para a regra inegociável:** os 5 números do resumo executivo que você descreveu **já são exatamente os campos que o motor devolve**. Modernizar não exige tocar em fórmula nenhuma — é montar tela sobre o que já é calculado.

---

## 3. ACHADO — o checkpoint que você definiu já falha hoje

Antes de qualquer tela nova, testei o critério "números do Painel = telas de tributo = resumo do Portal do Cliente":

```
GET /v1/fiscal/apuracoes                                  (tela do tributo)
GET /v1/fiscal/portal-cliente/apuracoes?empresaId=…       (Portal do Cliente)

apuracao-icms-0726 · ICMS · 2026-07
   Portal do Cliente : R$    0,00
   Motor (tela)      : R$  961,29     ← DIVERGENTE
```

**Causa:** o endpoint do Portal lê `apuracoes` **cru** e devolve `a.saldoDevedor` — o campo materializado no fixture, que nasce zerado. As telas internas passam por `enriquecerApuracao` → `derivarApuracao`, que recalcula a partir dos itens reais. **São duas fontes para o mesmo número**, e o cliente está vendo a errada.

Não é hipótese: é o mesmo cliente, a mesma competência, dois números diferentes hoje, em produção.

**Correção proposta — e ela NÃO viola a regra inegociável:** o Portal passa a chamar `enriquecerApuracao`, a mesma função que as telas já usam. Nenhuma fórmula muda; some uma leitura de campo materializado. É a terceira vez neste projeto que a fonte dupla aparece (prévia × efetivo no F1-B1, conta por código sem escopo no item 43) — e a única razão de aparecer aqui antes do relatório final é que o seu checkpoint obriga a comparar.

---

## 4. Proposta

### 4.1 Painel Central de Apurações — `/gestor-fiscal/apuracoes`

Porta de entrada. Hoje essa rota não existe; o menu vai direto aos tributos.

- **Seletor de empresa + competência** no topo (competência atual por default, virando chip no painel de filtros aplicados — default não-neutro é recorte que o usuário precisa enxergar).
- **Card por tributo**: valor apurado, situação (**a recolher** / **nada a recolher** / **credor a transportar**), status (aberta · fechada · transmitida) e vencimento da guia.
- **Frase direta em cada card**, como você pediu: *"ICMS de junho: R$ 4.200 a recolher — vence 10/07"*. Sem rótulo técnico onde cabe uma frase.
- **Total do mês** consolidado, separando o que é a recolher do que é saldo credor (somar as duas coisas num número só seria mentira aritmética).
- **Alertas**: competência com escrituração feita e apuração ainda aberta · guia vencendo em ≤ 5 dias · guia vencida · tributo sem apuração na competência.
- Card clica → tela do tributo, já na competência selecionada.

### 4.2 Tela do tributo em camadas

Reembalagem das 9 telas existentes, **sem tocar no cálculo**:

1. **Resumo executivo** — os 5 números, grandes, na ordem da conta: `débitos − créditos − saldo anterior ± ajustes = saldo`. Cada um **clicável**.
2. **Detalhamento colapsado** — o que hoje aparece de cara (composição, informativos, ajustes) passa a abrir sob demanda.
3. **Drill-down**: clicar em *débitos* leva à lista dos documentos que compõem aquele número, com o valor de cada um somando exatamente o total exibido.

### 4.3 Comparativo de competências

Evolução mês a mês por tributo — tabela + gráfico de barras, com variação percentual. Serve para a pergunta que o contador faz sozinho hoje: *"por que este mês está tão diferente do anterior?"*.

---

## 5. Decisões que preciso de você (trava desta fase)

| # | Decisão | Minha recomendação |
|---|---|---|
| **D-ap-1** | Corrigir a fonte dupla do Portal do Cliente (§3) dentro desta fase? | **Sim, no primeiro lote.** É pré-requisito do seu próprio checkpoint — sem isso ele não passa. Não muda fórmula. |
| **D-ap-2** | O Painel substitui os 11 itens do menu por um item só ("Apurações"), com os tributos alcançáveis por dentro? | **Sim** — 11 irmãos no menu é o sintoma que a fase quer curar. Mantendo as rotas atuais funcionando (deep-links e docs existentes não podem quebrar). |
| **D-ap-3** | O drill-down abre em modal ou navega para uma tela de documentos filtrada? | **Modal** — o contador está conferindo um número, não mudando de tarefa; tirá-lo da tela perde o contexto do que ele estava somando. |
| **D-ap-4** | O comparativo entra no primeiro lote ou depois do Painel validado? | **Depois.** O Painel e o drill-down entregam o valor central; o comparativo é enriquecimento e pode esperar sua validação dos dois primeiros. |
| **D-ap-5** | Reconciliação apuração × Diário contábil (o que levantei no fim da Fase 2) entra aqui? | **Não nesta fase.** É comparação de mecânica entre dois motores — merece trava própria. Registro no backlog para você priorizar. |

---

## 6. Lotes propostos (após sua validação)

| Lote | Escopo |
|---|---|
| **F3-B1** | Correção da fonte dupla do Portal (D-ap-1) + endpoint do Painel (agregado por competência, sem cálculo novo) + **checkpoint de igualdade provado por endpoint** |
| **F3-B2** | Painel Central de Apurações (cards, total, alertas, filtros/chips, 6 temas) + reorganização do menu (D-ap-2) + doc in-app |
| **F3-B3** | Tela do tributo em camadas + drill-down até os documentos + doc atualizada |
| **F3-B4** | Comparativo de competências + fechamento da fase |

---

## 7. Confirmação da regra inegociável

Nenhum item desta proposta altera fórmula de apuração. O motor `derivarApuracao` e os `CONFIG_TRIBUTOS` permanecem intactos; a fase consome o que eles já calculam. A única mudança de comportamento proposta é o Portal **parar de ler um campo materializado desatualizado** e passar a ler o mesmo derivado que as telas leem — que é o oposto de mexer no cálculo: é fazer os dois lados usarem o cálculo que já existe.

*Documento de especificação técnica. Não contém credenciais nem dados de clientes reais.*
