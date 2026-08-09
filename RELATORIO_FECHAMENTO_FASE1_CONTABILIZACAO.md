# RELATÓRIO DE FECHAMENTO — FASE 1 · CONTABILIZAÇÃO EFETIVA (item #22)

**Data:** 2026-08-09 · **Branch:** `feat/contabilizacao-efetiva`
**Lote final:** F1-B3 · **PARE** para validação antes de merge + tag + deploy.

---

## 1. O que a Fase 1 entregou

A escrituração fiscal confirmada **gera o lançamento contábil real** no Diário, vinculado ao documento nos dois sentidos. O que antes era uma prévia que morria na tela virou registro contábil com estorno, imutabilidade, permissão e pendência visível.

| Lote | Entrega |
|---|---|
| **Etapa A** | `SPEC_CONTABILIZACAO.md` + decisões D-1..D-6. Achado que reduziu o escopo: `LancamentoContabil`, `PartidaContabil`, validação de partida dobrada, `/reverse` e a tela do Diário **já existiam** — a Fase 1 virou "ligar dois motores", não "construir a contabilidade" |
| **F1-B1** | Motor da ponte: `lib/fiscal/motor-contabil.ts` como **fonte única** de montagem de partidas · store compartilhado do Diário · geração no `escriturar` · estorno em cadeia · imutabilidade (422) · RBAC no estorno (403) |
| **F1-B2** | Navegação bidirecional: aba Contabilidade no Conecta com o lançamento efetivo e deep-link ao Diário; coluna **Origem** no Diário com deep-link de volta à nota |
| **F1-B3** | Pendência contábil **derivada** na inbox · aviso no fechamento de competência · filtro/chip de origem · docs in-app com conceitos de contador · correção do parser de cobertura |

---

## 2. F1-B3 em detalhe

### 2.1 Pendência contábil na inbox — derivada, sem entidade nova

Novo tipo `sem_contabilizacao` em `derivarPendencias()`. Um documento **escriturado sem lançamento ativo** vira pendência calculada do estado — não há registro guardado, e ninguém "fecha" a pendência: ela some quando a causa deixa de existir.

O motivo exibido vem da **mesma** `montarPartidasDocumento` que monta o lançamento efetivo. O operador lê exatamente o que impediu a contabilização, nunca uma segunda explicação escrita à parte que pode divergir da real.

**Severidade proposta (para sua validação):**

| Situação da competência | Severidade | Por quê |
|---|---|---|
| Aberta / sem período cadastrado | **média** | Falta o lançamento, mas há tempo de resolver |
| **Em fechamento** | **alta** | Resolver antes de assinar — depois vira reabertura |
| **Já fechada** | **alta** | O resultado do período **está** incompleto; o motivo diz isso com essas palavras |

**Estado real medido:** 31 pendências `sem_contabilizacao`, 2 delas em **alta** (competência 2026-06 fechada).

### 2.2 Aviso no fechamento de competência (RN-FEC-04) — "mínimo: avisar"

Você pediu no mínimo um aviso. A proposta implementada vai um passo além, seguindo a regra da casa de ação quase-irreversível: **avisa, exige ciência explícita e audita.**

```
POST /v1/contabilidade/periodos/periodo-2026-07/fechar   {}
  → 409 "4 documento(s) escriturado(s) desta competência ainda não geraram lançamento contábil.
         Resolva as pendências ou confirme o fechamento com ciência."
     pendenciasContabeis: [ {doc-fiscal-1, 000123, motivo…}, {doc-m24n1, 100118, …}, {doc-s01, 100127, …}, … ]

POST /v1/contabilidade/periodos/periodo-2026-07/fechar   { "cienciaPendenciasContabeis": true }
  → 200 · status "fechado" · pendenciasContabeisNoFechamento: 4

Auditoria (criticidade alta):
  "fechamento.assinado (2026-07) COM CIÊNCIA de 4 documento(s) sem contabilização"
```

Não bloqueia — a decisão continua sendo do contador. Mas deixa de ser silenciosa e passa a ter dono.

**Como os dois lados conversam sem acoplar:** `handlers-contabilidade` não pode importar `handlers-fiscal` (ciclo). O fiscal, que detém o conhecimento, **se registra** como provedor em `mocks/state/contabilidade-state.ts`; o contábil pergunta. Inversão de dependência, não gambiarra de import.

### 2.3 Deep-links

Toda pendência traz **"Abrir a nota no Conecta"** (`?documento=<id>`). As que dependem de regra — `sem_regra` e `sem_contabilizacao` — trazem também **"Criar a regra que falta"**, que abre a Central de Regras (Produto ou Serviço, conforme o documento) **já com o formulário de nova regra aberto** (`?nova=1`). Ler o problema e resolvê-lo não podem estar em telas que o operador tem de procurar.

### 2.4 Filtro/chip de origem no Diário

Filtro **no backend** (`?origem=fiscal|estorno|manual`), não recorte de tela — o contador precisa auditar "o que veio do fiscal nesta competência" sem depender de a página inteira estar carregada.

A tela ganhou o **painel de filtros aplicados** padrão (§4f), que não tinha. A **empresa** entra como chip mesmo tendo default não-neutro: o Diário sempre mostra uma empresa, e um recorte invisível é um recorte que o contador não sabe que existe.

```
Empresa Iguassu Resort:  todas 4  =  fiscal 1 + estorno 0 + manual 3   ✓ soma bate
```

### 2.5 Documentação in-app

**Nova** — `contabilidade/lancamentos` (Diário), que não tinha doc. Os quatro conceitos que você pediu, escritos para contador:

- **Partidas dobradas** — inclui o caso da retenção: debita a despesa pelo BRUTO, credita o fornecedor pelo LÍQUIDO, mais uma conta por tributo retido; *"a diferença não some, muda de lugar"*.
- **Lançamento efetivado é imutável** — a razão é contábil, não de software: o Diário é registro histórico, e valor que muda depois destrói a rastreabilidade de tudo que se apoiou nele. Corrige-se por estorno; rascunho ainda não é registro e pode ser editado.
- **Competência contábil × fiscal** — quase sempre o mesmo mês; separam-se quando o período contábil está fechado.
- **Origem do lançamento** — Fiscal / Estorno / Manual, e por que a distinção importa na conferência.

Mais: conta sintética × analítica, centro de custo e rateio, e 4 FAQs (incluindo "escriturei e não apareceu no Diário, por quê?").

**Atualizadas** — `jornada-pendencias` (6º tipo com origem e caminho de resolução, os deep-links no passo a passo, 2 FAQs novas) e `gestor-fiscal/monitores/conecta` (a prévia é a mesma função do efetivo; o lançamento aparece após confirmar; a escrituração nunca trava por causa do contábil).

### 2.6 Parser do inventário de cobertura — corrigido e versionado

O parser vivia num scratchpad. Virou `scripts/inventario-docs.mjs`, versionado, para a métrica ser reproduzível.

A causa do falso-negativo: a detecção exigia `conceitos: [` seguido de `{` ou letra maiúscula, então **não via** `conceitos: conceitosFluxo` (identificador) nem `conceitos: [...conceitos('a','b')]` (só spread). A correção troca o teste por "a chave existe e o valor não é `[]`", resolvendo identificadores na própria definição do arquivo — e vale para **todas** as seções, não só `conceitos`.

| | Antes | Depois |
|---|---|---|
| BOA | 34 | **51** |
| MÍNIMA (placeholder) | 31 | **14** |
| PLENA / SEM DOC | 3 / 134 | 3 / 134 |

**17 telas** estavam subestimadas. Não houve superestimação: `manifestacao-destinatario` continua MÍNIMA porque de fato não tem `acoes`.

> Regra registrada no cabeçalho do script: **prefira o falso-positivo ao falso-negativo** — declarar presente algo duvidoso custa uma conferência; declarar ausente algo existente custa um lote inteiro de retrabalho.

**Cobertura atual: 202 telas no registry · PLENA 3 · BOA 51 · MÍNIMA 14 · SEM DOC 134 (PLENA+BOA = 26,7%).**

---

## 3. E2E — o cenário da pendência, ponta a ponta

Documento **`integdoc-m20n1`** (NF-e 100111 · `empresa-auditto-cliente1` · item "Pasta AZ ofício" · R$ 2.200,00), escriturado e sem regra.

**Passo 1 — a pendência está na inbox**
```
GET /v1/fiscal/acompanhamento/pendencias
  → integdoc-m20n1:sem_regra          (média)
    integdoc-m20n1:sem_contabilizacao (média) · "Item item-m20n1-1: sem regra fiscal aplicada."
```

**Passo 2 — criar a regra pelo deep-link** (`/central-regras-fiscais/produto?nova=1` → `POST /v1/fiscal/central-regras`)
```
  → 201 · RP-E2E-22 · Industrialização / Produção / CFOP 1101 · D 1.01.03 · C 2.01.01
```

**Passo 3 — reescriturar**
```
POST /v1/fiscal/conecta/integdoc-m20n1/escriturar
  → 200 · lancamentoContabilId "lanc-…kw61jn" · pendenciasContabeis []
```

**Passo 4 — a pendência sumiu e o lançamento nasceu**
```
GET /v1/fiscal/acompanhamento/pendencias  → []   ← as DUAS saíram
Aba Contabilidade: lancado · 2026-06 · 2 partidas
   D 1.01.03  2.200,00
   C 2.01.01  2.200,00           ΣD = ΣC ✓
GET /v1/contabilidade/lancamentos?origem=fiscal
  → origemTipo "DocumentoFiscal" · origemId "doc-m20n1" · lancado
```

**Passo 5 — idempotência**
Reescriturar de novo **não** cria um segundo lançamento: `contabilizarDocumento` devolve o existente. Substituir exige estornar antes (reverter-regra), coerente com "efetivado é imutável".

### Achado do E2E — a etapa 1 não passa pelo Conecta

`POST /conecta/:id/escriturar` **recusa** documento com item sem regra:

```
→ 422 "Aplique a regra tributária a todos os itens antes de escriturar."
```

Comportamento **pré-existente**, anterior à ponte contábil — não foi alterado (não é escopo deste lote, e mudar mecânica sem sua aprovação violaria a trava). Consequência prática: o estado "escriturado sem regra" chega por importação/legado, não pelo Conecta. A decisão D-1 ("o contábil não trava o fiscal") continua íntegra e provada; o que este 422 mostra é que **o fiscal trava a si mesmo**. Registrado como item 45 do BACKLOG para você decidir.

---

## 4. Travas de segurança revalidadas

```
PUT  /v1/contabilidade/lancamentos/<lançado>              → 422  (efetivado é imutável)
POST /v1/contabilidade/lancamentos/<id>/reverse [analista] → 403  (estorno é Gerente+)
POST /v1/fiscal/central-regras            [sem sessão]     → 401
POST /v1/contabilidade/periodos/:id/fechar [com pendência] → 409 + lista
```

Todas no **backend**. Auditoria em toda mutação (`lancamento_contabil.gerado`, `fechamento.assinado … COM CIÊNCIA`), multi-tenant respeitado.

`npx tsc --noEmit` limpo · `next lint` sem erros nem warnings.

---

## 5. O que vai para o BACKLOG

**Quitados:** item **22** (contabilização automática) e item **25** (o avanço "prévia não efetivada" — a prévia deixou de ser cálculo paralelo e passou a delegar para a montagem única, o que também corrigiu o bug latente de creditar o BRUTO ao fornecedor havendo retenção).

**Novos, todos achados deste ciclo:**

| # | Item | Por quê não foi feito agora |
|---|---|---|
| **43** | Conta da regra × Plano de Contas não são reconciliados — a regra guarda o código (`1.01.03`); se não existir no plano da empresa, a partida é gerada e a conta fica sem nome | É mecânica contábil: **depende da trava de fórmula**, não se decide no código |
| **44** | Contabilização retroativa em lote — 31 documentos escriturados sem lançamento; hoje só se resolve um a um | Fora do escopo do F1-B3; o passivo agora é visível, que era o ponto |
| **45** | O 422 do fiscal na escrituração sem regra | Comportamento pré-existente; mudá-lo é decisão de produto sua |

**Fases seguintes do plano** (não iniciadas): Fase 2 (memória contábil), Fase 3 (apurações modernizadas), Fase 4 (IRPJ/CSLL real).

---

## 6. Estado para a sua decisão

`feat/contabilizacao-efetiva` está pronta para **merge + tag + deploy**, sobre os ids unificados da `main`. Aguardando sua validação.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
