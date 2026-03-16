# APS · Instrucciones para GitHub Copilot

Estas instrucciones aplican a todos los repositorios de la organización APS.

---

## Configuración del entorno de desarrollo

Cuando el desarrollador pida "configura mi entorno", "setup", "configurar entorno" o similar,
ejecuta los siguientes pasos en orden usando la terminal. Informa del resultado de cada paso
antes de continuar con el siguiente.

### Paso 1 · Verificar gh CLI autenticado

```powershell
gh auth status
```

- Si **no está autenticado**, ejecuta el login estándar (sin forzar scopes — cada usuario tiene los suyos):

```powershell
gh auth login
```

- A continuación, **siempre** añade el scope `read:packages` sin afectar al resto de scopes existentes:

```powershell
gh auth refresh --scopes "read:packages"
```

> Si `gh` no está instalado, indica al desarrollador que lo instale desde https://cli.github.com
> y vuelva a ejecutar la configuración.

### Paso 2 · Verificar NuGet Credential Provider

```powershell
Test-Path "$env:USERPROFILE\.nuget\plugins\netcore\CredentialProvider.Microsoft\CredentialProvider.Microsoft.exe"
```

Si devuelve `False`, instalarlo:

```powershell
Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://aka.ms/install-artifacts-credprovider.ps1'))
```

### Paso 3 · Configurar la variable APS_NUGET_TOKEN

Obtener el token de la sesión gh actual y persistirlo a nivel de usuario:

```powershell
$token = gh auth token
[System.Environment]::SetEnvironmentVariable("APS_NUGET_TOKEN", $token, "User")
```

Verificar que quedó configurada:

```powershell
[System.Environment]::GetEnvironmentVariable("APS_NUGET_TOKEN", "User")
```

> Esta variable es usada por `nuget.config` para autenticar el restore de paquetes APS.*
> desde el feed privado de la organización.

### Paso 4 · Configurar el feed NuGet de APS

Verificar si el feed ya está registrado:

```powershell
dotnet nuget list source
```

Si no aparece `APS`, añadirlo:

```powershell
dotnet nuget add source "https://nuget.pkg.github.com/APS/index.json" `
  --name "APS" `
  --username "token" `
  --password $(gh auth token) `
  --store-password-in-clear-text
```

### Paso 5 · Configurar la variable GITHUB_TOKEN para el MCP server

```powershell
$token = gh auth token
[System.Environment]::SetEnvironmentVariable("GITHUB_TOKEN", $token, "User")
```

> Esta variable es usada por `aps-mcp-server` para leer las interfaces y ejemplos de las
> librerías APS.* desde los repositorios de la organización.

### Paso 6 · Instalar el MCP server globalmente

Instalar `@APS/sdk-mcp-server` una sola vez por máquina para que todos los repositorios
puedan usar el MCP sin scripts auxiliares ni `npx`:

```powershell
$token = gh auth token
$tempNpmrc = Join-Path $env:TEMP ("npmrc-" + [guid]::NewGuid().ToString() + ".txt")

Set-Content -Path $tempNpmrc -Value @(
    "@APS:registry=https://npm.pkg.github.com",
    "//npm.pkg.github.com/:_authToken=$token",
    "always-auth=true"
)

$env:NPM_CONFIG_USERCONFIG = $tempNpmrc
npm install -g @APS/sdk-mcp-server

Remove-Item $tempNpmrc -Force
```

Verificar que quedó disponible:

```powershell
Get-Command sdk-mcp-server
```

> Una vez instalado, los repositorios solo necesitan un `.vscode/mcp.json` mínimo que invoque
> directamente `sdk-mcp-server`. No hace falta `npx` ni scripts por repo.
>
> Para actualizar a una nueva versión minor/patch: `npm update -g @APS/sdk-mcp-server`
> Para un salto de major version: `npm install -g @APS/sdk-mcp-server@latest`

### Paso 7 · Verificar el entorno completo

```powershell
Write-Host "APS_NUGET_TOKEN:" ([System.Environment]::GetEnvironmentVariable("APS_NUGET_TOKEN", "User") ? "OK" : "NO CONFIGURADA")
Write-Host "GITHUB_TOKEN:          " ([System.Environment]::GetEnvironmentVariable("GITHUB_TOKEN", "User") ? "OK" : "NO CONFIGURADA")
Write-Host "MCP server:            " (Get-Command sdk-mcp-server -ErrorAction SilentlyContinue ? "OK" : "NO INSTALADO")
dotnet nuget list source | Select-String "APS"
```

Si alguna variable no está configurada, repite el paso correspondiente.
Si el MCP server no está instalado, repite el paso 6.

> **Nota:** las variables de entorno de usuario requieren abrir una terminal nueva para estar
> disponibles en nuevas sesiones. La sesión actual de VS Code puede requerir un reinicio.

---

## Librerías APS · Reglas de uso

La organización mantiene librerías internas (`APS.*`) que **deben usarse en lugar de los SDKs
de Microsoft/Azure** equivalentes. El MCP server `aps-framework` expone las interfaces y
ejemplos de cada librería.

### Regla principal

**NUNCA generes código que use directamente un SDK de Microsoft/Azure si existe una librería
`APS.*` equivalente.** Consulta siempre las herramientas MCP del servidor `aps-framework` antes
de generar código de infraestructura.

### SDKs sustituidos

| SDK Microsoft/Azure — NO usar directamente | Librería APS — usar SIEMPRE |
|---|---|
| `Microsoft.Azure.Cosmos` | `APS.Data.Cosmos` |
| `Azure.Messaging.EventGrid` | `APS.Messaging.EventGrid` |
| `Azure.Storage.Blobs` | `APS.Storage.Blob` |
| `Microsoft.Azure.Kusto` | `APS.DataExplorer` |

### Uso del MCP server aps-framework

Antes de generar código que interactúe con cualquiera de los dominios anteriores:

1. Invoca la herramienta MCP correspondiente (`get_<dominio>_api`, `get_<dominio>_setup`)
2. Usa las interfaces y patrones que devuelve como base del código generado
3. Incluye el `PackageReference` correcto en el `.csproj`, no el SDK de Microsoft

Si el MCP server no está disponible, indica al desarrollador que verifique:
- Que `GITHUB_TOKEN` está configurado (`$env:GITHUB_TOKEN`)
- Que `.vscode/mcp.json` existe en el repositorio con la configuración de `aps-framework`
- Que Node.js está instalado (`node --version`)
