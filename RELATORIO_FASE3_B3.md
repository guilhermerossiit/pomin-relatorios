# RELATÓRIO — Fase 3 · F3-B3 (Tela do tributo em camadas)

**Branch:** `feat/apuracoes-modernas` · **Nenhuma fórmula alterada.**

---

## 1. O que foi entregue

Componente `ResumoExecutivoApuracao`, inserido no `ApuracaoView` — o componente compartilhado pelas **9 telas de tributo**. Uma mudança, nove telas modernizadas.

### 1.1 Resumo executivo — os 5 números na ordem da conta

```
DÉBITOS        −CRÉDITOS      −SALDO ANTERIOR   +AJUSTES      =SALDO A RECOLHER
R$ 2.775,69     R$ 1.814,40    R$ 0,00           R$ 0,00        R$ 961,29
```

2.775,69 − 1.814,40 = **961,29** — a conta fecha na tela, com o sinal de cada parcela à mostra. As quatro primeiras são clicáveis ("ver documentos" no hover); a última é o resultado.

### 1.2 Detalhamento colapsado

A lista de documentos, que antes vinha aberta e empurrava a conta para baixo, virou "Documentos (N)" sob demanda. O "Valor apurado: R$ X" solto do cabeçalho saiu — o resumo executivo diz isso melhor e com a conta inteira.

### 1.3 Drill-down em modal (D-ap-3)

```
Débitos · ICMS · 2026-07
  000123 ↗  Jogo de cama king 300 fios · Têxtil Cataratas    R$ 9.000,00   R$ 1.620,00
  000123 ↗  Toalha de banho premium 90x150 · Têxtil …        R$ 5.100,00   R$   918,00
  000123 ↗  Roupão felpudo adulto · Têxtil Cataratas         R$ 1.320,50   R$   237,69
  Soma de 3 linha(s)                                                       R$ 2.775,69
  ✓ A soma das linhas é exatamente o valor exibido na apuração (R$ 2.775,69).
```

O número do documento é **deep-link ao Conecta**. Quando o documento não tem integração, o número aparece sem link — melhor que um link que leva a lugar nenhum.

**A conferência viaja com o dado:** o endpoint devolve `confere`, e a tela exibe a frase. O operador não precisa somar de cabeça para confiar.

---

## 2. Checkpoint principal — a soma bate na ponta do lápis

O endpoint decompõe usando a **mesma `config.contribuicao`** que o motor usa para somar. Não há cálculo "de exibição" e outro "de verdade".

| | |
|---|---|
| Rubricas testadas | **16** |
| Tributos distintos | **8** (ICMS, ISS, PIS, COFINS, IPI, IBS, CBS, Retenções) |
| **Divergentes** | **0** |

Exemplos, com a soma conferida linha a linha:

| Tributo | Comp. | Rubrica | Linhas | Soma das linhas | Número exibido |
|---|---|---|---|---|---|
| ICMS | 2026-07 | débitos | 3 | 2.775,69 | **2.775,69** |
| ICMS | 2026-07 | créditos | 1 | 1.814,40 | **1.814,40** |
| ISS | 2026-04 | débitos | 2 | 400,00 | **400,00** |
| PIS | 2026-04 | créditos | 1 | 66,00 | **66,00** |

Você pediu ao menos 3 tributos; foram **8**.

---

## 3. Um problema que medi e NÃO resolvi

Ao rodar o teste dos 6 temas nesta tela, encontrei algo que não consegui explicar dentro deste lote, e prefiro reportar do que empurrar:

```
Sob dark-purple, na MESMA tela:
  --pomin-surface-raised = #0f172a          (token correto)
  body                   = rgb(2,6,23)      (Dark aplicado)
  15 elementos com a MESMA classe .card-premium:
      3  →  rgb(15,23,42)    ✔ correto
     12  →  rgb(255,255,255) ✘ claros no Dark
```

**Hipóteses que descartei por medição, não por palpite:**
- wrapper `<button>` repintando o filho → removido, **persiste**;
- `role="button"` → removido, **persiste**;
- algum ancestral redeclarando a variável → verificado nível a nível, **todos com `#0f172a`**;
- regra CSS concorrente → a única regra que casa o elemento aponta para o token.

E o dado que mais confunde: **o Painel de Apurações (F3-B2), usando o mesmo componente `Card`, mede correto.** Então é específico de contexto, não do componente.

**Não vou declarar "6 temas OK" neste lote.** Registrei como **item 47 do BACKLOG** (MÉDIA), com todas as medições e as hipóteses já eliminadas, para uma investigação dedicada — provavelmente com o dev server reiniciado, pela armadilha conhecida do JIT do Tailwind neste projeto.

O que isso significa na prática: no **Light** (onde os três temas medem igual e correto) a tela está perfeita; no **Dark**, parte das parcelas do resumo fica clara. Funcionalmente tudo opera; é legibilidade.

---

## 4. Regra inegociável

`motor-apuracao.ts` **intocado**. O endpoint de composição *lê* `CONFIG_TRIBUTOS[tributo].contribuicao` — a mesma função — sem alterá-la. Nenhum número é calculado no front.

`tsc --noEmit` limpo · `next lint` sem erros nem warnings.

---

## 5. Pendente deste lote

- **Doc in-app** das telas de tributo ainda **não atualizada** com o resumo executivo e o drill-down — fica para o F3-B4, junto com o comparativo. Registro explicitamente porque a doc é definição de pronto, e não quero que isso passe como feito.
- **Item 47** (temas) aguardando investigação dedicada.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
