# Civil3D-mcp — Contexto de projeto para Claude Code

Fork de [Sacred-G/Civil3D-mcp](https://github.com/Sacred-G/Civil3D-mcp), plugin MCP para Autodesk Civil 3D 2026 (AeccDbMgd.dll v13.8.0.1809).

Owner: Gabriel Roberto Silvestri (gabrielrsilvestri) — CREA-RS 238509, G23 Engenharia.
Remotes: `origin` = fork pessoal, `upstream` = repo original do Sacred-G.

## Padrão de erro recorrente no plugin

Os bugs já confirmados neste projeto seguem o **mesmo padrão estrutural**: código que acessa
propriedades da API do Civil 3D no **nível errado do grafo de objetos** — seja por reflection
genérica (`GetType().GetProperty("NomeQualquer")`) que não encontra a propriedade no objeto de
grupo e falha silenciosamente (retornando `0`/`null` em vez de propagar erro), seja por ler um
parâmetro num objeto pai quando na verdade ele só existe numa subentidade.

**Regra de investigação:** sempre que uma leitura da API Civil3D retornar um valor "zerado"
suspeito (raio 0, comprimento 0, parâmetro A 0) em vez de lançar exceção, a primeira hipótese
a testar é essa — objeto de grupo sem a propriedade, precisa descer para a subentidade correta.

## Bugs confirmados

### 1. `ProfileView.Create` — parâmetros fora de ordem
Ver histórico de debugging anterior (`ProfileEditCommands.cs`). Retorno nulo ao criar
Profile View por parâmetros passados na ordem incorreta na chamada da API.

### 2. `AlignmentSCS.Radius` / spiral `.A` retornam 0 (CONFIRMADO via decompilação)

**Sintoma:** `civil3d_alignment get/report` e `civil3d_qc check_alignment` retornam
`radius: 0` e `spiralParameter (A): 0` para toda entidade `SpiralCurveSpiral`, mesmo com
curvas circulares válidas no desenho (confirmado visualmente na grade do Civil 3D:
R=180,000m / R=135,000m / R=180,000m). Além disso, tangentes curtas entre clotoides
(~1–3m reais) são reportadas com `length: 0` e sinalizadas como erro `tangent_zero_length`.

**Causa raiz confirmada por decompilação de `AeccDbMgd.dll` (ILSpy/dnSpy):**

`Autodesk.Civil.DatabaseServices.AlignmentSCS` **não tem propriedade `Radius` nem `A`**.
Seus membros reais são:
```
Arc : AlignmentSubEntityArc
SpiralIn : AlignmentSubEntitySpiral
SpiralOut : AlignmentSubEntitySpiral
Constraint2 : AlignmentSCSConstraintType
GreaterThan180 : bool
```

O raio real está em `AlignmentSCS.Arc.Radius` (getter confirmado limpo — lê via
`AttributeHelper.getAttributeDouble`, sem dependência de `GreaterThan180` nem de nenhum
outro flag condicional). O parâmetro A está em `SpiralIn.A` / `SpiralOut.A`.

**Correção:**
```csharp
if (entity.EntityType == AlignmentEntityType.SpiralCurveSpiral)
{
    var scs = (AlignmentSCS)entity;
    double radius = scs.Arc.Radius;
    double aIn    = scs.SpiralIn.A;
    double aOut   = scs.SpiralOut.A;
}
```

**Arquivo provável do bug:** `AlignmentCommands.cs` (mesmo módulo/estilo de
`ProfileEditCommands.cs`). Procurar por `GetProperty("Radius")` ou qualquer
`try/catch` genérico ao redor da leitura de entidades de alinhamento.

**Pendente de verificação:**
- Confirmar se o mesmo padrão se repete em `AlignmentSTS`, `AlignmentSSCSS`, `AlignmentSCSCS`.
- Isolar a causa do falso `tangent_zero_length` nas linhas curtas entre clotoides — não
  confirmado ainda se é o mesmo tipo de erro de reflection ou um cálculo de comprimento
  usando a estação errada.
- Rodar diagnóstico `DUMPALIGNSCS` (comando de reflection completa, mesmo padrão do
  `DUMPPVCREATE` usado no bug do `ProfileView.Create`) para confirmar em runtime que
  `Arc.Radius` retorna os valores esperados antes de aplicar o patch em produção.

## Ambiente de dev
- Node.js + .NET 10 (houve mismatch net8.0→net10.0 no `.csproj`, já resolvido)
- Civil 3D 2026, plugin conectado via `localhost:8080`
- Referenciar `AeccDbMgd.dll` diretamente no projeto em vez de reflection genérica sempre
  que possível — reduz esse tipo de erro e é mais robusto a mudanças de versão da API.

## Workflow de PR
1. Corrigir no fork (`origin`), branch separada por bug (ex: `fix/alignment-scs-radius`).
2. Testar contra o desenho real (`RAMO_100`, curvas R=180/135/180m) antes de abrir PR.
3. Abrir PR de `origin` para `upstream` (Sacred-G/Civil3D-mcp) com a nota técnica em
   `docs/issues/`.
