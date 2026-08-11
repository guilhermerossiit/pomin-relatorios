# RELATÓRIO — Fase 3 · F3-B1 (fonte única + endpoint do Painel)

**Branch:** `feat/apuracoes-modernas` · **Nenhuma fórmula alterada.**

---

## 1. A correção — prioridade máxima, como você determinou

### 1.1 Portal do Cliente › Apurações

```diff
  const lista = apuracoes
    .filter((a) => a.clienteId === empresaId)
+   .map(enriquecerApuracao)          // a MESMA derivação que as telas usam
    .map((a) => ({ …, aRecolher: a.saldoDevedor ?? 0, … }))
```

Antes lia o campo materializado do fixture, que nasce zerado. **Era o cliente vendo R$ 0,00 onde o motor calculava R$ 961,29.**

---

## 2. Varredura preventiva — e ela achou **mais dois**

Fui atrás de todo consumidor do array `apuracoes` e de leituras de campo materializado. Resultado: o bug reportado na Etapa A **não era único**.

| # | Local | Situação | Gravidade |
|---|---|---|---|
| 1 | `GET /portal-cliente/apuracoes` | lia `a.saldoDevedor` cru | **cliente via 0,00** |
| 2 | **Monitor de Saúde Fiscal do Portal** | somava `a.saldoDevedor ?? 0` cru | **cliente via 0,00 — e é a PRIMEIRA tela que ele abre** |
| 3 | **`saldoCredorAnteriorDe`** | lia `prev.saldoCredorTransportado` cru | **a cadeia inteira de competências se apoiava em cache** |

**O #2** é o mesmo bug no lugar mais visível possível: o Monitor é a porta de entrada do cliente. Corrigido com a mesma linha (`.map(enriquecerApuracao)`), e provado — o Monitor agora devolve **"R$ 961,29 a recolher"**.

**O #3 é o mais insidioso e eu não o teria encontrado sem a varredura que você pediu.** O saldo credor da competência anterior alimenta o cálculo da competência seguinte. Lendo do campo materializado, a competência N se apoiava num número de N−1 que podia nunca ter sido recalculado — e o erro se propagaria **para frente**, mês após mês, sem nunca aparecer. Agora a competência anterior é derivada recursivamente, com duas travas: `emDerivacao` (corta ciclo em dado inconsistente) e profundidade máxima de 24 competências.

> Isto não é mudança de fórmula: a regra continua sendo *"saldo credor anterior = crédito transportado da competência anterior"*. O que mudou é de onde esse número vem — do cálculo, não do cache. É exatamente a regra que você mandou escrever.

---

## 3. Endpoint do Painel — `GET /v1/fiscal/apuracoes/painel`

Puro **agregado**: não calcula nada por conta própria; cada tributo passa por `enriquecerApuracao`.

- Cards por tributo com `apurado`, `saldoCredor`, `situacao` (**a_recolher · nada_a_recolher · credor**), `status`, `vencimentoGuia` e o `href` da tela.
- **Frase pronta** em cada card, como você especificou: `"ICMS de julho: R$ 961,29 a recolher"`.
- **Total do mês com a recolher e saldo credor SEPARADOS** — somar os dois num número só seria mentira aritmética: um é dívida, o outro é crédito.
- **Alertas** funcionando com dados reais:
  - *"1 apuração(ões) ainda aberta(s) numa competência que já tem documentos escriturados."*
  - *"1 tributo(s) com valor a recolher e guia ainda não gerada."*
  - guia vencida / vencendo em ≤ 5 dias.
- Rota estática registrada **antes** da paramétrica `/apuracoes/:id` (a regra do MSW que já nos mordeu 3 vezes).

Aproveitei para promover o mapa de rótulos de tributo a constante única (`TRIBUTO_LABEL`), que estava duplicado dentro do handler do Portal — mesma família do problema deste lote, em escala menor.

---

## 4. Checkpoint de igualdade — provado por endpoint

**9 apurações · 2 empresas · 8 tributos distintos · ZERO divergentes.**

| Empresa | Tributo | Comp. | Painel | Tela do tributo | Portal do Cliente |
|---|---|---|---|---|---|
| Iguassu | ICMS | 2026-07 | **961,29** | 961,29 | 961,29 |
| Metalúrgica | ICMS | 2026-04 | **580,00** | 580,00 | 580,00 |
| Metalúrgica | ISS | 2026-04 | **400,00** | 400,00 | 400,00 |
| Metalúrgica | PIS | 2026-04 | **99,00** | 99,00 | 99,00 |
| Metalúrgica | COFINS | 2026-04 | **456,00** | 456,00 | 456,00 |
| Metalúrgica | IPI | 2026-04 | **600,00** | 600,00 | 600,00 |
| Metalúrgica | IBS | 2026-04 | **636,00** | 636,00 | 636,00 |
| Metalúrgica | CBS | 2026-04 | **424,00** | 424,00 | 424,00 |
| Metalúrgica | Retenções Federais | 2026-04 | **1.715,00** | 1.715,00 | 1.715,00 |

**Antes deste lote, a coluna "Portal do Cliente" da primeira linha era R$ 0,00.**

E o quarto ponto, o Monitor de Saúde Fiscal: **"R$ 961,29 a recolher"** — batendo com os outros três.

---

## 5. Regra escrita no CLAUDE.md

Como você determinou, a quarta da família, com as três ocorrências citadas:

> **Número derivado tem UMA fonte: quem exibe chama a função de derivação, nunca lê o campo materializado cru** — […] Campo materializado no fixture/banco é cache, não verdade: ele envelhece em silêncio, e a divergência só aparece quando alguém compara dois lados — normalmente o cliente. **É a família de bug mais recorrente deste projeto** (ocorrências: prévia × efetivo no F1-B1, conta resolvida por código sem escopo no item 43, Portal do Cliente × telas de tributo na Fase 3 […]). Ao criar um endpoint que devolve número calculado, a pergunta obrigatória é: *existe função de derivação para isto?* Se existe, use-a; se não existe, crie-a antes — nunca duplique a conta.

---

## 6. Regra inegociável — confirmada

`motor-apuracao.ts` **não foi tocado**. `derivarApuracao`, `apurarDecomposta` e `CONFIG_TRIBUTOS` estão byte a byte como estavam. As três correções trocaram **de onde o número vem**, nunca **como ele é calculado**.

`tsc --noEmit` limpo · `next lint` sem erros nem warnings.

---

## 7. Próximo — F3-B2 (não iniciado)

Painel Central de Apurações na tela: cards, total, alertas, filtros/chips, 6 temas, reorganização do menu (D-ap-2, mantendo as rotas atuais vivas) e doc in-app.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
