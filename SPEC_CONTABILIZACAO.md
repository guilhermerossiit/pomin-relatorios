# SPEC — CONTABILIZAÇÃO EFETIVA (#22)

**Fase 1 · Etapa A** do plano *Gestor Fiscal × Contabilidade*
**Data:** 2026-08-09 · **Branch:** `feat/contabilizacao-efetiva` (a partir da `main`)
**Natureza:** especificação. **Nenhum código foi escrito.** PARE para aprovação do PO.

---

## 0. ACHADO CENTRAL — o modelo já existe; o que falta é a PONTE

O plano previa, no F1-B1, *"criar `LancamentoContabil` + validação de partidas dobradas + estorno vinculado"*. O levantamento mostra que **o módulo Contabilidade já tem tudo isso, implementado e enforçado**:

| Peça que o plano previa criar | Situação real | Onde |
|---|---|---|
| Entidade `LancamentoContabil` | **JÁ EXISTE** — com `origemTipo`/`origemId`, `estornadoPorLancamentoId`, `partidas[]`, `status`, `usuario`, `competencia`, `loteId` | `types/contabilidade.ts` |
| `PartidaContabil` (conta + CC + tipo + valor) | **JÁ EXISTE** | idem |
| Validação de partidas dobradas | **JÁ EXISTE e é enforçada** — `validarPartidaDobrada()`, rejeita com **409** ao postar | `handlers-contabilidade.ts:126` |
| Estorno vinculado | **JÁ EXISTE** — `POST /lancamentos/:id/reverse` grava `origemTipo:'LancamentoContabil'` | `handlers-contabilidade.ts:324` |
| Tela de lançamentos (Diário) | **JÁ EXISTE e é real** — listagem, detalhe, postar, estornar | `app/(app)/contabilidade/lancamentos` |
| Vínculo a documento fiscal | **PADRÃO JÁ EXISTE** — a fixture `lanc-4` usa `origemTipo:'DocumentoFiscal'` + `origemId` | `fixtures/contabilidade.ts:181` |

**Conclusão:** a Fase 1 **não é construir um motor contábil — é ligar dois motores que já existem.** Isso reduz o escopo, elimina risco de duplicar entidade (regra do `CLAUDE.md`) e concentra o trabalho onde ele realmente está: a geração no ato da escrituração, o estorno em cadeia e a navegação.

### O que de fato falta

1. **A ponte:** `POST /v1/fiscal/conecta/:id/escriturar` hoje **não gera lançamento** — só existe a prévia (`gerarPreviaLancamentos`), que nunca é efetivada.
2. **Tradução prévia → partidas** com a mecânica contábil correta (§2.2 — é aqui que mora a TRAVA DE FÓRMULA).
3. **Estorno em cadeia:** reverter a escrituração precisa estornar o lançamento.
4. **Navegação bidirecional** documento ↔ lançamento.
5. **Aba Contabilidade do modal:** hoje mostra só prévia; passa a mostrar prévia + efetivados + estornos.

---

## 1. Modelo do lançamento real — REUSO, sem entidade nova

Usa-se `LancamentoContabil` como está. Preenchimento na geração automática:

| Campo | Valor na contabilização automática |
|---|---|
| `clienteId` / `clienteNome` | empresa do documento (`doc.clienteId`) — multi-tenant preservado |
| `tipo` | `'automatico'` |
| `origemTipo` / `origemId` | `'DocumentoFiscal'` / `doc.id` — **o vínculo bidirecional** |
| `competencia` | competência do documento (§5) |
| `dataLancamento` | data da escrituração (o ato), não a de emissão |
| `historico` | gerado do contexto (§4) |
| `status` | `'lancado'` (nasce efetivado — decisão **D-3**) |
| `usuarioNome` | usuário que confirmou a escrituração (autoria real, não "motor") |
| `partidas[]` | traduzidas da prévia (§2.2) |
| `loteId` | `null` por documento; preenchido se **D-2** = consolidado |

**Nenhum campo novo é necessário.** Ganho: o lançamento gerado pelo fiscal é indistinguível — para as telas, o balancete e o SPED — de um lançamento manual.

---

## 2. Gatilho e tradução

### 2.1 Gatilho
No `POST /v1/fiscal/conecta/:id/escriturar`, **após** a escrituração ser confirmada com sucesso: gerar o lançamento a partir dos itens escriturados e das Regras de Contabilização já resolvidas. Documento sem regra de contabilização → **pendência**, não bloqueio (decisão **D-1**).

### 2.2 Tradução prévia → partidas ⚠️ **TRAVA DE FÓRMULA (F1-B1)**

A prévia atual (`gerarPreviaLancamentos`) produz, **por item**, uma linha de débito e uma de crédito no **valor bruto do item**, rateadas por centro de custo. Isso fecha as partidas no caso simples — **mas há três casos em que não fecha ou não reflete a realidade contábil**:

| Caso | Problema da prévia atual | Mecânica proposta (1 linha cada — a aprovar) |
|---|---|---|
| **A · Compra de mercadoria** | ok | D: Estoques/Despesa (conta da regra) · C: Fornecedores a Pagar — pelo valor do item |
| **B · Serviço tomado com retenções** | prévia credita o **bruto** ao fornecedor; a realidade é **líquido + tributos retidos** | D: Despesa (bruto) · C: Fornecedores (bruto − retenções) · C: uma conta por tributo retido (IRRF/CSLL/PIS/COFINS/INSS/ISS) — soma fecha |
| **C · Regra com apenas UMA conta preenchida** | gera só um lado → **partida não fecha** → o `POST` seria rejeitado com 409 | não gerar lançamento; abrir **pendência** "regra de contabilização incompleta" (decisão **D-1**) |
| **D · Rateio multi-centro de custo** | ok (rateio 100% já validado na Central) | as N linhas do rateio somam o valor do item; a contrapartida pode ser única |

**Nada disso será implementado antes do seu OK**, conforme o princípio 5 do plano. O caso **B** é o que muda o número na tela — é a mecânica que precisa da sua confirmação explícita.

---

## 3. Imutabilidade e estorno

- Lançamento com `status:'lancado'` **não se edita** — o `PUT` existente deve passar a rejeitar (**422**) quando `status !== 'rascunho'`. *(Hoje o PUT valida balanceamento, mas não imutabilidade — é um ajuste a fazer.)*
- Correção = **estorno** (`POST /:id/reverse`, já implementado: cria o inverso e marca `estornadoPorLancamentoId`) + nova contabilização.
- **Estorno em cadeia:** reverter/reabrir a escrituração de um documento estorna automaticamente o(s) lançamento(s) com `origemTipo:'DocumentoFiscal'` + `origemId = doc.id`, com auditoria em ambos os módulos.
- **Reescrituração** após estorno gera lançamento novo (não ressuscita o antigo) — trilha completa preservada.

---

## 4. Histórico padrão

Gerado do contexto, **editável** e com origem marcada:

```
NF-e 000123 · Suprimentos TF · Compra de mercadoria para revenda
NFS-e 000045 · Landim Advocacia · Serviço tomado (com retenções)
```

Padrão: `{tipo do documento} {número} · {participante} · {natureza da operação}`. Editado pelo humano → marcado como `manual` e **nunca sobrescrito** por reprocessamento (enriquecimento aditivo, princípio 4).

---

## 5. Competência contábil

- **Competência do lançamento = competência do documento fiscal** (não a data da escrituração) — assim o mês fiscal e o contábil batem.
- **Cruzamento com o fechar/reabrir da apuração fiscal** e com `PeriodoContabil` (entidade que já existe no módulo): lançar em competência **contábil fechada** exige regra → decisão **D-4**.
- A apuração fiscal materializada (P-13) e o lançamento contábil são derivações independentes do mesmo fato — não se recalculam entre si.

---

## 6. Navegação bidirecional

- **Documento → lançamento:** aba Contabilidade do modal de Escrituração lista os lançamentos efetivados com deep-link para `/contabilidade/lancamentos?lancamento=<id>`.
- **Lançamento → documento:** no detalhe do lançamento, quando `origemTipo === 'DocumentoFiscal'`, link para o documento no Conecta (`?nota=<integracaoId>`), padrão de deep-link já usado no Acompanhamento/Pedido de Compra.

---

## 7. Telas do módulo Contabilidade — real × placeholder

| Tela | Estado | Ação nesta fase |
|---|---|---|
| **Lançamentos Contábeis** | **real** (listagem, detalhe, postar, estornar) | + filtro/chip por origem (Fiscal × Manual), coluna de origem, deep-link |
| Plano de Contas · Centros de Custo | real (catálogo) | consumir — sem mudança |
| Lotes · Rateios | real | relevante se **D-2** = consolidado |
| Lançamentos Automáticos | a verificar no B2 | pode ser a casa natural da listagem "gerados pelo fiscal" |
| Conciliações · Fechamento · Demonstrações · SPED · Auditoria | herdadas da unificação | **fora do escopo** desta fase; mapear no B2 o que é real vs placeholder |

---

## 8. DECISÕES DE NEGÓCIO — **PARE** (preciso do seu aval)

**D-1 · Documento sem regra de contabilização (ou com regra incompleta — caso C):** *(recomendo **a**)*
- **(a)** Escritura normalmente e abre **pendência contábil** (aparece nas Pendências do Acompanhamento e numa fila no módulo Contabilidade). O fiscal não trava por causa do contábil.
- **(b)** Bloqueia a escrituração até haver regra completa.

**D-2 · Granularidade:** *(recomendo **a**)*
- **(a)** **Um lançamento por documento** — rastreabilidade 1:1, estorno cirúrgico, casa com `origemId`.
- **(b)** Consolidado por dia/lote (usa `loteId`) — menos registros, porém estorno e conferência ficam grosseiros.

**D-3 · Status ao nascer:** *(recomendo **a**)*
- **(a)** Nasce **`lancado`** (efetivado) — a escrituração já é o ato de confirmação humana.
- **(b)** Nasce **`rascunho`** e exige um segundo aceite no módulo Contabilidade (dupla conferência, mais lento).

**D-4 · Competência contábil fechada:** *(recomendo **a**)*
- **(a)** **Bloqueia** o lançamento e abre pendência ("competência fechada — reabrir ou lançar no mês seguinte"), decisão explícita do contador.
- **(b)** Permite retroativo com registro em auditoria.
- **(c)** Lança automaticamente na competência aberta seguinte (silencioso — **contra o princípio 3**, não recomendo).

**D-5 · Quem pode estornar (RBAC):** *(recomendo **b**)*
- **(a)** Qualquer papel com acesso a Contabilidade.
- **(b)** **Gerente+** (mesmo padrão do escape de duplicidade no Conecta).
- **(c)** Somente perfil contábil dedicado.

**D-6 · Mecânica do caso B (retenções)** — a TRAVA DE FÓRMULA da §2.2: confirma o desdobramento **bruto → líquido + contas de tributos retidos**?

---

## 9. Escopo dos lotes (após aprovação)

- **F1-B1 — Motor da ponte:** tradução prévia → partidas (mecânica aprovada em D-6), geração no `escriturar`, estorno em cadeia, imutabilidade (422 no PUT), pendência quando não houver regra. **Checkpoint por endpoint:** escriturar → lançamento com partidas fechando na ponta do lápis (com rateio de CC **e** com retenções); reverter → estorno vinculado; PUT em lançado → 422; D ≠ C → 409.
- **F1-B2 — Telas:** origem/filtro/chip e deep-link no Diário; aba Contabilidade do modal com prévia + efetivados + estornos; fila de pendências contábeis.
- **F1-B3 — Docs + E2E:** doc in-app das telas afetadas; jornada capturar → check-in → escriturar → lançamento efetivado → visível na Contabilidade → estornar → reescriturar.

---

## PARADA

Aguardo: **(1)** confirmação do reuso do `LancamentoContabil` existente (§0/§1), **(2)** as decisões **D-1 a D-5**, e **(3)** o **OK da trava de fórmula D-6** — sem ele o F1-B1 não começa.

*Especificação técnica. Não contém credenciais, tokens nem dados de clientes reais — os identificadores citados são fixtures do ambiente de desenvolvimento.*
