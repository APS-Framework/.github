# NuGet · Publicación y Consumo

> Referencia para desarrolladores y pipelines CI/CD.
> Aplicable a cualquier repositorio que publique paquetes NuGet en GitHub Packages.

---

## Índice

- [1. Feed de la organización](#1-feed-de-la-organización)
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
  - [6.4 Secrets de CI/CD](#64-secrets-de-cicd)
- [7. Referencia técnica del workflow](#7-referencia-técnica-del-workflow)
  - [7.1 `.github/workflows/nuget-ci-publish.yml`](#71-githubworkflowsnuget-ci-publishyml)
  - [7.2 Especificación de steps](#72-especificación-de-steps)
- [8. Scripts compartidos](#8-scripts-compartidos)
  - [8.1 `.github/scripts/nuget_publish.py`](#81-githubscriptsnuget_publishpy)
  - [8.2 Responsabilidades internas del script](#82-responsabilidades-internas-del-script)

---

## 1. Feed de la organización

Cada repositorio publica sus paquetes en el feed de GitHub Packages de su propia organización:

```
https://nuget.pkg.github.com/{tu-org}/index.json
```

Los nombres de paquete siguen la convención de la organización publicadora (p.ej. `APS.*` para APS-Framework, `CS.*` para CS-Level).

Los paquetes disponibles en cada repositorio se listan en su propio `README.md`. Un repositorio puede publicar uno o múltiples paquetes:

| Paquete | Descripción |
|---|---|
| `{org}.{Paquete1}` | _{descripción del paquete 1}_ |
| `{org}.{Paquete2}` | _{descripción del paquete 2}_ |

---

## 2. Configurar credenciales (entorno local)

GitHub Packages requiere autenticación para consumir paquetes desde local. No es necesario generar ningún PAT manualmente — se reutiliza el token de la sesión `gh` activa.

> **Para los secrets usados en CI/CD** (publicación desde GitHub Actions), ver sección [6.4](#64-secrets-de-cicd). Esos PATs los gestiona únicamente el administrador de la organización.

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

> Incluir únicamente los feeds de los que el proyecto tiene dependencias reales. Si solo se consumen paquetes de NuGet.org, no es necesaria ninguna fuente privada.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGet official package source" value="https://api.nuget.org/v3/index.json" />
    <!-- Una entrada por cada organización de la que se consuman paquetes privados -->
    <add key="MiOrg" value="https://nuget.pkg.github.com/MiOrg/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <MiOrg>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </MiOrg>
  </packageSourceCredentials>
</configuration>
```

> Sustituir `MiOrg` por el nombre real de la organización en los tres lugares donde aparece. El nombre en `<packageSourceCredentials>` debe coincidir exactamente con el atributo `key` de `<packageSources>`.

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

> **Importante:** `GITHUB_TOKEN` en GitHub Actions solo tiene acceso a los paquetes publicados por el **propio repositorio**. Para consumir paquetes de otros repositorios de la organización (p.ej. un paquete publicado por otro repo de la misma org), se obtiene un **403 Forbidden**. Es obligatorio usar el secret de organización `APS_NUGET_TOKEN`.

El secret `APS_NUGET_TOKEN` se configura a nivel de organización (ver sección [6.4](#64-secret-de-organización-APS_nuget_token)) y se propaga automáticamente a todos los repos mediante `secrets: inherit` en el workflow caller.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v5

      - name: Setup .NET
        uses: actions/setup-dotnet@v5
        with:
          dotnet-version: '{dotnet-version}'   # ej: 10.x

      - name: Restore
        run: dotnet restore
        env:
          APS_NUGET_TOKEN: ${{ secrets.APS_NUGET_TOKEN }}

      - name: Build
        run: dotnet build --no-restore -c Release
```

> **Nota sobre runners self-hosted:** las versiones actuales de `checkout`, `setup-dotnet` y `setup-python` usan runtime Node 24. Si no usas runners hospedados por GitHub, actualiza tu runner self-hosted al menos a la versión `v2.327.1` para evitar fallos al arrancar el job.

> **Para despliegues a Azure** (App Service, Container Apps, Functions): el build y el restore ocurren en el runner de GitHub Actions, no en Azure. Azure recibe los binarios ya compilados. El feed NuGet nunca necesita ser accesible desde Azure.

---

## 5. Referencia rápida de credenciales

| Escenario | Secret / Variable | Tipo de token | Scope requerido |
|---|---|---|---|
| Consumir en local | Variable de usuario `APS_NUGET_TOKEN` (token de sesión `gh`) | Classic PAT | `read:packages` |
| Consumir en GitHub Actions (feeds corporativos) | Org secret `APS_NUGET_TOKEN` | Classic PAT | `read:packages` |
| Consumir en GitHub Actions (feeds externos) | Secret `NUGET_EXTERNAL_TOKEN` | Classic PAT o Fine-grained | `read:packages` del proveedor |
| Publicar desde GitHub Actions | Secret `NUGET_PUBLISH_TOKEN` | Classic o Fine-grained de la org publicadora | `write:packages` |
| Azure en runtime | — | — | No necesario |

> `GITHUB_TOKEN` **no funciona** para consumir paquetes de otros repositorios de la organización. Solo da acceso a los paquetes del repo donde se ejecuta el workflow.
>
> `APS_NUGET_TOKEN` debe ser un **Classic PAT** porque los fine-grained PATs de GitHub solo tienen alcance a una única organización. Un Classic PAT con `read:packages` puede leer de múltiples orgs corporativas con un solo token.

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

Crea el `nuget.config` en la raíz del nuevo repositorio. **Incluir solo las fuentes de las que el proyecto tiene dependencias reales** — si el proyecto no consume paquetes privados, basta con NuGet.org.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGet official package source" value="https://api.nuget.org/v3/index.json" />
    <!-- Añadir una entrada por cada organización de la que este repositorio consume paquetes privados.
         El key puede ser cualquier nombre descriptivo; por convención se usa el nombre de la org.
         Borrar este comentario si no hay dependencias de feeds corporativos. -->
    <!-- <add key="OrgPublicadora" value="https://nuget.pkg.github.com/OrgPublicadora/index.json" /> -->
    <!-- Duplicar la línea anterior por cada org corporativa adicional. -->
    <!-- Fuentes externas a la empresa (usan %NUGET_EXTERNAL_TOKEN%): -->
    <!-- <add key="VendorExterno" value="https://nuget.pkg.github.com/VendorExterno/index.json" /> -->
  </packageSources>
  <packageSourceCredentials>
    <!-- Un bloque por cada entrada privada de packageSources. El nombre debe coincidir
         exactamente con el atributo key. Borrar si no hay feeds privados. -->
    <!-- <OrgPublicadora>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </OrgPublicadora> -->
    <!-- <VendorExterno>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%NUGET_EXTERNAL_TOKEN%" />
    </VendorExterno> -->
  </packageSourceCredentials>
</configuration>
```

**Regla práctica para decidir qué feeds añadir:**

| Dependencia del proyecto | Entrada en `nuget.config` | Variable |
|---|---|---|
| Solo paquetes de NuGet.org | Ninguna fuente privada | — |
| Paquetes de una org corporativa (ej. `OrgA.Paquete`) | `<add key="OrgA" value="https://nuget.pkg.github.com/OrgA/index.json" />` | `%APS_NUGET_TOKEN%` |
| Paquetes de varias orgs corporativas | Una entrada `<add>` por org | `%APS_NUGET_TOKEN%` (mismo token, cubre todas las orgs corporativas) |
| Paquetes de un proveedor externo | `<add key="Vendor" value="..." />` | `%NUGET_EXTERNAL_TOKEN%` |

**Convención de variables por tipo de fuente:**

| Tipo de fuente | Variable en `ClearTextPassword` | Secret en CI |
|---|---|---|
| Organizaciones corporativas | `%APS_NUGET_TOKEN%` | `APS_NUGET_TOKEN` (org secret de la organización) |
| Proveedores externos a la compañía | `%NUGET_EXTERNAL_TOKEN%` | `NUGET_EXTERNAL_TOKEN` (secret por repo/org) |

> El `nuget.config` controla qué fuentes activa cada repositorio. El workflow centralizado inyecta ambas variables en el step `Restore` — si una fuente no está en el `nuget.config`, la variable correspondiente se ignora aunque esté presente.

---

### 6.4 Secrets de CI/CD

El workflow reutilizable requiere dos secrets y acepta uno opcional. Todos se propagan al
workflow centralizado mediante `secrets: inherit` en el caller.

#### `APS_NUGET_TOKEN` (requerido)

Classic PAT con scope `read:packages`. Otorga acceso de lectura a todos los feeds NuGet
privados de las organizaciones corporativas de la empresa. Se usa en el step `Restore` de
build y publish.

> Debe ser un **Classic PAT** — los fine-grained PATs solo tienen alcance a una única
> organización y no pueden cubrir múltiples orgs corporativas.

Configuración inicial (una sola vez), por el administrador de la organización:

1. Genera un Classic PAT con scope `read:packages`.
2. Configúralo como org secret en la organización, visible para todos sus repos:

```powershell
"ghp_TU_TOKEN_LECTURA" | gh secret set APS_NUGET_TOKEN --org {tu-org} --visibility all
```

> Cada organización que publique o consuma paquetes privados debe configurar este secret
> en su propia org para que sus repos lo reciban automáticamente vía `secrets: inherit`.

#### `NUGET_PUBLISH_TOKEN` (requerido)

PAT con scope `write:packages` de la organización publicadora. Se usa exclusivamente para
`dotnet nuget push`.

Cada organización gestiona el suyo de forma independiente. Puede ser un Classic PAT o un
Fine-grained PAT con `write:packages` sobre esa org.

```powershell
# Configurar en la organización que publica los paquetes
"ghp_TU_TOKEN_ESCRITURA" | gh secret set NUGET_PUBLISH_TOKEN --org {tu-org} --visibility all
```

#### `NUGET_EXTERNAL_TOKEN` (opcional)

PAT para feeds NuGet privados de proveedores externos a la compañía. Solo necesario en
repos que referencien fuentes externas en su `nuget.config` bajo la variable
`%NUGET_EXTERNAL_TOKEN%`. Configurar como secret en el repo o en la org correspondiente.

**Renovación:** cuando cualquier PAT expire, genera uno nuevo y actualiza el secret
correspondiente. No es necesario tocar ningún repositorio individual.

---

## 7. Referencia técnica del workflow

### 7.1 `.github/workflows/nuget-ci-publish.yml`

Este es el workflow reutilizable oficial para publicación NuGet en la organización. Expone estos inputs:

| Input | Tipo | Descripción |
|---|---|---|
| `release_type` | `string` | `rc`, `stable` o vacío. Vacío ejecuta solo CI. |
| `packages` | `string` | Uno o varios `PackageId` separados por comas. Si el repositorio solo tiene un paquete, puede omitirse. |

Y estos secrets:

| Secret | Requerido | Uso |
|---|---|---|
| `APS_NUGET_TOKEN` | Sí | Classic PAT `read:packages`. Restore de feeds corporativos en build y publish. |
| `NUGET_PUBLISH_TOKEN` | Sí | PAT `write:packages` de la org publicadora. Solo para `dotnet nuget push`. |
| `NUGET_EXTERNAL_TOKEN` | No | PAT para feeds externos. Solo si el `nuget.config` lo referencia. |

| Job | Cuándo se ejecuta | Objetivo |
|---|---|---|
| `build` | Siempre | Ejecutar `restore`, `build` y `test` del repositorio. |
| `publish` | Solo si `release_type != ''` | Publicar paquetes NuGet, crear tags y generar releases. |

### 7.2 Especificación de steps

Las siguientes tablas enumeran todos los steps reales del workflow en el orden exacto en que se ejecutan.

#### Job `build`

| Orden | Step | Tipo | Descripción breve |
|---|---|---|---|
| 1 | `actions/checkout@v5` | Action | Descarga el repositorio caller en el runner para ejecutar CI sobre su código real. |
| 2 | `Setup .NET` | Action (`actions/setup-dotnet@v5`) | Prepara el SDK .NET soportado por la organización, actualmente `8.x` y `10.x`. |
| 3 | `Restore` | Shell step | Ejecuta `dotnet restore` con `APS_NUGET_TOKEN` para restaurar dependencias privadas cross-repo. |
| 4 | `Build` | Shell step | Ejecuta `dotnet build --no-restore -c Release` para validar compilación completa. |
| 5 | `Test` | Shell step | Ejecuta `dotnet test --no-build -c Release`; si falla, no se publica nada. |

#### Job `publish`

| Orden | Step | Tipo | Descripción breve |
|---|---|---|---|
| 1 | `actions/checkout@v5` | Action | Descarga el repositorio caller con `fetch-depth: 0` para poder inspeccionar y crear tags Git. |
| 2 | `Checkout shared workflow scripts` | Action (`actions/checkout@v5`) | Descarga `APS-Framework/.github` en `.APS-github` para disponer de scripts compartidos del repositorio central. |
| 3 | `Setup .NET` | Action (`actions/setup-dotnet@v5`) | Prepara el SDK .NET para el empaquetado y la publicación. |
| 4 | `Setup Python` | Action (`actions/setup-python@v6`) | Garantiza una versión estable de Python para ejecutar el script de publicación. |
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
