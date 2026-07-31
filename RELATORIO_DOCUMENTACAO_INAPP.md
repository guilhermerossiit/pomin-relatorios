# Relatório — Documentação Funcional In-App

**Ciclo:** Documentação funcional in-app (Etapa A — fundação + piloto)
**Branch:** `feat/documentacao-inapp`
**Commits:** 6 · **Arquivos:** 15 (+1689 / −24)
**Status:** fechado, aguardando validação do padrão e merge

---

## 1. Objetivo do ciclo

Criar a fundação de um sistema de documentação funcional acessível de dentro da aplicação: um
acesso consistente em todas as telas, um painel que explica a tela em que o usuário está, e um
formato de conteúdo que permita documentar o sistema tela a tela sem reescrever componente.

O ciclo entrega a **fundação** e uma tela **piloto** como padrão de qualidade. O conteúdo por grupo
de telas é o ciclo seguinte.

---

## 2. Arquitetura

### 2.1 Conteúdo é dado, não markup

Cada tela documentada é um arquivo em `apps/web/src/lib/docs/telas/` que exporta um
`DocumentacaoTela` (tipo em `types/documentacao.ts`). O `DocPanel` renderiza **qualquer** definição
sem conhecer a tela.

Consequência prática: documentar a próxima tela é escrever um arquivo e registrá-lo — nunca tocar
no componente. Evoluir o texto de uma tela não é mudança de UI.

### 2.2 Binding por `telaId`

O identificador é a rota sem a barra inicial (`/gestor-fiscal/monitores/conecta` →
`gestor-fiscal/monitores/conecta`) — **o mesmo do Guided Process Engine**. Foi escolha deliberada:
é o que permite a seção "Simulação interativa" disparar o processo guiado na tela real sem nenhuma
camada de tradução entre os dois sistemas.

### 2.3 Arquivos

| Arquivo | Papel |
| --- | --- |
| `types/documentacao.ts` | Modelo do conteúdo (7 seções, sinônimos por verbete) |
| `lib/docs/registry.ts` | Registro por `telaId` + índice de busca derivado |
| `lib/docs/busca.ts` | Motor de busca aproximada |
| `lib/docs/telas/*.tsx` | Conteúdo por tela (dado) |
| `components/docs/doc-panel.tsx` | Painel — renderiza qualquer definição |
| `components/docs/doc-busca.tsx` | Barra de busca, resultados e realce |
| `components/docs/doc-button.tsx` | Acesso no shell |

---

## 3. Acesso

O botão substituiu o `HelpCenterButton`, que era um placeholder desabilitado ("em breve") na barra
superior. Como vive no shell, aparece nas **178 telas** do registry de módulos na mesma posição, sem
que nenhuma tela precise montá-lo — e a próxima tela do sistema já nasce com ele.

| Estado | Aparência (medida) | Comportamento |
| --- | --- | --- |
| Tela **com** documentação | ícone pleno, `rgb(124,58,237)`, opacidade 1 | abre o conteúdo da tela |
| Tela **sem** documentação | esmaecido, opacidade 0.5 | abre explicando que está em construção — e com a busca global funcionando |

O botão **nunca some** conforme a tela. Um acesso que aparece e desaparece ensina o usuário a não
procurar; esmaecido ainda é um convite.

---

## 4. Painel

`Sheet` à direita com `size="workstation"`: ocupa a largura útil total, encostando na divisa exata
do sidebar via `--pomin-sidebar-w`, e acompanha o menu recolhido (72px), expandido (264px) ou
ausente. Medido: `left: 264px`, largura `1016px` em viewport de 1280.

**Layout interno:** índice lateral fixo (≥ `lg`) com as seções renderizadas — clicar navega **e abre
a seção se estiver fechada** — e as seções em grid (`xl:grid-cols-2`), com a Visão Geral ocupando as
duas colunas.

### Anatomia do conteúdo (ordem canônica)

1. **O que é esta tela** — 2–3 frases de propósito
2. **Visão geral** — ilustração esquemática com callouts numerados
3. **Ações disponíveis** — pré-condições, passo a passo, resultado esperado
4. **Conceitos** — termos da tela, com "onde aparece"
5. **Simulação interativa** — dispara o Guided Process Engine na tela real
6. **Perguntas frequentes**
7. **Telas relacionadas** — deep-links reais

A ilustração é **componente SVG com tokens**, nunca screenshot: acompanha os 6 temas e não
envelhece em silêncio a cada ajuste de layout.

---

## 5. Conteúdo entregue

| | Integração Conecta (piloto) | Pedido de Compra |
| --- | --- | --- |
| Ações | 4 | 3 |
| Conceitos | 6 | 4 |
| Perguntas frequentes | 5 | 3 |
| Telas relacionadas | 4 | 2 |
| Verbetes com sinônimos | 12 | 11 |
| Ilustração de anatomia | ✓ (5 callouts) | — |
| Simulação (GPE) | ✓ (5 etapas) | — |

A **definição de processo guiado do Conecta não existia** e foi criada neste ciclo (check-in →
vínculo do pedido → financeiro → escrituração → integração). Sem ela, "Ver na prática" seria um
botão morto.

O Pedido de Compra foi documentado por necessidade estrutural: com uma única tela, a busca global
não teria o que agrupar e o recurso não se provaria.

---

## 6. Busca aproximada

### 6.1 Escolha de implementação

Motor próprio em `lib/docs/busca.ts` (~240 linhas), **sem dependência nova**.

O que este caso exige — dobra de acentos do PT-BR, match por radical, sinônimos declarados pelo
autor do conteúdo e "você quis dizer" — não vem pronto em biblioteca alguma. O fuse.js trataria
"escrituraçao" e "escrituração" como palavras diferentes, então a normalização viria por cima de
qualquer forma; o mesmo para vocabulário declarado e sugestão. O que sobraria da lib é o
Levenshtein, que são ~25 linhas com poda. A alternativa era trocar 4 camadas próprias por 4 camadas
próprias **mais** uma dependência.

`cmdk`, já presente no projeto, não expõe busca genérica — apenas o filtro interno do command
palette.

### 6.2 Capacidades

- Tolerância a erro de digitação e acentuação, com limiar proporcional ao tamanho da palavra
- Match por radical (`rejeit` → rejeição, rejeitadas)
- **Sinônimos declarados na própria definição** — o autor enriquece o vocabulário no arquivo da tela
- **Dois escopos** com alternância sempre visível: `Nesta tela` (padrão) e `Toda a documentação`,
  com resultados agrupados por tela e deep-link para a tela real
- Vazio honesto com "você quis dizer"
- Índice **derivado do registry** — documentar uma tela nova a torna pesquisável sem passo manual

### 6.3 Validação

| Caso | Consulta | Resultado |
| --- | --- | --- |
| Erro de digitação | `diverjencia` | 7 resultados |
| Acentuação | `escrituraçao` | 10 resultados |
| Radical | `rejeit` | acha "Tratar uma falha de integração" |
| Sinônimo declarado | `PO` | 14 locais, incluindo o FAQ de pedido de compra |
| Escopo global | `PO` | 27, agrupados em *Pedido de Compra* e *Integração Conecta* |
| Navegação global | `extrato de consumo` | abre a doc do Pedido de Compra na seção certa |
| Vazio + sugestão | `conciliassaum` | "Você quis dizer **conciliacao**?" |

### 6.4 Realce

Amarelo (token `--pomin-warning`), a convenção de marca-texto — o accent do tema fazia o realce
mudar de cor a cada tema e se confundir com os elementos de identidade da própria tela.

| Tema | Fundo | Texto |
| --- | --- | --- |
| 3 Light | âmbar `#f59e0b` a 45% | roxo / verde / azul |
| 3 Dark | âmbar `#fbbf24` a 45% | neutro claro |

O realce é **aproximado** (mesmo critério da busca) e **persiste** no conteúdo após abrir um
resultado, até a próxima busca.

---

## 7. Defeitos encontrados e corrigidos no ciclo

Quatro problemas foram introduzidos por mim e achados **testando**, não lendo código:

| # | Defeito | Causa | Correção |
| --- | --- | --- | --- |
| 1 | Conteúdo sumia ao expandir uma seção | colunas CSS transbordam na horizontal; o `overflow-hidden` do Sheet cortava | grid, com cada seção como item próprio |
| 2 | Ilustração ilegível | wireframe abstrato renderizado em coluna estreita, texto ~7px | rótulos reais + largura de duas colunas |
| 3 | "Você quis dizer" nunca dispararia | limiar da sugestão igual ao da busca — o que fosse sugerível já teria sido encontrado | tolerância da sugestão maior de propósito |
| 4 | Realce literal numa busca aproximada | quem digitava `diverjencia` achava "Divergência" mas não via nada marcado | realce passa a usar o mesmo critério da busca |

O *porquê* de cada um está registrado no código e no `DESIGN_SYSTEM.md`, para não serem
"simplificados" de volta.

---

## 8. Pontos em aberto

### 8.1 Esc não fecha o painel — não confirmado

Testado com tecla real e com evento sintético: não fecha. O modal de **Check-in**, anterior a este
ciclo, comporta-se **exatamente igual**. O `Sheet` usa Radix Dialog 1.1.2 sem override, versão em
que Esc fecha por padrão.

Duas hipóteses: limitação do teclado automatizado do ambiente de teste, ou comportamento do `Sheet`
compartilhado — e nesse caso afeta **todos os modais do sistema**, não este painel.

> **Ação:** verificação humana de um segundo — abrir o painel e apertar Esc. Se não fechar, é ciclo
> próprio no componente compartilhado.

### 8.2 Nenhuma validação visual

Pelo mesmo motivo do item anterior, o navegador automatizado não avança a animação de entrada e o
painel fica fora da viewport na medição (o Check-in também). **Todas as afirmações visuais deste
relatório vêm de CSS calculado e inspeção de DOM, não de observação da tela renderizada.**

### 8.3 Tokens semânticos sem opacidade (BACKLOG 33)

`bg-pomin-warning/40` não gerava CSS nenhum. Os tokens `warning`, `danger`, `success`, `info` e
`neutral` estão em `tailwind.config.ts` como `var(--pomin-*)` puro, sem o placeholder
`<alpha-value>` que `accent` e `primary-*` têm — todo modificador de opacidade neles é ignorado em
silêncio.

São **~34 usos só de `bg-pomin-warning/`** no código (faixas de alerta, chips de divergência, radar
de rejeições). Nada quebra layout; tudo está sem o tom pretendido.

Não corrigido de passagem (alcance grande em config compartilhada). Resolvido apenas no caso deste
ciclo, com `color-mix` sobre o token. Causa raiz e correção proposta registradas no BACKLOG.

---

## 9. Padrão registrado

- **`DESIGN_SYSTEM.md §4g`** — anatomia do DocPanel, estados do botão, layout interno e busca
- **`CLAUDE.md`** — regra obrigatória: *toda tela nova nasce com definição de documentação em
  `lib/docs/telas/`, com no mínimo o que é, ações e conceitos*

---

## 10. Estado

| Verificação | Resultado |
| --- | --- |
| `tsc --noEmit` | limpo |
| `next lint` | limpo |
| Console do navegador | sem erros |
| 6 temas | conferidos (botão, painel, ilustração, realce) |

**Cobertura: 2 de 178 telas.** É o esperado de uma fundação com piloto — e é o número que o próximo
ciclo precisa mover, documentando por grupo.

---

## 11. Commits

| Hash | Descrição |
| --- | --- |
| `107fea3` | fundação e piloto Conecta |
| `85d3940` | DocPanel ocupa a largura útil até a divisa do sidebar |
| `bf02603` | índice lateral e correção do conteúdo que sumia |
| `aaef45a` | ilustração de anatomia legível |
| `e049f39` | busca aproximada com escopo duplo e sinônimos |
| `6a5a993` | realce amarelo, aproximado e persistente |
