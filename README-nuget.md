# NuGet · Publicación y Consumo

> Referencia para desarrolladores y pipelines CI/CD.
> Aplicable a cualquier repositorio de la organización APS que publique paquetes NuGet.

---

## Índice

- [1. Feed privado de la organización](#1-feed-privado-de-la-organización)
- [2. Configurar credenciales (entorno local)](#2-configurar-credenciales-entorno-local)
  - [2.1 Requisitos previos](#21-requisitos-previos)
  - [2.2 Configurar las variables de entorno](#22-configurar-las-variables-de-entorno)
- [3. Flujo de versiones y publicación](#3-flujo-de-versiones-y-publicación)
  - [3.1 Durante desarrollo (sin publicar)](#31-durante-desarrollo-sin-publicar)
  - [3.2 Publicar una pre-release (RC)](#32-publicar-una-pre-release-rc)
  - [3.3 Publicar la versión estable](#33-publicar-la-versión-estable)
  - [3.4 Preparar la siguiente versión](#34-preparar-la-siguiente-versión)
  - [3.5 Repos con múltiples paquetes](#35-repos-con-múltiples-paquetes)
- [4. Consumir los paquetes](#4-consumir-los-paquetes)
  - [4.1 En desarrollo local](#41-en-desarrollo-local)
  - [4.2 En GitHub Actions (build / test / deploy)](#42-en-github-actions-build--test--deploy)
- [5. Referencia rápida de credenciales](#5-referencia-rápida-de-credenciales)
- [6. Configurar un nuevo repositorio SDK](#6-configurar-un-nuevo-repositorio-sdk)
  - [6.1 Repositorio con un solo paquete](#61-repositorio-con-un-solo-paquete)
  - [6.2 Repositorio con múltiples paquetes](#62-repositorio-con-múltiples-paquetes)
  - [6.3 nuget.config](#63-nugetconfig)
  - [6.4 Secret de organización `APS_NUGET_TOKEN`](#64-secret-de-organización-APS_nuget_token)
- [7. Referencia técnica del workflow](#7-referencia-técnica-del-workflow)
  - [7.1 `.github/workflows/nuget-ci-publish.yml`](#71-githubworkflowsnuget-ci-publishyml)
  - [7.2 Especificación de steps](#72-especificación-de-steps)
- [8. Scripts compartidos](#8-scripts-compartidos)
  - [8.1 `.github/scripts/nuget_publish.py`](#81-githubscriptsnuget_publishpy)
  - [8.2 Responsabilidades internas del script](#82-responsabilidades-internas-del-script)

---

## 1. Feed privado de la organización

Todos los paquetes `APS.*` se publican en GitHub Packages de la organización APS-Framework:

```
https://nuget.pkg.github.com/APS-Framework/index.json
```

Los paquetes disponibles en cada repositorio se listan en su propio `README.md`. Un repositorio puede publicar uno o múltiples paquetes:

| Paquete | Descripción |
|---|---|
| `APS.{Paquete1}` | _{descripción del paquete 1}_ |
| `APS.{Paquete2}` | _{descripción del paquete 2}_ |

---

## 2. Configurar credenciales (entorno local)

GitHub Packages requiere autenticación para consumir paquetes desde local. No es necesario generar ningún PAT manualmente — se reutiliza el token de la sesión `gh` activa.

> **Para el secret de organización usado en CI/CD** (publicación desde GitHub Actions), ver sección [6.4](#64-secret-de-organización-APS_nuget_token). Ese PAT lo gestiona únicamente el administrador de la organización.

### 2.1 Requisitos previos

1. Tener **gh CLI** instalado y autenticado: https://cli.github.com
2. Asegurarse de que la sesión tiene el scope `read:packages`:

```powershell
# Verificar sesión activa
gh auth status

# Si no está autenticado:
gh auth login

# Añadir read:packages sin afectar otros scopes existentes (siempre seguro ejecutar):
gh auth refresh --scopes "read:packages"
```

> El comando `gh auth refresh --scopes` es acumulativo — añade el scope indicado sin eliminar los que ya tiene el usuario.

### 2.2 Configurar las variables de entorno

Captura el token de la sesión activa y persístelo a nivel de usuario:

```powershell
$token = gh auth token
[System.Environment]::SetEnvironmentVariable("APS_NUGET_TOKEN", $token, "User")
[System.Environment]::SetEnvironmentVariable("GITHUB_TOKEN", $token, "User")
```

> **Nota:** abre una terminal nueva tras ejecutar estos comandos para que las variables estén disponibles.

Para verificar que están configuradas:

```powershell
[System.Environment]::GetEnvironmentVariable("APS_NUGET_TOKEN", "User")
[System.Environment]::GetEnvironmentVariable("GITHUB_TOKEN", "User")
```

Cuando `gh` renueve su sesión, repite estos comandos para sincronizar las variables con el nuevo token. Si usas **Copilot en modo Agent**, puedes pedir `configura mi entorno` y lo ejecutará automáticamente.

---

## 3. Flujo de versiones y publicación

La publicación se lanza **manualmente** desde GitHub Actions (`workflow_dispatch`). Un push normal a `main` nunca publica nada — solo ejecuta build y tests.

El workflow lee el `VersionPrefix` del `.csproj`, calcula la versión correspondiente, publica el paquete, crea el tag en Git y genera la GitHub Release.

### 3.1 Durante desarrollo (sin publicar)

Un push a `main` o la apertura de una PR dispara automáticamente el workflow — solo la fase **Build & Test**, sin publicar nada. No hay ningún comando adicional que lanzar:

```powershell
git add .
git commit -m "feat: descripción del cambio"
git push   # → GitHub Actions ejecuta build + test automáticamente
```

### 3.2 Publicar una pre-release (RC)

Cuando quieres que otros equipos prueben el trabajo en progreso, lanza el workflow indicando `rc`:

```powershell
# El workflow calcula automáticamente el número de RC consultando los tags existentes
gh workflow run publish.yml -f release_type=rc
```

El workflow:
1. Lee `VersionPrefix` del `.csproj` (ej: `0.2.0`)
2. Consulta los tags existentes para calcular el siguiente RC (ej: `v0.2.0-rc.1` si no hay ninguno, `v0.2.0-rc.2` si ya existe el primero)
3. Ejecuta `dotnet pack /p:Version=0.2.0-rc.1`
4. Publica el paquete en GitHub Packages
5. Crea el tag `v0.2.0-rc.1` en el repositorio
6. Crea una GitHub Release marcada como pre-release

Si la publicación del paquete falla, no se llega a crear el tag. Si la creación de la GitHub Release falla después de crear el tag, el workflow elimina ese tag automáticamente para evitar dejar RCs huérfanos.

> Los paquetes pre-release **no se instalan automáticamente** en proyectos consumidores al hacer `dotnet restore`. Deben referenciarse de forma explícita: `Version="0.2.0-rc.1"`. Esto garantiza que no rompen a nadie por defecto.

### 3.3 Publicar la versión estable

Cuando la versión está validada y lista para producción:

```powershell
gh workflow run publish.yml -f release_type=stable
```

El workflow verifica que no existe ya un tag `v{VersionPrefix}` y falla con un mensaje claro si lo hay — lo que indicaría que hay que actualizar el `VersionPrefix` antes de publicar.

### 3.4 Preparar la siguiente versión

Tras publicar la versión estable, actualiza el `VersionPrefix` en los `.csproj` para iniciar el siguiente ciclo:

```xml
<!-- src/APS.{Paquete1}/APS.{Paquete1}.csproj -->
<VersionPrefix>0.3.0</VersionPrefix>
```

```powershell
git add .
git commit -m "chore: bump version to 0.3.0"
git push
```

### 3.5 Repos con múltiples paquetes

Cuando un repositorio publica más de un paquete, cada uno tiene su propio `VersionPrefix` en su `.csproj`. El workflow usa un único parámetro, `packages`, que acepta uno o varios paquetes separados por comas.

```powershell
# Publicar solo APS.{Paquete1}
gh workflow run publish.yml -f release_type=stable -f packages=APS.{Paquete1}

# Publicar solo APS.{Paquete2} como RC
gh workflow run publish.yml -f release_type=rc -f packages=APS.{Paquete2}

# Publicar varios paquetes estables en una sola ejecución
gh workflow run publish.yml -f release_type=stable -f packages=APS.{Paquete1},APS.{Paquete2}

# Publicar varios paquetes RC en una sola ejecución
gh workflow run publish.yml -f release_type=rc -f packages=APS.{Paquete1},APS.{Paquete2}
```

El workflow localiza el `.csproj` de cada paquete indicado, lee su `VersionPrefix`, calcula su versión de forma independiente y publica los paquetes dentro de la misma ejecución en un orden compatible con sus dependencias internas.

Como la lógica de publicación avanzada vive en este repositorio central, el workflow reutilizable descarga su script compartido desde `APS-Framework/.github` durante la ejecución. El repositorio caller no necesita copiar ese script.

Si un paquete referencia otro paquete del mismo repositorio mediante `ProjectReference`, el workflow genera un `.csproj` temporal para el empaquetado y sustituye esa referencia por un `PackageReference`:

- Si la dependencia también se publica en la misma ejecución, usa la versión calculada en esa ejecución.
- Si la dependencia no se publica en esa ejecución, usa la última versión ya publicada en GitHub Packages.
- En releases `rc`, intenta usar la última RC publicada de esa dependencia; si no existe, usa la última estable disponible.

Si se omite `packages` y el repositorio contiene múltiples `.csproj` bajo `src`, el workflow falla de forma explícita para evitar publicar el paquete incorrecto por accidente.

Los tags en repos multi-paquete incluyen el nombre del paquete como prefijo para evitar colisiones entre versiones de distintos paquetes: `APS.{Paquete1}-v0.2.0`, `APS.{Paquete2}-v1.0.0-rc.1`.

---

## 4. Consumir los paquetes

### 4.1 En desarrollo local

**1. `nuget.config` en la raíz del repositorio consumidor:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGet official package source" value="https://api.nuget.org/v3/index.json" />
    <add key="APS-Framework" value="https://nuget.pkg.github.com/APS-Framework/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <APS-Framework>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </APS-Framework>
  </packageSourceCredentials>
</configuration>
```

**2. Variable de entorno** configurada según la sección [2.2](#22-configurar-las-variables-de-entorno).

**3. Referencia en el `.csproj`:**

```xml
<!-- Versión estable -->
<PackageReference Include="APS.{Paquete1}" Version="0.2.0" />

<!-- Pre-release (explícito) -->
<PackageReference Include="APS.{Paquete1}" Version="0.2.0-rc.1" />
```

`dotnet restore` resolverá los paquetes automáticamente usando la variable de entorno.

---

### 4.2 En GitHub Actions (build / test / deploy)

> **Importante:** `GITHUB_TOKEN` en GitHub Actions solo tiene acceso a los paquetes publicados por el **propio repositorio**. Para consumir paquetes de otros repositorios de la organización (p.ej. `APS.Common` desde `sdk-common`), se obtiene un **403 Forbidden**. Es obligatorio usar el secret de organización `APS_NUGET_TOKEN`.

El secret `APS_NUGET_TOKEN` se configura a nivel de organización (ver sección [6.4](#64-secret-de-organización-APS_nuget_token)) y se propaga automáticamente a todos los repos mediante `secrets: inherit` en el workflow caller.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '{dotnet-version}'   # ej: 10.x

      - name: Restore
        run: dotnet restore
        env:
          APS_NUGET_TOKEN: ${{ secrets.APS_NUGET_TOKEN }}

      - name: Build
        run: dotnet build --no-restore -c Release
```

> **Para despliegues a Azure** (App Service, Container Apps, Functions): el build y el restore ocurren en el runner de GitHub Actions, no en Azure. Azure recibe los binarios ya compilados. El feed NuGet nunca necesita ser accesible desde Azure.

---

## 5. Referencia rápida de credenciales

| Escenario | Credencial | Scope requerido |
|---|---|---|
| Consumir en local | Token de sesión `gh` en `APS_NUGET_TOKEN` (variable de usuario) | `read:packages` |
| Consumir en GitHub Actions | Org secret `APS_NUGET_TOKEN` (via `secrets: inherit`) | `read:packages` |
| Publicar desde GitHub Actions | Org secret `APS_NUGET_TOKEN` (via `secrets: inherit`) | `write:packages` |
| Azure en runtime | — | No necesario |

> `GITHUB_TOKEN` **no funciona** para consumir paquetes de otros repositorios de la organización. Solo da acceso a los paquetes del repo donde se ejecuta el workflow.

---

## 6. Configurar un nuevo repositorio SDK

La lógica de CI y publicación está centralizada en el workflow reutilizable del repositorio `APS-Framework/.github`. Cada repositorio SDK solo necesita un fichero caller de ~15 líneas que invoca ese workflow centralizado.

### 6.1 Repositorio con un solo paquete

Crea `.github/workflows/publish.yml`:

```yaml
name: CI & Publish

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      release_type:
        description: 'Tipo de release (rc | stable | vacío = solo CI)'
        required: false
        type: choice
        default: ''
        options: ['', rc, stable]

# Permisos top-level: necesarios para crear tags, releases y publicar paquetes.
# No poner permissions a nivel de job — causa startup_failure en workflows reutilizables.
permissions:
  contents: write
  packages: write

jobs:
  ci-publish:
    uses: APS-Framework/.github/.github/workflows/nuget-ci-publish.yml@main
    with:
      release_type: ${{ inputs.release_type || '' }}
    secrets: inherit
```

### 6.2 Repositorio con múltiples paquetes

El caller expone un único input `packages`. Puede recibir un solo paquete o varios separados por comas:

```yaml
name: CI & Publish

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      release_type:
        description: 'Tipo de release (rc | stable | vacío = solo CI)'
        required: false
        type: choice
        default: ''
        options: ['', rc, stable]
      packages:
        description: 'Uno o varios paquetes separados por comas'
        required: false
        type: string
        default: ''

permissions:
  contents: write
  packages: write

jobs:
  ci-publish:
    uses: APS-Framework/.github/.github/workflows/nuget-ci-publish.yml@main
    with:
      release_type: ${{ inputs.release_type || '' }}
      packages: ${{ inputs.packages || '' }}
    secrets: inherit
```

Los comandos de publicación son los mismos descritos en las secciones 3.2, 3.3 y 3.5 — el caller expone los mismos inputs hacia el usuario final. Si el repositorio tiene varios paquetes y no se envía `packages`, el workflow falla por ambigüedad.

### 6.3 nuget.config

Copia el siguiente `nuget.config` a la raíz del nuevo repositorio:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGet official package source" value="https://api.nuget.org/v3/index.json" />
    <add key="APS-Framework" value="https://nuget.pkg.github.com/APS-Framework/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <APS-Framework>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </APS-Framework>
  </packageSourceCredentials>
</configuration>
```

Cada fuente privada usa su propia variable de entorno en `ClearTextPassword`. Si el repositorio consume paquetes de otras organizaciones GitHub, añade una entrada por fuente con su variable correspondiente:

```xml
<!-- En packageSources -->
<add key="OtraOrg" value="https://nuget.pkg.github.com/OtraOrg/index.json" />

<!-- En packageSourceCredentials -->
<OtraOrg>
  <add key="Username" value="x" />
  <add key="ClearTextPassword" value="%OTRA_ORG_NUGET_TOKEN%" />
</OtraOrg>
```

> La variable `%OTRA_ORG_NUGET_TOKEN%` debe estar disponible en el entorno (local o CI). En GitHub Actions, configúrala como secret en el repositorio u organización y expónla como variable de entorno en el step `Restore`.

---

### 6.4 Secret de organización `APS_NUGET_TOKEN`

Este secret es la pieza central que permite a todos los workflows de CI/CD de la organización consumir y publicar paquetes NuGet cross-repo. Se configura **una sola vez** a nivel de organización.

**Por qué es necesario:**
`GITHUB_TOKEN` en GitHub Actions solo tiene acceso a los paquetes del repositorio donde se ejecuta. Para acceder a paquetes de otros repos del mismo org (p.ej. `APS.Common` desde `sdk-common`), se devuelve un **403 Forbidden**. El org secret soluciona esto con un PAT que tiene permisos explícitos sobre todos los paquetes de la organización.

**Configuración inicial (una sola vez):**

1. Genera un Classic PAT con scopes:
   - `write:packages` (incluye `read:packages`)
   - `repo` (para acceso a repos privados)

2. Configura el secret a nivel de organización (visible para todos los repos):

```powershell
"ghp_TU_TOKEN" | gh secret set APS_NUGET_TOKEN --org APS-Framework --visibility all
```

3. Los workflows usan `secrets: inherit` en el job caller, lo que propaga automáticamente este secret al workflow reutilizable.

**Renovación:** cuando el PAT expire, genera uno nuevo y repite el paso 2. No es necesario tocar ningún repositorio individual.

---

## 7. Referencia técnica del workflow

### 7.1 `.github/workflows/nuget-ci-publish.yml`

Este es el workflow reutilizable oficial para publicación NuGet en la organización. Expone estos inputs:

| Input | Tipo | Descripción |
|---|---|---|
| `release_type` | `string` | `rc`, `stable` o vacío. Vacío ejecuta solo CI. |
| `packages` | `string` | Uno o varios `PackageId` separados por comas. Si el repositorio solo tiene un paquete, puede omitirse. |

El workflow tiene dos jobs principales:

| Job | Cuándo se ejecuta | Objetivo |
|---|---|---|
| `build` | Siempre | Ejecutar `restore`, `build` y `test` del repositorio. |
| `publish` | Solo si `release_type != ''` | Publicar paquetes NuGet, crear tags y generar releases. |

### 7.2 Especificación de steps

Las siguientes tablas enumeran todos los steps reales del workflow en el orden exacto en que se ejecutan.

#### Job `build`

| Orden | Step | Tipo | Descripción breve |
|---|---|---|---|
| 1 | `actions/checkout@v4` | Action | Descarga el repositorio caller en el runner para ejecutar CI sobre su código real. |
| 2 | `Setup .NET` | Action (`actions/setup-dotnet@v4`) | Prepara el SDK .NET soportado por la organización, actualmente `8.x` y `10.x`. |
| 3 | `Restore` | Shell step | Ejecuta `dotnet restore` con `APS_NUGET_TOKEN` para restaurar dependencias privadas cross-repo. |
| 4 | `Build` | Shell step | Ejecuta `dotnet build --no-restore -c Release` para validar compilación completa. |
| 5 | `Test` | Shell step | Ejecuta `dotnet test --no-build -c Release`; si falla, no se publica nada. |

#### Job `publish`

| Orden | Step | Tipo | Descripción breve |
|---|---|---|---|
| 1 | `actions/checkout@v4` | Action | Descarga el repositorio caller con `fetch-depth: 0` para poder inspeccionar y crear tags Git. |
| 2 | `Checkout shared workflow scripts` | Action (`actions/checkout@v4`) | Descarga `APS-Framework/.github` en `.APS-github` para disponer de scripts compartidos del repositorio central. |
| 3 | `Setup .NET` | Action (`actions/setup-dotnet@v4`) | Prepara el SDK .NET para el empaquetado y la publicación. |
| 4 | `Setup Python` | Action (`actions/setup-python@v5`) | Garantiza una versión estable de Python para ejecutar el script de publicación. |
| 5 | `Restore` | Shell step | Ejecuta `dotnet restore` en el contexto del job de publicación, con acceso al feed privado. |
| 6 | `Publicar paquetes` | Shell step | Ejecuta el script `nuget_publish.py`, que orquesta versiones, dependencias internas, `pack`, `push`, tags y releases. |

---

## 8. Scripts compartidos

### 8.1 `.github/scripts/nuget_publish.py`

Script Python compartido usado por `nuget-ci-publish.yml` para publicar paquetes NuGet en repositorios mono-paquete y multi-paquete.

Entradas principales:

| Argumento | Descripción |
|---|---|
| `--release-type` | Canal de publicación: `rc` o `stable`. |
| `--packages` | Lista CSV de `PackageId` a publicar. |
| `--repo-root` | Ruta raíz del repositorio caller. |
| `--org` | Organización de GitHub donde se consulta GitHub Packages. |

Variables de entorno requeridas:

| Variable | Uso |
|---|---|
| `APS_NUGET_TOKEN` | Publicar paquetes y consultar versiones publicadas. |
| `GH_TOKEN` | Crear GitHub Releases. |

### 8.2 Responsabilidades internas del script

1. **Descubrir proyectos publicables**
  - Recorre `src/**/*.csproj`.
  - Lee `PackageId` y `VersionPrefix`.
  - Detecta `ProjectReference` internos entre paquetes del mismo repositorio.

2. **Validar selección de paquetes**
  - Interpreta el input `packages`.
  - Falla si se pide un paquete inexistente o si hay ambigüedad en repos multi-paquete sin input.

3. **Ordenar la publicación**
  - Construye el grafo de dependencias internas.
  - Publica en orden topológico para que los paquetes base estén disponibles antes que sus dependientes.

4. **Calcular versión y tag por paquete**
  - `stable`: usa `VersionPrefix` tal cual y falla si el tag ya existe.
  - `rc`: calcula el siguiente sufijo `-rc.N` a partir de los tags existentes.

5. **Resolver dependencias internas**
  - Si una dependencia también se publica en la misma ejecución, usa esa versión calculada.
  - Si no se publica, consulta la última versión ya publicada en GitHub Packages.
  - En canal `rc`, prefiere la última RC publicada y, si no existe, usa la última estable.

6. **Generar un `.csproj` temporal**
  - Sustituye `ProjectReference` internos por `PackageReference` con versión fija.
  - Conserva el `PackageId` original del paquete para que el `.nupkg` mantenga su identidad real.
  - Mantiene metadatos como `IncludeAssets`, `ExcludeAssets` y `PrivateAssets` cuando existen.

7. **Usar un feed local temporal**
  - Copia cada `.nupkg` generado a `.artifacts/local-feed`.
  - Permite que los paquetes publicados antes en la misma ejecución estén disponibles inmediatamente para paquetes dependientes, sin esperar a la propagación del feed remoto.

8. **Publicar y finalizar**
  - Ejecuta `dotnet pack` y `dotnet nuget push`.
  - Crea el tag Git correspondiente.
  - Crea la GitHub Release.
  - Si falla la creación de la release tras crear el tag, elimina el tag para evitar inconsistencias.
