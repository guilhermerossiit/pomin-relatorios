# RELATÓRIO — Unificação dos IDs de Empresa · ETAPA A (levantamento + plano)

**Data:** 2026-08-09
**Branch:** `fix/unificacao-ids-empresa` (a partir da `main`)
**Item:** BACKLOG #41 (MÉDIA) — hoje contornado pelo mapa canônico
**Natureza:** análise e proposta. **Nenhum código foi alterado.** Aguardando aprovação do PO.

---

## 1. Inventário — todas as grafias no sistema

Varredura de `src/**/*.ts(x)` por ids de empresa. **8 grafias, 321 ocorrências**, cobrindo **4 empresas reais** + 1 id sintético:

| Empresa (CNPJ) | Grafia do **cadastro** (curta) | Grafia do **mundo fiscal** (longa) | Situação |
|---|---|---|---|
| Iguassu Resort Hotelaria (12.345.678/0001-90) | `empresa-iguassu` ×115 | `empresa-iguassu-resort` ×24 | **divergente** |
| Metalúrgica Pomin S.A. (34.567.890/0001-12) | `empresa-metalurgica` ×42 | `empresa-metalurgica-pomin` ×27 | **divergente** |
| Cliente Auditto 1 (45.678.901/0001-23) | `empresa-auditto-cliente1` ×30 | `empresa-cliente-auditto-1` ×12 | **divergente** |
| Varejo Sul Comércio (23.456.789/0001-01) | `empresa-varejo-sul` ×71 | *(a mesma)* | ✅ **já unificada** |
| — (upload manual) | — | `empresa-manual` ×4 | ⚠️ **órfã** (ver §5) |

A identidade "mesma empresa" está confirmada por **razão social + CNPJ** no cadastro (`fixtures/admin-empresas.ts`) e pelo próprio mapa canônico.

### Onde cada grafia vive (o achado que define o plano)

- **Grafia curta = o sistema inteiro** — 258 ocorrências em **64 arquivos**: Administração, Controle & Compliance (Obrigações/Certidões), BPM Engine, Robot Engine, Dashboards, Central de Alertas, Ativo Imobilizado, Contabilidade, Organizações/Contratos — **inclusive as telas**.
- **Grafia longa = só o mundo fiscal** — 63 ocorrências em **54 linhas de apenas 4 arquivos**:

| Arquivo | Linhas afetadas |
|---|---|
| `mocks/fixtures/fiscal.ts` | 48 |
| `mocks/fixtures/portal-cliente.ts` | 3 *(o próprio mapa canônico)* |
| `mocks/fixtures/portal.ts` | 2 *(tickets de cliente, BC4)* |
| `mocks/handlers-fiscal.ts` | 1 |

**Nenhuma ocorrência longa em telas, componentes, `lib/` ou docs in-app** — a migração não toca UI.

### Onde o mapa canônico é usado hoje

`fixtures/portal-cliente.ts` (`EMPRESAS_PORTAL_CLIENTE` + `empresaPortalPorId`), consumido em **7 pontos** de `handlers-fiscal.ts`: o endpoint `/portal-cliente/empresas` (alimenta o seletor da `ClienteScopeBar`) e 6 reconciliações `empresaPortalPorId(id)?.empresaComplianceId ?? id` (Monitor, Obrigações lista/detalhe, Certidões lista/detalhe, abertura de ticket). O tipo `EmpresaPortalCliente.empresaComplianceId` existe só para isso.

---

## 2. ID canônico proposto: **o do cadastro de Empresa** (grafia curta)

`empresa-iguassu` · `empresa-varejo-sul` · `empresa-metalurgica` · `empresa-auditto-cliente1`

**Por quê:**
1. **É a fonte de verdade organizacional** — a entidade `Empresa` (`admin-empresas.ts`) carrega CNPJ, razão social, contrato, filiais. Documento fiscal referencia empresa; a empresa não referencia documento.
2. **Já é 80% do sistema** (258 de 321 ocorrências, 64 arquivos) — migrar no sentido oposto exigiria tocar telas de 8 módulos.
3. **Já está provado no fiscal**: `empresa-varejo-sul` aparece **13×** dentro de `fixtures/fiscal.ts` e funciona. Não há impedimento estrutural — as outras três são inconsistência histórica, não decisão de design.
4. Menor superfície de risco: **54 linhas, 4 arquivos, zero UI**.

---

## 3. Plano de migração (Etapa B)

1. **`fixtures/fiscal.ts` (48 linhas)** — trocar as 3 grafias longas pelas canônicas. Inclui obrigatoriamente **as CHAVES** dos dois mapas do arquivo: `UF_POR_CLIENTE` e `ORG_POR_CLIENTE` (ver §4 — é aqui que mora o risco).
2. **`fixtures/portal.ts` (2)** — `empresaId` dos tickets de cliente (`tkt-c1`, `tkt-c2`).
3. **`handlers-fiscal.ts` (1)** — a ocorrência longa remanescente.
4. **Remover o mapa canônico:**
   - `EMPRESAS_PORTAL_CLIENTE` passa a listar apenas `id` + `nome` (id já canônico), ou deriva direto do cadastro de Empresas;
   - apagar `empresaComplianceId` do tipo `EmpresaPortalCliente`;
   - simplificar os **6 pontos** `empresaPortalPorId(id)?.empresaComplianceId ?? id` → usar o id direto;
   - manter o endpoint `/portal-cliente/empresas` (o seletor continua precisando da lista).
5. **Remover a regra "todo lote que toca empresa usa o mapa"** — do BACKLOG #41 e do comentário-cabeçalho de `portal-cliente.ts`. Quitar o item 41.

---

## 4. ⚠️ O que quebra — e o risco que o fallback esconde

**RISCO ALTO — cruzamento de tenant, silencioso.** Em `fixtures/fiscal.ts`:

```ts
export const ORG_POR_CLIENTE: Record<string, string> = {
  'empresa-iguassu-resort': 'org-pomin',
  'empresa-varejo-sul': 'org-pomin',
  'empresa-metalurgica-pomin': 'org-pomin',
  'empresa-cliente-auditto-1': 'org-auditto',   // ← única empresa de OUTRO tenant
};
export function organizacaoDoCliente(clienteId: string): string {
  return ORG_POR_CLIENTE[clienteId] ?? 'org-pomin';   // ← fallback SILENCIOSO
}
```

Se a migração trocar os `clienteId` dos documentos **sem** trocar as **chaves** deste mapa, a busca falha e o fallback devolve `org-pomin` **sem erro nenhum**. Para `empresa-cliente-auditto-1` (tenant `org-auditto`) isso significa **documentos migrando para o tenant errado** — exatamente o tipo de falha que não aparece em tela. O mesmo padrão vale para `UF_POR_CLIENTE` (SP → viraria PR por omissão, mudando a sugestão de regra fiscal).

**Trava obrigatória da Etapa B:** provar por endpoint, depois da migração, que **a organização e a UF de cada empresa continuam as mesmas de antes** — com atenção especial a Auditto → `org-auditto`.

**Demais impactos (baixos):**
- **Escopo de sessão do portal:** o valor gravado em `localStorage` (`empresa:empresa-metalurgica-pomin`) passa a ser `empresa:empresa-metalurgica`. Não quebra produção (é escolha de sessão, remontada pelo seletor), mas os ids das evidências dos ciclos BC1–BC4 mudam de grafia.
- **Nenhuma migração de dados persistidos** — o backend é MSW em memória; o estado nasce das fixtures a cada carga.
- **Sem impacto em UI/docs** — nenhuma ocorrência longa fora de `mocks/`.

---

## 5. Achado adicional (fora do item 41) — decisão do PO

**`empresa-manual`** é um `clienteId` sintético criado pelo upload manual de documentos (`/v1/fiscal-capture/manual/upload`) e **não existe no cadastro de Empresas**. Consequências hoje: o documento cai no fallback `org-pomin` e **não aparece para nenhuma empresa** no Portal do Cliente (seguro, mas invisível) nem no recorte interno por empresa.

**D-ids-1 — como tratar?** *(recomendação: opção **a**, para não misturar com a unificação)*
- **(a)** Manter como está e registrar no backlog: documento de upload manual nasce "sem empresa" até a triagem interna definir a empresa.
- **(b)** Passar a exigir a empresa no upload manual (mudança de comportamento — fora do escopo deste ciclo).

---

## 6. Resumo do alcance

| Dimensão | Números |
|---|---|
| Grafias hoje | 8 (4 empresas + 1 id órfão) |
| Pares divergentes a unificar | **3** (`varejo-sul` já unificada) |
| Ocorrências que mudam | **63** (54 linhas, **4 arquivos**, todos em `mocks/`) |
| Ocorrências que permanecem | 258 (64 arquivos) — já canônicas |
| Telas/componentes afetados | **0** |
| Pontos que perdem a reconciliação | 7 (6 usos + 1 tipo) — o mapa é deletado |
| Risco principal | Fallback silencioso de `ORG_POR_CLIENTE` (cruzamento de tenant) |

---

## PARADA

Aguardo sua aprovação de: **(1)** o id canônico proposto (grafia do cadastro de Empresa), **(2)** o plano de migração da §3, e **(3)** a decisão **D-ids-1** sobre o `empresa-manual`. Só então executo a Etapa B, com a trava de tenant/UF provada por endpoint.

*Relatório de análise. Não contém credenciais, tokens nem dados de clientes reais — os identificadores citados são fixtures do ambiente de desenvolvimento.*
