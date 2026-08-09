# RELATÓRIO — Fase 2 · Memória Contábil

**Branch:** `feat/memoria-contabil` · sobre os itens 43 e 44 já mergeados.

---

## 1. O que foi entregue

`MemoriaContabilizacao` — entidade própria, escopo empresa. Chave: **participante + produto/serviço + destinação + finalidade** → contas D/C, centros de custo e rateio da última contabilização confirmada por humano. Irmã da `MemoriaClassificacao` (que aprende destinação/finalidade), com a mesma chave de identidade e a **mesma curva de confiança 60/75/90**.

**A memória nasce vazia de propósito.** Semear entradas seria fabricar um histórico que ninguém viveu, e a confiança exibida mentiria sobre quantas vezes aquilo de fato aconteceu.

### Precedência — implementada e provada

```
manual do documento  >  Regra da Central (explícita)  >  memória (aprendida)
```

Sua lista dizia "Regra → Memória → manual". Implementei nessa ordem para as fontes **automáticas**, e coloquei o **manual acima das duas** — porque é o que "memória nunca sobrescreve o que o humano tocou" exige. As duas afirmações só são compatíveis assim: o manual entra tanto como último recurso (quando nada automático decide) quanto como palavra final (quando o humano decidiu explicitamente).

Cada linha da partida carrega `fonteConta` (`regra` | `memoria` | `manual` | `retencao`) e, quando vem da memória, a confiança. **A origem nunca é implícita.**

### Onde a memória de fato atua

A regra fiscal precede a memória, então a memória entra onde a regra **não decidiu** — tipicamente as **regras auto-criadas**, que nascem sem contabilização e até agora só produziam a pendência "contabilização incompleta". É exatamente o buraco que ela fecha.

---

## 2. Checkpoint — os três cenários que você pediu

### C1 · Segunda escrituração da mesma combinação chega com contabilização proposta pela memória

```
1ª escrituração (integdoc-3) — a REGRA decide (D 1.1.3 · C 2.1.1)
   → memória aprende:  FORN-4410 · 7606.12.90 · Industrialização/Produção
                       D 1.1.3 · C 2.1.1 · ocorrências 1 · confiança 60

2ª escrituração (integdoc-6) — mesma combinação, regra RP-MEM-F2 SEM contabilização
   → memoriasUsadas: [{ confianca: 75, ocorrencias: 2 }]
   → lançamento NASCE:  D 1.1.3 "Estoques"            10.000,00
                        C 2.1.1 "Fornecedores a Pagar" 10.000,00
   → memória: ocorrências 2 · confiança 60 → 75
```

O segundo item do mesmo documento (combinação diferente, sem histórico) virou pendência com a mensagem certa: *"contabilização incompleta … **e sem histórico para esta combinação**"*. A memória não inventa quando não sabe.

### C2 · Ajuste manual atualiza a memória e dispara a pergunta

```
PATCH /v1/fiscal/conecta/integdoc-6/contabilizacao  { D 5.1.4 · C 2.1.1 }
  → 200 · contabilização fixada neste documento
  → memória atualizada: D 5.1.4 · ocorrências 3
  → promocao: {
       disponivel: true, regraCodigo: "RP-MEM-F2", jaTemContabilizacao: false,
       pergunta: "Criar a contabilização na regra RP-MEM-F2 para os próximos documentos?"
     }
```

O texto da pergunta muda conforme o caso — *criar* quando a regra não tem contabilização, *atualizar* quando já tem. O sistema não finge que os dois atos são a mesma coisa.

**O ajuste manual não é atalho para burlar o item 43:**
```
PATCH … { contaDebito: "2.1.3" }  → 422
  "Conta 2.1.3 é sintética (agrupadora) e não é lançável — indique uma conta analítica."
```

### C3 · Promoção a regra cria na Central, com auditoria

```
POST /v1/fiscal/memoria-contabil/<id>/promover-regra  { regraId }
  → 200 · regra RP-MEM-F2 agora tem D 5.1.4 · C 2.1.1 · sem avisos (contas válidas)
  → memória guarda promovidaParaRegraId

Auditoria:
  regra_fiscal.contabilizacao_promovida_da_memoria   (com memoriaId e ocorrenciasAprendidas)
  contabilizacao.ajustada_manualmente                (com as contas e as memórias afetadas)
```

O evento de promoção registra **quantas ocorrências** a memória tinha — dá para saber depois se a regra nasceu de um caso isolado ou de um padrão repetido.

---

## 3. O pré-requisito do item 43, honrado

`aprenderContabilizacao` só é chamada **depois** que o lançamento nasceu de fato — o que, com a validação do item 43 no caminho, garante que todas as contas são válidas no plano daquela empresa. E a guarda é explícita no início da função:

```ts
if (montagem.contasInvalidas.length > 0 || !montagem.balanceado) return;
```

Duas travas, não uma: não aprende de conta inválida, e não aprende de partida que não fecha.

---

## 4. Enriquecimento aditivo

- A memória **só entra onde a regra não decidiu**. Regra preenchida nunca é sobrescrita.
- O ajuste manual do documento **vence memória e regra**, e persiste: reescriturar não desfaz.
- Aprender é sempre **acréscimo** — `ocorrencias + 1` e atualização do que foi visto por último; nada é apagado.
- A memória de uma empresa **nunca alcança outra** (escopo `empresaId` na chave de busca, no mesmo espírito do item 43).

---

## 5. Superfícies e documentação

- `GET /v1/fiscal/memoria-contabil?empresaId=` — o que a máquina aprendeu, com a confiança calculada.
- Detalhe do documento devolve `memoriasUsadas` e `contabilizacaoManual` — a aba Contabilidade mostra de onde veio cada conta.
- Ações internas: ajuste e promoção negados no portal (**403**) e sob o RBAC de `lancamentos:lancar` / `central-regras-fiscais:editar`.

**Docs in-app** (mesmo ciclo): conceitos **"Memória contábil (o que a máquina aprendeu)"** — com a ordem de precedência em português e a curva de confiança explicada — e **"Memória vira regra pelo uso"** no Diário; passo novo na ação de escriturar do Conecta, avisando para conferir quando a conta vier da memória.

---

## 6. Estado e o que fica em aberto

`feat/memoria-contabil` pronta para sua validação. `tsc --noEmit` limpo · `next lint` sem erros nem warnings.

**Duas observações honestas sobre limites do que entreguei:**

1. **A prévia só mostra a proposta da memória depois que uma regra fiscal foi aplicada ao item** — porque o motor exige `regraTributariaId` antes de montar qualquer partida. Na prática isso significa que a proposta aparece no momento da escrituração, não antes dela. Funciona, mas não é o ideal: o operador veria mais cedo se a memória pudesse falar já no Check-in. Registro como refinamento, não como bug.
2. **A UI do ajuste manual não foi construída** — os endpoints existem, estão sob RBAC e provados, e a aba Contabilidade já expõe a origem e a memória usada; falta o formulário de edição inline com o botão "gravar na regra". É o próximo lote natural desta fase, se você quiser fechá-la visualmente antes da Fase 3.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
