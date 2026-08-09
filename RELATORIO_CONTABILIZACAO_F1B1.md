# RELATÓRIO — Contabilização Efetiva (#22) · Lote F1-B1

**Data:** 2026-08-09 · **Branch:** `feat/contabilizacao-efetiva` · **Commit:** `e7e6d78`
**Escopo:** modelo + motor + geração no escriturar + estorno em cadeia, com o checkpoint por endpoint da spec.

---

## 1. O que foi feito — ligar dois motores, não construir um

Conforme o achado da Etapa A, `LancamentoContabil`, `PartidaContabil`, a validação de partidas dobradas e o estorno **já existiam** no módulo Contabilidade. Este lote **não criou entidade nova**: construiu a ponte.

### FONTE ÚNICA (a correção que você exigiu)
`lib/fiscal/motor-contabil.ts` · `montarPartidasDocumento()` é a **mesma** montagem usada pela **prévia** (modal de Escrituração) e pelo **lançamento efetivado**. `gerarPreviaLancamentos` passou a delegar a ela.

> **Bug corrigido no caminho:** a prévia anterior creditava o **bruto** ao fornecedor mesmo havendo retenções — divergia do que seria contabilizado. Agora prévia e efetivado mostram líquido + uma conta por tributo.

### Mecânica implementada (trava de fórmula D-6, como aprovada)
| Caso | Tratamento |
|---|---|
| **A** compra/despesa | D: conta de débito da regra (rateada por CC) · C: fornecedor |
| **B** com retenções | D: despesa **bruto** · C: fornecedor **líquido** · C: **uma conta por tributo** (2.1.3.1…6) |
| **C** regra com conta única | **pendência**, nunca 409 |
| **D** rateio multi-CC | N linhas somando o item; resíduo de arredondamento na última linha (soma exata) |

### Decisões aplicadas
D-1 pendência sem regra (o fiscal **não trava** pelo contábil) · D-2 um lançamento por documento · D-3 nasce `lancado` · D-4 competência contábil fechada bloqueia + pendência · D-5 estorno **Gerente+**.

### Infra
- **Store compartilhado** `mocks/state/contabilidade-state.ts` (padrão do `auditoria-state`): o Diário lê o **mesmo array vivo** que o fiscal alimenta. `handlers-contabilidade` trocou a cópia (`[...MOCK]`) por mutação in-place — sem isso o lançamento gerado não apareceria no módulo.
- **Plano de contas:** 6 contas analíticas de tributos retidos sob `2.1.3` (que virou sintética).
- **Imutabilidade:** `PUT` em lançamento efetivado → **422**.

---

## 2. CHECKPOINT por endpoint — evidências reais

### Caso B · serviço com retenções (`integdoc-4`, NFS-e de R$ 5.300,00)

| Conta | D/C | Valor |
|---|:--:|---:|
| 4.02.05 Despesa (bruto) | **D** | 5.300,00 |
| 2.1.3.1 IRRF a Recolher | C | 79,50 |
| 2.1.3.2 CSLL a Recolher | C | 53,00 |
| 2.1.3.3 PIS a Recolher | C | 34,45 |
| 2.1.3.4 COFINS a Recolher | C | 159,00 |
| 2.1.3.5 INSS a Recolher | C | 583,00 |
| 2.1.3.6 ISS a Recolher | C | 265,00 |
| **2.01.01 Fornecedores (líquido)** | **C** | **4.126,05** |

Σ retenções = **1.173,95** · 5.300,00 − 1.173,95 = **4.126,05** ✓ · **ΣD = ΣC = 5.300,00** ✓

### Caso A/D · multi-item com rateio (`integdoc-1`)
6 partidas · contas 1.01.03 / 2.01.01 · **ΣD = ΣC = 15.420,50** ✓

### Geração e vínculo
```
POST /v1/fiscal/conecta/integdoc-4/escriturar
  → 200 · lancamentoContabilId "lanc-1786279012450-86v2iw" · pendencias []

GET /v1/contabilidade/lancamentos   (Diário do módulo Contabilidade)
  → encontrado · status "lancado" · tipo "automatico"
    origemTipo "DocumentoFiscal" · origemId "doc-fiscal-9" · competencia "2026-07"
    historico "NFSE 000210 · … · Prestação de serviços contábeis" · 8 partidas · ΣD=ΣC=5.300,00
```
✅ Store compartilhado provado: o lançamento nasceu no fiscal e aparece no Diário.

### Estorno em cadeia
```
POST /v1/fiscal/conecta/integdoc-4/reverter-regra
  → 200 · estornosContabeis ["lanc-1786279052604-d4s0hs"]

Original  → status "estornado" · estornadoPorLancamentoId "lanc-…d4s0hs"
Estorno   → status "lancado" · origemTipo "LancamentoContabil" · origemId (o original)
            historico "Estorno — NFSE 000210 · …" · 8 partidas · ΣD=ΣC=5.300,00
            partidas INVERTIDAS (D↔C confirmado)
```

### Travas
```
PUT  /v1/contabilidade/lancamentos/<lançado>        → 422 "Lançamento efetivado é imutável — corrija por estorno e nova contabilização."
POST /v1/contabilidade/lancamentos {D 100 × C 90}   → 409 "Lançamento desbalanceado — soma de débitos deve ser igual à soma de créditos."
POST /v1/contabilidade/lancamentos/<id>/reverse  [papel analista] → 403 "Estorno de lançamento requer papel Gerente ou superior."
```

---

## 3. Verificações
`tsc --noEmit` limpo · `next lint` sem erros/warnings · nenhuma entidade duplicada · auditoria registrada na geração e no estorno (`lancamento_contabil.gerado` / `.estornado`).

## 4. Próximos lotes
- **F1-B2 — Telas:** origem/filtro/chip e deep-link no Diário; aba Contabilidade do modal com prévia + efetivados + estornos; fila de pendências contábeis.
- **F1-B3 — Docs + E2E:** doc in-app; jornada capturar → check-in → escriturar → lançamento → Contabilidade → estornar → reescriturar.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais — os identificadores são fixtures de desenvolvimento.*
