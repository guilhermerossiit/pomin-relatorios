# RELATÓRIO — Merge do item 41 + Integração na branch da Contabilização

**Data:** 2026-08-09
**Etapas 1 e 2** da ordem recomendada · **PARE** antes do F1-B3.

---

## Etapa 1 — Merge do item 41 na `main`

| Ação | Resultado |
|---|---|
| Lint gate antes do merge | ✔ sem erros/warnings |
| Merge `--no-ff` | `fa037fe..7fc05a1` · 11 arquivos, +297/−72 |
| Tag | **`ids-empresa-unificados`** |
| Push | `main`, branch `fix/unificacao-ids-empresa` e a tag — todos no GitHub |

O merge levou junto o **G0** (binding das docs do BPM), que estava na mesma branch.

---

## Etapa 2 — Integração da `main` na `feat/contabilizacao-efetiva`

**Merge automático, sem conflitos** — apesar de as duas frentes tocarem `handlers-fiscal.ts`, elas mexem em pontos distintos (a unificação nos ids/fixtures e no bloco do portal-cliente; a contabilização nos helpers e no `escriturar`).

### Integridade pós-merge
- **Ids longos sobreviventes: ZERO** — a unificação não foi desfeita pelo merge.
- `lib/fiscal/motor-contabil.ts` e `mocks/state/contabilidade-state.ts` **presentes** — o motor da ponte não foi perdido.
- `tsc --noEmit` limpo · `next lint` sem erros/warnings.

### Revalidação do F1-B1 por endpoint (pós-integração)

**Geração**
```
POST /v1/fiscal/conecta/integdoc-4/escriturar
  → 200 · lancamentoContabilId "lanc-…3mx0ac" · pendencias []
Aba Contabilidade: status "lancado" · 8 partidas · ΣD = ΣC = 5.300,00 ✓
```

**Diário (store compartilhado + vínculo)**
```
GET /v1/contabilidade/lancamentos
  → lançamento encontrado · status "lancado"
    origemTipo "DocumentoFiscal" · origemId "doc-fiscal-9" · competência "2026-07"
    clienteId "empresa-auditto-cliente1"   ← ID UNIFICADO, empresa de OUTRO tenant
```

> Este último campo é a prova cruzada das duas frentes: o lançamento contábil nasceu já com a **grafia unificada** do item 41, na empresa que pertence a `org-auditto`. Se a migração tivesse quebrado o mapa de tenant, seria aqui que apareceria.

**Estorno em cadeia**
```
POST /v1/fiscal/conecta/integdoc-4/reverter-regra
  → original "estornado" com estornadoPorLancamentoId vinculado
    estorno "lancado" · 8 partidas (invertidas)
```

**Travas**
```
PUT  /v1/contabilidade/lancamentos/<lançado>            → 422 (imutável)
POST /v1/contabilidade/lancamentos/<id>/reverse [analista] → 403 (Gerente+)
```

✅ **Nada regrediu na integração.** Geração, vínculo bidirecional, estorno em cadeia e as travas continuam como no checkpoint original do F1-B1.

---

## Estado das branches

| Branch | Situação |
|---|---|
| `main` | contém o item 41 + G0 · tag `ids-empresa-unificados` |
| `feat/contabilizacao-efetiva` | F1-B1 + F1-B2 **sobre os ids unificados** · pronta para o F1-B3 |

---

## Próximo — F1-B3 (não iniciado; **PARE** aqui)

Conforme combinado, o lote inclui:
1. **Pendência contábil na INBOX** — reconhecido como requisito, não acabamento: a decisão **D-1** (escriturar sem regra não trava o fiscal) só se sustenta se a pendência for **visível**;
2. **Filtro/chip de origem** no Diário (a coluna já existe);
3. **Docs in-app** das telas afetadas;
4. **E2E completo**, incluindo o cenário: escriturar sem regra → pendência na inbox → criar regra pelo deep-link → reescriturar → **pendência some**;
5. **Correção do parser** do inventário de cobertura (falso-negativo do `conceitos: <variável>`), para a métrica não subestimar o que já existe.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
