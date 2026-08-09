# RELATÓRIO — Item 43 · Etapa 2 (implementação + checkpoint)

**Ciclo curto 1** · branch `fix/reconciliacao-plano-contas` · trava R-1..R-8 aprovada integralmente.

---

## 1. A correção de tenant — feita primeiro, como determinado

### Antes

```ts
function resolverContaContabil(codigo: string) {
  const c = CONTAS_CONTABEIS_MOCK.find((x) => x.codigo === codigo);   // acervo global
  return c ? { id: c.id, nome: c.nome } : undefined;
}
```

### Depois

```ts
function resolverContaContabil(codigo: string, clienteId: string): ResolucaoConta {
  const planoDaEmpresa = CONTAS_CONTABEIS_MOCK.filter((x) => x.clienteId === clienteId);
  if (planoDaEmpresa.length === 0) return { ok: false, motivo: 'sem_plano' };
  const c = planoDaEmpresa.find((x) => x.codigo === codigo);
  if (!c)          return { ok: false, motivo: 'inexistente' };
  if (c.sintetica) return { ok: false, motivo: 'sintetica' };
  if (!c.ativa)    return { ok: false, motivo: 'inativa' };
  return { ok: true, id: c.id, nome: c.nome };
}
```

O `clienteId` entrou na assinatura de `montarPartidasDocumento` também — **o motor sempre passa o escopo**, então nenhum chamador pode esquecê-lo. A regra virou estrutura, não disciplina.

Removido junto: o `contaId` fixo que `CONTA_POR_TRIBUTO_RETIDO` carregava (`conta-2-1-3-1`…). Era ele que injetava id da Iguassu em qualquer empresa. Agora o mapa só tem **código e rótulo**; o id vem sempre da resolução com escopo.

### Prova de tenant — o caso exato do relatório anterior

`POST /conecta/integdoc-4/escriturar` · empresa **`empresa-auditto-cliente1`** (tenant `org-auditto`)

| Antes | Depois |
|---|---|
| `id=conta-2-1-3-1` (Iguassu) | **`id=conta-auditto-cliente1-2-1-3-1`** |
| `id=conta-2-1-3-2` … `-6` (Iguassu) | `conta-auditto-cliente1-2-1-3-2` … `-6` |
| `4.02.05` → id inventado, sem nome | `5.1.3` → `conta-auditto-cliente1-5-1-3` · "Serviços de Terceiros" |
| `2.01.01` → id inventado, sem nome | `2.1.1` → `conta-auditto-cliente1-2-1-1` · "Fornecedores a Pagar" |

**8 de 8 partidas** agora com conta da própria empresa, nome real, ΣD = ΣC = 5.300,00.
Conta de outra empresa **não resolve mais** — a busca nem chega a enxergá-la.

---

## 2. O número que você pediu — lançamentos existentes com conta de outra empresa

Endpoint novo `GET /v1/contabilidade/lancamentos/auditoria-contas`:

```
lancamentosAuditados: 4      partidasAuditadas: 8
partidasComProblema: 0       lancamentosComProblema: 0
partidasComContaDeOutraEmpresa: 0
lancamentosEfetivadosComContaDeOutraEmpresa: 0
porMotivo: { outra_empresa: 0, sem_plano: 0, inexistente: 0, sintetica: 0, inativa: 0 }
porTenant: {}                ← nenhum tenant afetado
```

### **ZERO. Não vira ciclo de saneamento.** Por quê, honestamente:

1. O fixture contábil era **mono-empresa** — os 4 lançamentos pré-existentes são todos da Iguassu e foram escritos à mão com contas da própria Iguassu.
2. A contaminação só acontecia em lançamentos **gerados pela ponte em runtime**, e o backend hoje é MSW **em memória**: nada persiste entre sessões.

Ou seja: o dano era **real e reprodutível**, mas **não acumulou passivo**. Foi pego antes de existir backend com escrita durável — que é exatamente onde ele teria virado um problema caro. A auditoria fica no código para quando o `apps/api` existir.

**R-7 respeitada:** nenhum lançamento foi editado. O endpoint audita; o Diário expõe (banner só aparece quando há achado — banner permanente dizendo "está tudo certo" vira ruído que ninguém lê); a correção continua sendo estorno + reescrituração em competência aberta.

---

## 3. As travas aprovadas, provadas por endpoint

### R-2/R-3/R-4/R-5 — cada motivo com sua mensagem, **por empresa**

`POST /v1/fiscal/central-regras` com contas diferentes:

| Conta na regra | Iguassu | Varejo Sul / Metalúrgica / Auditto |
|---|---|---|
| `2.1.3` | **sintética** — "não é lançável, indique uma analítica" | **sintética** (idem) |
| `5.2.9` | **inativa** — "está inativa no Plano de Contas" | **inexistente** — "não existe no Plano de Contas" |
| `9.9.9` | **inexistente** | **inexistente** |
| `5.1.3` | ✅ sem aviso | ✅ sem aviso |

> A linha do `5.2.9` é a demonstração mais forte de que a resolução é por empresa: **o mesmo código recebe veredictos diferentes** conforme o plano de cada uma.

### R-6 — avisa sem bloquear, e marca para revisão

Todas as três regras inválidas foram salvas (**201**), com `avisosContabilizacao` listando empresa por empresa — e `statusRevisao: "revisar"` porque **nenhuma** empresa do escopo reconhece as contas. A regra válida saiu `"ok"`, sem aviso.

### O loop completo — conta inválida não gera lançamento, e o fiscal não trava

```
1. pendência (legado)      → "Escrituração anterior à contabilização automática…"
2. regra → conta sintética → "Conta 2.1.3 é sintética (agrupadora) e não é lançável…"
3. POST escriturar         → 200 · lancamentoContabilId: NULL · pendências: [motivo]
                             ↑ D-1 intacta: o fiscal segue, o contábil não nasce errado
4. regra corrigida + reescriturar → lancamentoContabilId: "lanc-…ie597g" · pendências: []
5. pendência                → []  (sumiu)
```

---

## 4. Dados de demonstração acertados (como você previu na aprovação)

| | Antes | Depois |
|---|---|---|
| Empresas com Plano de Contas | 1 | **4** — Iguassu 35 contas · Varejo Sul, Metalúrgica e Auditto 21 cada |
| Códigos das regras existentes no plano | 0 de 4 | **4 de 4** |

Códigos alinhados à notação real do plano: `1.01.03 → 1.1.3` (Estoques) · `2.01.01 → 2.1.1` (Fornecedores a Pagar) · `4.01.01 → 5.1.4` (Materiais de Consumo) · `4.02.05 → 5.1.3` (Serviços de Terceiros). Contas `5.1.3` e `5.1.4` acrescentadas aos planos.

**R-8 respeitada:** nenhuma normalização automática de notação. Os códigos foram reescritos **um a um**, com escolha semântica explícita — `4.02.05` virou `5.1.3` porque é despesa com serviço, não porque "4" viraria "5".

Incluí de propósito uma conta **inativa** (`5.2.9`) no plano da Iguassu: sem um caso real de conta inativa, a trava R-4 seria código sem prova.

---

## 5. Superfícies e documentação

- **Central de Regras** — banner ao salvar: "Regra salva, mas não vai contabilizar em N empresas", com o motivo por empresa e link para o Plano de Contas.
- **Conecta › aba Contabilidade** — aviso **antes** de confirmar, listando cada conta inválida e por quê; deixa claro que a escrituração segue liberada. Descobrir depois custaria um estorno.
- **Diário** — banner de auditoria quando há lançamento com conta fora do plano, destacando o caso `outra_empresa` e lembrando que se corrige por estorno.
- **Prévia = efetivo**: `gerarPreviaLancamentos` recebeu o mesmo `clienteId` + resolvedor. A prévia valida contra o mesmo plano que o lançamento efetivo usará — não existe caminho em que a prévia diga "vai dar certo" e a contabilização falhe.

**Docs in-app** (mesmo ciclo): conceito novo **"O plano de contas é de CADA empresa"** no Diário, 2 FAQs novas ("a regra funciona numa empresa e não em outra — é bug?" e "dá para corrigir lançamento antigo?"), conceito de sintética estendido com o motivo contábil, e o verbete "Sem contabilização" das Pendências agora cita a conta inválida entre as causas.

**CLAUDE.md** — regra escrita, como determinado:

> **Resolução de entidade por código SEMPRE filtra pelo escopo do tenant/empresa** — buscar por código em acervo global é violação de multi-tenant. […] Ocorrência: `resolverContaContabil` (F1-B1). Ao escrever qualquer resolução por código, o escopo entra na assinatura, não no chamador.

Terceira família de bug recorrente virada regra escrita, junto com a ordem de rotas do MSW e o painel de filtros aplicados.

---

## 6. Checkpoint da seção 4 da trava

| Prova | Resultado |
|---|---|
| Regra com conta inexistente → comportamento aprovado | ✅ não gera lançamento · pendência com motivo · fiscal 200 |
| Conta sintética · conta inativa, cada uma com seu motivo | ✅ provadas, e diferenciadas por empresa |
| Conta de outra empresa não resolve mais | ✅ ids agora são `conta-auditto-cliente1-*` |
| Auditoria dos lançamentos existentes, número real | ✅ **0** de 8 partidas · 0 por tenant |
| Pendência some pelo caminho normal | ✅ corrigir a regra → reescriturar → lançamento nasce |

`tsc --noEmit` limpo · `next lint` sem erros nem warnings.

---

## 7. Estado

`fix/reconciliacao-plano-contas` pronta para merge. **Item 43 quitado** — a menos que você queira alguma coisa diferente no checkpoint.

Nada novo foi para o backlog neste ciclo: o único achado (a violação de tenant) foi corrigido aqui, e o passivo de lançamentos deu zero.

*Relatório de evidências técnicas. Não contém credenciais nem dados de clientes reais.*
