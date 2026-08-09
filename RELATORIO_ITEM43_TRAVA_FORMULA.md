# TRAVA DE FÓRMULA — Item 43 · Reconciliação Conta da Regra × Plano de Contas

**Ciclo curto 1** · branch `fix/reconciliacao-plano-contas` · **Etapa 1: mecânica proposta. Nenhuma linha de código escrita.**

---

## 0. Antes da proposta: o que a auditoria encontrou

O item 43 foi registrado como "conta fantasma que só aparece no balancete". A medição mostrou **três** problemas, e o mais grave não é a conta fantasma.

### Evidência — um lançamento gerado pela ponte, auditado partida a partida

`POST /conecta/integdoc-4/escriturar` → `lanc-…a8plri` · empresa **`empresa-auditto-cliente1`** (tenant `org-auditto`)

| # | Tipo | Código | Nome gravado | `contaContabilId` gravado | Diagnóstico |
|---|---|---|---|---|---|
| 1 | D | `4.02.05` | `4.02.05` | `4.02.05` | **conta fantasma** — não existe em plano nenhum |
| 2 | C | `2.1.3.1` | IRRF a Recolher | `conta-2-1-3-1` | **conta de OUTRA empresa** (`empresa-iguassu`) |
| 3 | C | `2.1.3.2` | CSLL a Recolher | `conta-2-1-3-2` | idem |
| 4 | C | `2.1.3.3` | PIS a Recolher | `conta-2-1-3-3` | idem |
| 5 | C | `2.1.3.4` | COFINS a Recolher | `conta-2-1-3-4` | idem |
| 6 | C | `2.1.3.5` | INSS a Recolher | `conta-2-1-3-5` | idem |
| 7 | C | `2.1.3.6` | ISS a Recolher | `conta-2-1-3-6` | idem |
| 8 | C | `2.01.01` | `2.01.01` | `2.01.01` | **conta fantasma** |

**8 de 8 partidas problemáticas.** E a empresa do lançamento **não tem plano de contas cadastrado**.

### Causa raiz

```ts
function resolverContaContabil(codigo: string) {
  const c = CONTAS_CONTABEIS_MOCK.find((x) => x.codigo === codigo);   // ← sem clienteId
  return c ? { id: c.id, nome: c.nome } : undefined;
}
```

O plano de contas é **por empresa** (`ContaContabil.clienteId`), mas a resolução procura o código no acervo inteiro e devolve a primeira que casar — de qualquer empresa, de qualquer tenant.

**Isto é uma violação da regra invariável de multi-tenant, e é responsabilidade minha: entrou no F1-B1, que você validou e está em produção.** Não estava no escopo do item 43 como registrado — apareceu ao medir. Não é um defeito de exibição: o `contaContabilId` de outra empresa fica **gravado** no lançamento.

### Situação do cadastro hoje

| Fato medido | Valor |
|---|---|
| Empresas com plano de contas | **1** (`empresa-iguassu`, 32 contas: 12 sintéticas, 20 analíticas) |
| Empresas sem plano nenhum | todas as demais |
| Códigos usados pelas 4 regras | `1.01.03`, `2.01.01`, `4.01.01`, `4.02.05` |
| Desses, quantos existem no plano | **0 de 4** |
| Notação do plano × notação das regras | `1.1.1` / `2.1.3.1` **×** `1.01.03` / `4.02.05` — duas convenções diferentes |
| Partidas dos 4 lançamentos de fixture fora do plano | **0** (foram escritos à mão com códigos coerentes) |

Ou seja: o passivo real não está no que já existe — está no que a ponte **vai** gerar. Por isso a correção vale mais que a limpeza, e por isso este ciclo vem antes da Fase 2.

---

## 1. A mecânica proposta

Princípio: **é melhor não contabilizar do que contabilizar errado.** Não contabilizar já tem tratamento maduro — a pendência derivada que você validou no F1-B3, com motivo, severidade e deep-link. Contabilizar errado produz um registro que entra no balancete e no SPED, e que depois só se corrige por estorno.

### R-1 · Resolução da conta passa a ser POR EMPRESA — **não vai à trava**

`resolverConta(codigo, clienteId)`. Uma conta só é encontrada no plano da própria empresa.

Não peço aprovação para isto: é a regra invariável de multi-tenant, e vou corrigir de qualquer forma. Está aqui para você saber que muda — depois dela, os 6 créditos de retenção do exemplo deixam de resolver, porque aquelas contas pertencem a outra empresa.

### R-2 · Conta inexistente no plano da empresa → **não gera o lançamento; vira pendência**

Das três opções que você levantou:

| Opção | Veredito |
|---|---|
| **Bloqueia a contabilização com pendência** | ✅ **proposta** |
| Gera com aviso | ❌ o aviso morre na tela; o lançamento errado fica no Diário, no balancete e no SPED |
| Sugere a conta mais próxima | ❌ ver R-2b |

Custo de bloquear: **zero para o fiscal**. A escrituração segue (D-1 intacta), o documento fica escriturado, e a falta aparece na inbox — mesma família de `sem_contabilizacao`, só um motivo novo. Nada de mecânica nova a inventar.

**R-2b · Por que não sugerir a conta mais próxima.** Proximidade de código não é proximidade de significado. `4.02.05` "mais parecido" com `4.2.5` é palpite sobre o plano de contas de um cliente que eu não conheço — e um palpite que vira lançamento é pior que uma pendência. Isso é julgamento humano: *a máquina preenche fatos, o humano preenche julgamento.*

**O meio-termo que proponho:** a pendência e a tela de regra oferecem o **catálogo de contas analíticas ativas daquela empresa** para o contador escolher, **sem pré-selecionar nenhuma**. Oferece a lista, não a decisão.

### R-3 · Conta **sintética** usada onde deveria ser analítica → **mesma pendência, motivo próprio**

Sintética existe para totalizar; não é lançável. Lançar nela quebra o balancete — o total do grupo passa a divergir da soma das filhas.

Motivo exibido: *"A conta 2.1.3 é sintética (agrupadora). Indique uma conta analítica."*

**Sub-caso que decidi NÃO automatizar:** sintética com exatamente uma filha analítica é tentador resolver sozinho. Não proponho. Hoje tem uma filha; amanhã tem três — e o lançamento antigo teria ido para uma conta escolhida por acidente da época.

### R-4 · Conta **inativa** → mesma pendência

Inativar uma conta é ato deliberado do contador. Respeitar o ato é o mínimo.

### R-5 · Empresa **sem plano de contas** → pendência com motivo próprio

Não é erro da regra — é cadastro da empresa que não existe. Se o motivo não disser isso, o contador vai procurar defeito na regra. Deep-link para o Plano de Contas da empresa, que já tem "aplicar template".

### R-6 · Central de Regras: **avisa no ato, não bloqueia**

Ao salvar a regra, validar os códigos contra o plano de **cada empresa no escopo dela**.

Não bloquear, porque uma regra vale para várias empresas e pode ser legítima em umas e não em outras — e a empresa que falta pode ser cadastrada depois. O que proponho:

- salva normalmente;
- mostra **"Esta regra não vai contabilizar em N empresas"**, com a lista e o motivo por empresa;
- marca a regra como `statusRevisao: 'revisar'` **quando nenhuma** empresa do escopo reconhece as contas — reusando o sinal que já existe no modelo para "isto precisa de olho humano".

### R-7 · Lançamentos já existentes: **não mexer; auditar e expor**

Acabamos de documentar que lançamento efetivado é imutável. Corrigir retroativamente por edição seria contradizer o princípio no primeiro teste.

- **Relatório de auditoria** com o número real de partidas apontando conta fora do plano (endpoint + tela).
- Os gerados pela ponte, em competência aberta, se corrigem pelo caminho que já existe: **estorno + reescrituração**.
- Os manuais ficam como estão, sinalizados — quem lançou decide.

Número real de hoje: **0 partidas problemáticas** nos 4 lançamentos de fixture; **8 de 8** no primeiro lançamento gerado pela ponte.

### R-8 · A notação divergente (`1.01.03` × `1.1.3`) — **não normalizar**

Zeros à esquerda podem ser significativos: num plano com nível de 2 dígitos, `1.01` e `1.1` são contas diferentes. Normalizar automaticamente é adivinhar a convenção do cliente e, na hipótese errada, mandar valor para a conta errada — silenciosamente.

Proposta: a validação **acusa**, e as fixtures de regra passam a usar códigos que existem no plano. Se você quiser normalização, que seja decisão explícita sua, nunca um efeito colateral.

---

## 2. Consequência que você precisa pesar antes de aprovar

Aprovada a R-2, **a quantidade de pendências vai subir e a de lançamentos gerados vai cair** — no cadastro de hoje, para perto de zero, porque nenhum código de regra existe em nenhum plano.

Isso é o remédio funcionando, não efeito colateral: hoje esses lançamentos nascem errados. Mas significa que a implementação precisa incluir **acerto de dados de demonstração** — dar plano de contas às demais empresas (o "aplicar template" já existe) e alinhar os códigos das 4 regras ao plano real. Sem isso, a demo fica sem lançamento nenhum.

Está no escopo da etapa 2 que você já aprovou; registro aqui para não haver surpresa.

---

## 3. O que decido sozinho e o que espera seu OK

**Não vai à trava** (regra invariável / princípio já aprovado):
- R-1 resolução por empresa — multi-tenant;
- reuso da pendência `sem_contabilizacao` em vez de entidade nova;
- fonte única: a validação entra em `montarPartidasDocumento`, valendo para prévia e efetivo no mesmo ato.

**Espera seu OK** (mecânica contábil):
- **R-2** bloquear e virar pendência, em vez de gerar com aviso;
- **R-2b** não sugerir conta por proximidade — oferecer o catálogo, não a decisão;
- **R-3** sintética tratada como inválida, sem auto-resolver a filha única;
- **R-4** inativa tratada como inválida;
- **R-6** avisar sem bloquear no cadastro da regra, com `revisar` quando nenhuma empresa reconhece;
- **R-7** não corrigir lançamento existente — auditar e expor;
- **R-8** não normalizar notação.

---

## 4. Checkpoint prometido para a etapa 3

- regra com conta inexistente → comportamento aprovado, **provado por endpoint**;
- conta sintética e conta inativa → cada uma com seu motivo, provadas;
- prova de tenant: conta de outra empresa **não** resolve mais;
- auditoria dos lançamentos existentes com **número real**;
- pendências novas somem pelo caminho normal (corrigir a regra → reescriturar).

*Documento de especificação técnica. Não contém credenciais nem dados de clientes reais.*
