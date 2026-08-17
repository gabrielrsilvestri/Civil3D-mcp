# Bug: `civil3d_profile view_create` falha — `ProfileView.Create` chamado com ordem de parâmetros incorreta

## Ambiente
- Civil 3D 2026.2.2 Update (Product Version 13.8.1809.0)
- Built on: AutoCAD 2026.1.2, AutoCAD Map 3D 2026.0.4, AutoCAD Architecture 2026.0.2
- Sacred-G/Civil3D-mcp — `Civil3D-MCP-Plugin`, versão reportada 1.2.1.0
- Target framework: `net10.0-windows` (ajustado manualmente de `net8.0-windows`)

## Sintoma
`civil3d_profile` com `action: "view_create"` falha sempre com:
```
civil3d_profile action 'view_create' failed: ProfileView.Create returned null — this Civil 3D version may require a different API signature.
```
Reproduzido de forma consistente (parâmetros mínimos e completos, com/sem `style` e
`bandSet` explícitos). `create_from_surface` (criação de `Profile`) funciona normalmente
— o problema é isolado à criação da **Profile View**.

## Localização
`Civil3D-MCP-Plugin\ProfileEditCommands.cs`, método `ProfileViewCreateAsync`
(~linha 220–270).

Código original: 3 tentativas de overload via reflection (`InvokeStaticMethod`),
encadeadas com `??`, todas com `profileViewName` como **primeiro parâmetro**:

```csharp
var pvId = (ObjectId?)(
  CivilObjectUtils.InvokeStaticMethod(profileViewType, "Create",
    profileViewName, alignment.ObjectId, insertionPoint, styleId, bandSetId)
  ?? CivilObjectUtils.InvokeStaticMethod(profileViewType, "Create",
    profileViewName, alignment.ObjectId, styleId, insertionPoint)
  ?? CivilObjectUtils.InvokeStaticMethod(profileViewType, "Create",
    profileViewName, alignment.ObjectId, insertionPoint));
```

O próprio autor original deixou comentários indicando incerteza sobre a assinatura:
```csharp
// ProfileView.Create(profileViewName, alignmentId, styleId, insertionPoint)
// or ProfileView.Create(profileViewName, alignmentId, insertPosition, styleId, bandSetId)
```

## Causa raiz (confirmada via decompilação de AeccDbMgd.dll com ILSpy 11.0.0.9375)

Namespace `Autodesk.Civil.DatabaseServices`, classe `ProfileView`. Os overloads estáticos
reais de `Create` que retornam `ObjectId` (não `ObjectIdCollection`, que é para profile
views empilhadas/múltiplas):

```csharp
// Overload completo:
public static ObjectId Create(ObjectId alignmentId, Point3d insertPosition,
    string profileViewName, ObjectId profileViewBandSetId, ObjectId profileViewStyleId)

// Overload simples (usa padrões do documento):
public static ObjectId Create(ObjectId alignmentId, Point3d insertPosition)

// Variante com opções de split:
public static ObjectId Create(ObjectId alignmentId, Point3d insertPosition,
    SplitProfileViewCreationOptions splitOptions)
```

Nenhuma das 3 tentativas do plugin bate com nenhum overload real:
1. `alignmentId` sempre é o **primeiro** parâmetro, não `profileViewName` — o nome só
   aparece como **terceiro** parâmetro no overload completo.
2. No overload completo, a ordem correta é
   `(alignmentId, insertPosition, profileViewName, profileViewBandSetId, profileViewStyleId)`
   — **band set vem antes de style**, invertido em relação ao que o código (e o
   comentário do autor) assumiam.

Como a chamada é via reflection (`InvokeStaticMethod`), nenhum dos três overloads
inexistentes gera exceção — a reflection simplesmente não encontra o método com essa
assinatura e retorna `null` silenciosamente, três vezes seguidas, até cair no erro
genérico "returned null — this Civil 3D version may require a different API signature",
que mascara a causa real (não é incompatibilidade de versão, é ordem de parâmetros).

## Correção aplicada

```csharp
var pvId = (ObjectId?)(
  CivilObjectUtils.InvokeStaticMethod(profileViewType, "Create",
    alignment.ObjectId, insertionPoint, profileViewName, bandSetId, styleId)
  ?? CivilObjectUtils.InvokeStaticMethod(profileViewType, "Create",
    alignment.ObjectId, insertionPoint));
```
- 1ª tentativa: overload completo, ordem de parâmetros corrigida.
- 2ª tentativa (fallback): overload simples, sem nome/estilo — usada se `styleId`/
  `bandSetId` vierem nulos ou inválidos.

## Padrão a varrer no restante do repositório

Mesmo autor, mesmo tipo de erro (`InvokeStaticMethod` + encadeamento `??` com ordem de
parâmetros assumida sem confirmação contra a assinatura real). Candidatos a verificar,
comparando cada um contra a assinatura real via ILSpy antes de alterar:
- Handlers de `Alignment`
- Handlers de `Corridor`
- Handlers de `PipeNetwork`
- Handlers de `Parcel`

## Status
- [x] Causa raiz confirmada via decompilação (ILSpy)
- [x] Correção escrita e aplicada em `ProfileEditCommands.cs`
- [ ] Build (`dotnet build .\Civil3D-MCP-Plugin\Civil3DMcpPlugin.csproj -c Release`)
- [ ] Varredura do restante do repo atrás do mesmo padrão
- [ ] Teste manual via `NETLOAD` no Civil 3D real (fora do alcance do Claude Code)
- [ ] PR para `Sacred-G/Civil3D-mcp` após teste confirmado
