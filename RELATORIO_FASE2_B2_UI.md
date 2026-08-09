# RELATÓRIO — Fase 2 · F2-B2 (fechamento visual)

**Branch:** `feat/memoria-contabil` · lote final da Fase 2.

---

## 1. Os quatro itens

### 1.1 UI do ajuste manual na aba Contabilidade

Editor inline (`contabilizacao-manual-editor.tsx`) com contas D/C e rateio por centro de custo.

**A validação do item 43 acontece no ato, em dois lugares:**
- o seletor só oferece **contas analíticas ativas do plano DAQUELA empresa** — sintética e inativa nem aparecem;
- o backend recusa de novo, com o motivo (`422`), porque a UI esconder é conveniência, não proteção.

O rateio só permite salvar fechando em 100%, com a soma visível enquanto se digita.

### 1.2 Botão "gravar na regra" com o texto correto por caso

Salvo o ajuste, aparece a pergunta devolvida pelo backend — e ela **muda conforme o caso**:

| Situação da regra | Pergunta | Botão |
|---|---|---|
| Sem contabilização | "**Criar** a contabilização na regra RP-MEM-F2 para os próximos documentos?" | Gravar na regra |
| Já tem contabilização | "**Atualizar** a contabilização da regra … ?" | Atualizar a regra |

Criar e atualizar não são o mesmo ato, e a interface não finge que são. Há "Agora não" — a promoção é oferta, não imposição. Confirmada, o resultado aparece: *"Regra X atualizada — os próximos documentos desta combinação já nascem contabilizados por ela."*

### 1.3 Origem por linha, com confiança

Coluna **Origem** na prévia da aba Contabilidade:

```
Regra  ·  Memória 75%  ·  Manual  ·  Retenção
```

Provado no payload:
```
GET /conecta/integdoc-3/detalhe
  → previaLancamentos: [ "1.1.3 · fonte=regra", "2.1.1 · fonte=regra" ]
```

A confiança aparece **ao lado** da origem quando vem da memória — 60% é palpite novo, 90% é padrão consolidado. O operador vê de onde veio **antes** de confirmar, que era exatamente o ponto.

### 1.4 Tela da Memória Contábil

Rota `/contabilidade/memoria-contabil`, registrada no menu. Verificada renderizando:

```
Filtros ativos: | Empresa: Metalúrgica Pomin | Limpar filtros | 1 de 1 combinações
COMBINAÇÃO APRENDIDA | CONTABILIZAÇÃO | CONFIANÇA | ÚLTIMA VEZ | REGRA
FORN-4410 · 7606.12.90       D 1.1.3        60%              09/08/2026   [Promover a regra]
Industrialização / Produção  C 2.1.1        1× confirmada
                             Produção 100%
```

Com painel de filtros aplicados padrão (empresa como chip, mesmo sendo default não-neutro), filtro "só as não promovidas", e a ação de **promover a regra dali** — escolhendo em qual regra gravar. Promovida, a linha passa a exibir o código da regra que absorveu o aprendizado.

**Estado vazio com texto útil**, não um "sem dados": explica que a memória se preenche sozinha a partir da primeira escrituração confirmada e passa a propor da segunda em diante. Uma tela vazia que não se explica parece defeito.

---

## 2. Documentação in-app

**Nova** — `contabilidade/memoria-contabil`: 2 ações (conferir o que foi aprendido; promover a regra) e 4 conceitos, entre eles **"Precedência: manual > regra > memória"** com o argumento em português, a **curva 60/75/90** e **"a memória só aprende do que é válido"**, explicando por que a reconciliação do plano de contas veio antes desta funcionalidade.

**Atualizadas** — Diário (conceitos "Memória contábil" e "Memória vira regra pelo uso") e Conecta (passo novo: conferir quando a conta vier da memória).

---

## 3. Registros pedidos

**CLAUDE.md** ganhou a precedência como regra de projeto, com o argumento que você validou:

> **Precedência das fontes de decisão automática: MANUAL > REGRA > MEMÓRIA** — […] O manual é ao mesmo tempo **último recurso** (quando nenhuma fonte automática decide) e **palavra final** (quando o humano decidiu explicitamente) — é isso que sustenta o enriquecimento aditivo: *automação nunca sobrescreve o que o humano tocou*.

Generalizei para toda sugestão aprendida (contabilização, classificação, e o que vier depois), não só para a Fase 2.

**BACKLOG item 46** — proposta antecipada no Check-in, prioridade BAIXA, com a causa técnica registrada: `montarPartidasDocumento` exige `regraTributariaId` antes de montar qualquer partida, então a memória só fala no momento da escrituração. Não é bug; é refinamento de **momento da fala**.

---

## 4. Estado

`tsc --noEmit` limpo · `next lint` sem erros nem warnings. Fase 2 fechada — modelo, motor, endpoints e interface.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
