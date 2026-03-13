# Instalacion Global del MCP de KolonLabs

## Objetivo

Definir un modelo de instalacion del MCP `kl-framework` que evite scripts por repositorio.

La idea es:

- hacer una instalacion una sola vez por maquina;
- dejar el arranque del server autocontenido;
- reducir cada repositorio a un `mcp.json` minimo.

Este documento describe la propuesta de trabajo, no el estado actual.

## Resultado esperado

Tras completar el setup de una maquina nueva:

1. Existe un comando global `kl-framework-mcp` disponible en `PATH`.
2. El comando puede arrancar el MCP sin wrapper PowerShell por repo.
3. Cada workspace solo necesita un `.vscode/mcp.json` muy pequeno.
4. La autenticacion a GitHub Packages ocurre en la instalacion, no en cada arranque del workspace.

## Principio de diseno

El problema actual no pertenece a cada repositorio consumidor. Pertenece al arranque y distribucion del MCP.

Por eso la propuesta es mover la complejidad a dos capas compartidas:

1. Un paquete oficial `@kolonlabs/sdk-mcp-server` con un ejecutable global.
2. Un setup organizacional que lo instala una vez por maquina.

Con esto se evita duplicar scripts como `run-kl-framework-mcp.ps1` en todos los repos.

## Requisitos del paquete MCP

El paquete npm debe exponer un comando global. En `package.json` deberia existir algo asi:

```json
{
  "name": "@kolonlabs/sdk-mcp-server",
  "version": "1.0.0",
  "bin": {
    "kl-framework-mcp": "dist/cli.js"
  }
}
```

Ese `cli.js` debe asumir las responsabilidades que hoy estan repartidas entre el repo y el script de arranque:

1. Leer `GITHUB_TOKEN` si existe.
2. Si no existe, intentar `gh auth token`.
3. Definir un valor por defecto de `GITHUB_DISCOVERY` si no viene informado.
4. Validar prerequisitos y emitir errores claros.
5. Arrancar el server MCP por `stdio`.

## Prerrequisitos en la maquina del desarrollador

Antes de instalar el MCP globalmente, la maquina debe tener:

1. Node.js y npm instalados.
2. GitHub CLI (`gh`) instalado.
3. Sesion autenticada en `gh`.
4. Scope `read:packages` habilitado para poder descargar el paquete privado.

Comprobacion basica:

```powershell
gh auth status
gh auth refresh --scopes "read:packages"
```

## Instalacion una sola vez por maquina

La instalacion inicial puede hacerse con PowerShell de esta forma:

```powershell
gh auth status
gh auth refresh --scopes "read:packages"

$token = gh auth token
$tempNpmrc = Join-Path $env:TEMP ("npmrc-" + [guid]::NewGuid().ToString() + ".txt")

Set-Content -Path $tempNpmrc -Value @(
  "@kolonlabs:registry=https://npm.pkg.github.com",
  "//npm.pkg.github.com/:_authToken=$token",
  "always-auth=true"
)

$env:NPM_CONFIG_USERCONFIG = $tempNpmrc
npm install -g @kolonlabs/sdk-mcp-server

Remove-Item $tempNpmrc -Force
```

### Que hace este proceso

1. Reutiliza la sesion ya autenticada de GitHub CLI.
2. Genera un `.npmrc` temporal para acceder al feed privado de GitHub Packages.
3. Instala el paquete una sola vez de forma global.
4. Deja disponible el comando `kl-framework-mcp` para cualquier repo.

## Configuracion minima por repositorio

Una vez instalado el comando global, cada repo solo necesita un `.vscode/mcp.json` con esta estructura:

```json
{
  "servers": {
    "kl-framework": {
      "type": "stdio",
      "command": "kl-framework-mcp",
      "env": {
        "GITHUB_DISCOVERY": "[{\"org\":\"kolonlabs\",\"topics\":[\"kl-framework\"]}]"
      }
    }
  }
}
```

Con este enfoque:

- no hace falta `run-kl-framework-mcp.ps1` por repo;
- no hace falta `npx` en cada arranque;
- no hace falta crear un `.npmrc` local por workspace.

## Actualizacion del MCP

Cuando haya una nueva version del paquete, la actualizacion seria tambien una operacion por maquina:

```powershell
$token = gh auth token
$tempNpmrc = Join-Path $env:TEMP ("npmrc-" + [guid]::NewGuid().ToString() + ".txt")

Set-Content -Path $tempNpmrc -Value @(
  "@kolonlabs:registry=https://npm.pkg.github.com",
  "//npm.pkg.github.com/:_authToken=$token",
  "always-auth=true"
)

$env:NPM_CONFIG_USERCONFIG = $tempNpmrc
npm update -g @kolonlabs/sdk-mcp-server

Remove-Item $tempNpmrc -Force
```

## Verificacion del setup

Comprobar que el ejecutable global existe:

```powershell
Get-Command kl-framework-mcp
```

Comprobar que el repo solo necesita el `mcp.json` minimo y que VS Code puede iniciar el server.

## Limitaciones de este modelo

Este enfoque reduce mucho la friccion, pero no elimina todo el bootstrap inicial.

Sigue siendo necesario:

1. Tener `gh` instalado y autenticado.
2. Tener acceso a GitHub Packages para descargar el paquete privado.
3. Ejecutar una instalacion inicial por maquina.

Lo que si elimina es la necesidad de repetir esa logica en todos los repos.

## Recomendacion de implantacion

Para KolonLabs, la forma mas razonable de implantar esto seria en dos pasos:

1. Publicar `@kolonlabs/sdk-mcp-server` con el binario `kl-framework-mcp`.
2. Incorporar la instalacion global al flujo de "configura mi entorno" de la organizacion.

Con ese modelo, el desarrollador hace el setup una vez y cualquier repo nuevo solo necesita declarar el server en `mcp.json`.

## Siguiente evolucion recomendada

Una vez resuelta la instalacion global, el siguiente paso natural es que el propio launcher sea autocontenido y resuelva internamente:

1. fallback a `gh auth token` cuando `GITHUB_TOKEN` no exista;
2. valor por defecto de `GITHUB_DISCOVERY`;
3. mensajes de diagnostico claros ante falta de login, scopes o conectividad.

Ese cambio es importante porque convierte el runtime del MCP en una pieza reutilizable de organizacion, en lugar de depender de scripts PowerShell duplicados en cada workspace.