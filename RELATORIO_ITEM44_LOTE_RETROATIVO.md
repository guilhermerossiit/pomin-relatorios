# RELATÓRIO — Item 44 · Contabilização Retroativa em Lote

**Ciclo curto 2** · branch `feat/contabilizacao-retroativa` · sobre o item 43 já mergeado.

---

## 1. O que foi entregue

A pendência "Sem contabilização" tornou o passivo **visível** (F1-B3). Este ciclo o torna **resolúvel em escala**: seleção múltipla na inbox → prévia do que vai acontecer → execução com auditoria → relatório do resultado.

### Fonte única: `avaliarContabilizacao`

A checagem "esse documento vai contabilizar?" existia dentro de `contabilizarDocumento`, misturada com a gravação. Foi extraída para uma função **sem efeito colateral**, e agora **três** caminhos usam a mesma resposta:

| Quem pergunta | Para quê |
|---|---|
| Prévia do lote | dizer o que vai acontecer antes de acontecer |
| Execução do lote | fazer acontecer |
| Pendência derivada na inbox | explicar por que ainda não aconteceu |

Prévia que promete o que a execução não cumpre seria pior que não ter prévia — o mesmo princípio da prévia de partidas do F1-B1.

---

## 2. Checkpoint — o lote de 31 documentos

### Prévia (antes de executar)

```
POST /v1/fiscal/contabilizacao-retroativa/previa   { documentoIds: [31] }
  → total 31 · vaiGerar 26 · exigemCiencia 2 · falhas 3 · jaContabilizados 0
    valorTotal            R$ 164.200,06
    valorTotalComCiencia  R$ 170.000,06
```

Cada linha traz o **motivo** quando não vai gerar:

| Situação | Qtd | Motivo exibido |
|---|---|---|
| Vai gerar | 26 | — |
| Competência fechada | 2 | "Competência contábil 2026-06 fechada — reabrir o período ou lançar na competência seguinte." |
| Não vai gerar | 3 | "Nenhuma regra de contabilização aplicável aos itens." · "Item item-m15n1-1: sem regra fiscal aplicada." · "Item item-m20n1-1: sem regra fiscal aplicada." |

### Execução

```
Lote 1 (sem ciência)   → solicitados 31 · gerados 26 · falhas 5 · R$ 164.200,06
   falhas: 2 competência fechada + 3 sem regra — cada uma com seu motivo no relatório
Lote 2 (com ciência)   → solicitados  2 · gerados  2 · falhas 0 · R$   5.800,00
```

**Falha isolada não abortou o lote:** 5 documentos falharam e os outros 26 foram contabilizados assim mesmo. Nenhum sumiu em silêncio — todos voltaram no relatório com o motivo, inclusive os que falhariam por conta inválida (a validação do item 43 entra por dentro da mesma avaliação).

### Conferência contábil

```
Lançamentos gerados nos dois lotes: 28
Desbalanceados (ΣD ≠ ΣC):            0
Auditoria de contas pós-lote:  98 partidas · 0 com problema · 0 de outra empresa
```

As 90 partidas novas nasceram todas com conta válida do plano da própria empresa — o item 43 sustentando o item 44 na prática.

### Pendências

```
Antes:  31 pendências "Sem contabilização"
Depois:  3 — exatamente os documentos que não têm regra fiscal
```

**28 de 31 resolvidas.** As 3 restantes não são falha do lote: são documentos que genuinamente precisam de uma regra antes de poder ser contabilizados, e continuam na inbox com o deep-link "Criar a regra que falta".

---

## 3. Competência fechada — a mesma ciência do fechamento, e um degrau a mais

Você pediu a mesma ciência exigida no fechamento. Implementei isso **e** acrescentei o gate de permissão correspondente, porque só a ciência me pareceu pouco:

| Chamada | Resultado |
|---|---|
| `analista` · sem ciência | **200** — pode lançar normalmente |
| `analista` · **com ciência** de competência fechada | **403** |
| escopo de portal (fornecedor/cliente) | **403** — ação interna |

**O raciocínio:** lançar num mês já assinado desfaz, na prática, parte do efeito de um fechamento. Quem não tem permissão para fechar o período não deveria poder alterar o resultado de um período fechado. O gate exige `contabilidade/fechamento:fechar` (Gerente+) **além** de `lancamentos:lancar`. Se você preferir que a ciência sozinha baste, é uma linha — mas registro que a assimetria me parece difícil de defender.

**Auditoria:** 28 eventos `lancamento_contabil.gerado_retroativo`, sendo **2 em criticidade alta** com `comCiencia: true`, `competencia: 2026-06` e o `loteId`. A decisão de mexer num mês fechado fica com nome, data e lote.

---

## 4. A tela

- **Barra de seleção** no topo da inbox, com "Selecionar todas as pendências contábeis (N)" e o contador de selecionados. Aparece só quando há pendência contábil na lista filtrada.
- **Caixa por pendência** apenas nas do tipo "Sem contabilização" — e apenas no modo interno: contabilizar é ação do escritório, não do portal.
- **Seleção por DOCUMENTO, não por pendência** — a mesma nota pode aparecer em dois tipos, e contabilizá-la duas vezes não faria sentido.
- **Diálogo em três tempos**: prévia (4 números + tabela com motivo por linha) → checkbox de ciência quando há competência fechada → relatório (gerados / falhas / já contabilizados, valor total, número do lote, linha a linha).
- A prévia é **recalculada a cada abertura**: o estado pode ter mudado desde a última vez (alguém corrigiu a regra, o período foi reaberto). Prévia velha decidiria com o mundo de ontem.

---

## 5. Documentação in-app (mesmo ciclo)

Em `Acompanhamento › Pendências`:
- **ação nova** "Contabilizar várias notas de uma vez (lote retroativo)" — pré-condições (incluindo o degrau de permissão), 5 passos e o resultado esperado;
- **conceito novo** "Contabilização retroativa em lote", explicando prévia, falha isolada e a regra da competência fechada;
- sinônimos para a busca: *lote, retroativo, contabilizar tudo, resolver a fila, passivo*.

---

## 6. Estado

`feat/contabilizacao-retroativa` pronta para merge. **Item 44 quitado.**

`tsc --noEmit` limpo · `next lint` sem erros nem warnings.

Nada novo foi para o backlog. Com o 43 e o 44 fechados, o cadastro de contas está confiável e o passivo drenado — as condições que você estabeleceu para abrir a **Fase 2 (memória contábil)**.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
