# RELATÓRIO — Fase 3 · F3-B2 (Painel Central de Apurações na tela)

**Branch:** `feat/apuracoes-modernas` · **Nenhuma fórmula alterada.**

---

## 1. A tela

`/gestor-fiscal/apuracoes` — a rota da porta de entrada não existia; o menu ia direto aos 11 tributos.

- **Card por tributo** com o valor, a **frase pronta**, situação, status e "guia gerada".
- **Dois totais separados**: *a recolher* e *saldo credor a transportar*, com a explicação embaixo do segundo (*"Não abate o total acima — segue para a competência seguinte"*).
- **Alertas** acima dos cards, na ordem em que exigem ação.
- **Seletor empresa + competência** com os dois virando **chip** no painel de filtros aplicados.
- **Estado vazio que se explica**: *"as apurações nascem da escrituração"* — não um "sem dados" mudo.

O front **não calcula nada**: valores, totais, situação e até as frases vêm montados do endpoint. Recalcular aqui seria a quinta ocorrência da fonte dupla.

---

## 2. Checkpoint

### 2.1 Números da tela = números do endpoint

| Empresa | Competência | Endpoint | Na tela |
|---|---|---|---|
| Iguassu | 2026-07 | total R$ 961,29 · 1 card · 2 alertas | ✔ todos presentes |
| Metalúrgica | 2026-04 | total R$ 4.910,00 · **8 cards** · 2 alertas | ✔ **8/8 frases**, total e alertas — contador "8 de 8 tributos" |

Frase renderizada, textual: `"ICMS de julho: R$ 961,29 a recolher"`.

### 2.2 Card leva à tela do tributo na competência certa

```
/gestor-fiscal/apuracoes/icms?competencia=2026-07&empresa=empresa-iguassu
```

### 2.3 Nenhuma rota antiga quebrou (D-ap-2)

As **11** rotas responderam **200** com os parâmetros novos: icms · iss · pis-cofins · ipi · ibs · cbs · irpj · csll · retencoes-federais · config-lucro-real · config-lucro-presumido.

No menu, o Painel entrou **como primeiro item** do grupo Apurações e os 11 tributos continuam listados abaixo — deep-links, docs e o hábito de quem já usa seguem valendo.

---

## 3. Um defeito meu que o teste dos 6 temas pegou

Montei o card como um `<a>` cru com `bg-pomin-surface-raised`, em vez do componente `Card` compartilhado. Medindo:

```
DARK-PURPLE   --pomin-surface-raised = #0f172a     (token correto)
              body                   = rgb(2,6,23) (Dark aplicado)
              Card compartilhado     = rgb(15,23,42) ✔
              MEU card (<a> cru)     = rgb(255,255,255) ✘  ← branco no Dark
```

Persegui a causa (regras que casavam, variável em cada ancestral, folhas de estilo) e a conclusão prática foi mais simples que o mistério de CSS: **a casa manda usar o componente compartilhado, e o componente compartilhado media certo.** Troquei o `<a>` cru por `<Card>` dentro do `<Link>`.

Depois da correção:

| | light-purple / green / blue | dark-purple / green / blue |
|---|---|---|
| Superfície do card | `rgb(255,255,255)` | **`rgb(15,23,42)`** |

**2 superfícies distintas nos 6 temas** — que é o resultado esperado (a cor é ortogonal ao modo; o que muda entre claro e escuro é a superfície).

> Registro isto com destaque porque é exatamente o tipo de coisa que passaria despercebida sem o checkpoint de temas: a tela parecia perfeita no Light, que é onde eu estava olhando.

---

## 4. Documentação in-app

Doc nova de `gestor-fiscal/apuracoes`, verificada renderizando no painel (sem "em construção"): 1 ação (ler a competência de uma vez) e 4 conceitos —

- **a recolher × nada a recolher × saldo credor** — as três situações, e que "nada a recolher" não é pendência;
- **por que os dois totais não se somam** — são naturezas opostas, e frequentemente de tributos diferentes: o crédito de ICMS não abate o PIS;
- **status da apuração** — aberta recalcula, fechada congela;
- **os números vêm todos da mesma fonte** — o que garante que cliente e escritório nunca vejam valores diferentes.

Mais 3 FAQs, incluindo *"um tributo que eu esperava não aparece — por quê?"*.

---

## 5. Regra inegociável

`motor-apuracao.ts` intocado. O endpoint agrega; a tela renderiza. Nenhum número é calculado no front.

`tsc --noEmit` limpo · `next lint` sem erros nem warnings.

---

## 6. Próximo — F3-B3 (não iniciado)

Tela do tributo em camadas: resumo executivo com os 5 números clicáveis, detalhamento colapsado e drill-down até os documentos que compõem cada valor (D-ap-3: em modal).

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
