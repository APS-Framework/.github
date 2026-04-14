# Documentación de repositorios · Convención APS

> Referencia para generar y mantener la documentación de cualquier repositorio de la organización APS.
> Aplicable tanto a repositorios que publican paquetes NuGet como a runtimes y repos híbridos.

---

## Índice

- [1. Convención de tres documentos](#1-convención-de-tres-documentos)
- [2. Tipos de repositorio](#2-tipos-de-repositorio)
  - [2.1 NuGet-only](#21-nuget-only)
  - [2.2 Runtime-only](#22-runtime-only)
  - [2.3 Híbrido (NuGet + Runtime)](#23-híbrido-nuget--runtime)
- [3. Evaluación previa antes de generar](#3-evaluación-previa-antes-de-generar)
- [4. Contenido de cada documento](#4-contenido-de-cada-documento)
  - [4.1 README.md](#41-readmemd)
  - [4.2 README-sdk.md](#42-readme-sdkmd)
  - [4.3 README-dev.md](#43-readme-devmd)
- [5. Reglas por tipo de proyecto .csproj](#5-reglas-por-tipo-de-proyecto-csproj)
- [6. Configuración del mcp-manifest.json](#6-configuración-del-mcp-manifestjson)
- [7. Plantillas](#7-plantillas)
  - [7.1 Plantilla README.md](#71-plantilla-readmemd)
  - [7.2 Plantilla README-sdk.md](#72-plantilla-readme-sdkmd)
  - [7.3 Plantilla README-dev.md](#73-plantilla-readme-devmd)
  - [7.4 Plantilla mcp-manifest.json](#74-plantilla-mcp-manifestjson)
- [8. Workflow reutilizable: Sync Vector Store Docs](#8-workflow-reutilizable-sync-vector-store-docs)
  - [8.1 Cuándo usarlo](#81-cuándo-usarlo)
  - [8.2 Caller mínimo](#82-caller-mínimo)
  - [8.3 Variables y secrets requeridos](#83-variables-y-secrets-requeridos)
  - [8.4 Inputs expuestos por el workflow](#84-inputs-expuestos-por-el-workflow)
  - [8.5 Semántica de sincronización](#85-semántica-de-sincronización)

---

## 1. Convención de tres documentos

Cada repositorio APS tiene hasta tres ficheros de documentación con roles estrictos y audiencias distintas:

| Fichero | Embebido en `.nupkg` | Visible en GitHub | Audiencia |
|---|---|---|---|
| `README.md` | No | Sí (portada del repo) | Cualquiera — índice mínimo |
| `README-sdk.md` | **Sí** (vía `<PackageReadmeFile>`) | Sí | Desarrolladores que **consumen** el paquete |
| `README-dev.md` | No | Sí (enlazado desde `README.md`) | Desarrolladores que **mantienen** el repo |

**Regla de contenido:** ningún documento debe contener información que pertenezca a otro.
- `README-sdk.md` no menciona despliegues, arquitectura interna ni decisiones de diseño.
- `README-dev.md` no incluye ejemplos de uso del paquete como si fuera un consumidor externo.
- `README.md` no contiene contenido propio — solo describe y enlaza.

---

## 2. Tipos de repositorio

### 2.1 NuGet-only

Repositorio que únicamente publica uno o más paquetes NuGet. No hay ningún proceso desplegable.

**Documentos requeridos:** `README.md`, `README-sdk.md`, `README-dev.md`

Ejemplos: `APS.Data.Cosmos`, `APS.Storage.Blob`, `APS.Messaging.EventGrid`

```
README.md           ← portada: qué es el repo, enlaza a README-sdk.md y README-dev.md
README-sdk.md       ← instalación, configuración, API, ejemplos de uso
README-dev.md       ← arquitectura interna, decisiones de diseño, cómo extender
```

Cuando hay **múltiples paquetes** en el mismo repo (ej: `Abstractions` + `Implementation`), sigue habiendo un único `README-sdk.md` que cubre todos los paquetes publicables y un único `README-dev.md` para el mantenimiento del conjunto.

### 2.2 Runtime-only

Repositorio que despliega un servicio o proceso (Function App, Web App, Worker) sin publicar ningún paquete NuGet.

**Documentos requeridos:** `README.md`, `README-dev.md`

```
README.md           ← portada: qué hace el servicio, enlaza a README-dev.md
README-dev.md       ← arquitectura, despliegue, variables de entorno, debug, extensión
```

`README-sdk.md` **no existe** en repos runtime-only — no hay nada que instalar como paquete.

### 2.3 Híbrido (NuGet + Runtime)

Repositorio con uno o más proyectos desplegables **y** uno o más proyectos que publican un paquete NuGet (típicamente un Service Gateway o cliente SDK del servicio).

**Documentos requeridos:** `README.md`, `README-sdk.md`, `README-dev.md`

Ejemplo típico — Function App con Service Gateway:

```
src/
  APS.{Servicio}.Function/         ← runtime, no publica NuGet
  APS.{Servicio}.Contracts/        ← soporte interno, no publica NuGet
  APS.{Servicio}.Implementations/  ← soporte interno, no publica NuGet
  APS.{Servicio}.ServiceGateway/   ← publica NuGet → PackageReadmeFile=README-sdk.md

README.md           ← portada: describe el servicio y el SDK, enlaza a ambos docs
README-sdk.md       ← cómo usar el ServiceGateway: instalación, configuración, API
README-dev.md       ← cómo mantener la Function y el conjunto del repo
```

---

## 3. Evaluación previa antes de generar o actualizar

### Creación desde cero

Cuando los documentos no existen, explorar el repositorio y evaluar su complejidad conjunta: número de interfaces y métodos públicos, cantidad y tipo de proyectos, naturaleza de los artefactos (runtime, paquetes, híbrido), volumen de DTOs y tipos expuestos.

Si se estima que la documentación resultante va a ser extensa, o que hay decisiones no obvias sobre qué incluir o cómo estructurarlo, **presentar primero un plan al usuario** antes de generar nada:

- Qué documentos se van a crear (`README.md`, `README-sdk.md`, `README-dev.md`)
- Qué secciones va a tener cada uno
- Qué interfaces, DTOs, métodos y tipos se van a documentar
- Si alguna parte requiere una decisión del usuario (ej: nivel de detalle de la API, si documentar tipos internos, si dividir secciones extensas)
- Si hay proyectos publicables, si tienen correctamente configurado `<PackageReadmeFile>README-sdk.md</PackageReadmeFile>` en el `.csproj`

Esperar confirmación o ajustes del usuario antes de escribir ningún fichero.

Si el alcance es claramente acotado y no hay ambigüedad, se puede proceder directamente sin presentar el plan.

### Actualización de documentación existente

Cuando los documentos ya existen y se pide actualizar la documentación (tras añadir o cambiar funcionalidad), el flujo es diferente al de creación:

1. **Leer el código modificado** — identificar qué interfaces, métodos, DTOs o tipos han cambiado o se han añadido
2. **Leer la documentación existente** — comparar con el código actual para detectar qué secciones están desfasadas o incompletas
3. **Presentar al usuario solo las diferencias** antes de escribir nada:
   - Qué secciones hay que actualizar y por qué
   - Qué hay que añadir que no existe aún
   - Qué hay que eliminar que ya no es válido
4. **Aplicar únicamente los cambios identificados** — no regenerar secciones que siguen siendo correctas

**Regla crítica:** en una actualización nunca se reescribe un documento completo salvo que el usuario lo pida explícitamente. El objetivo es mantener el contenido existente válido intacto y aplicar el mínimo cambio necesario.

---

## 4. Contenido de cada documento

### 4.1 README.md

Documento mínimo. No contiene documentación propia — su único propósito es orientar a quien llega al repo.

**Secciones obligatorias:**
- Título y descripción de una línea de qué hace el repo
- Tabla de contenido con los ficheros clave del repo
- Enlace a `README-sdk.md` si existe (con descripción: "para consumidores del paquete")
- Enlace a `README-dev.md` (con descripción: "para mantenedores del repo")

**No incluir:** ejemplos de código, instrucciones de instalación, arquitectura interna, variables de entorno.

### 4.2 README-sdk.md

Orientado exclusivamente a desarrolladores externos que quieren **usar** el paquete. Este documento se embebe dentro del `.nupkg` y es lo que muestra GitHub Packages y el gestor de NuGet de Visual Studio.

**Índice:** obligatorio. Debe incluir un índice con enlaces a todas las secciones de primer nivel al inicio del documento.

**Secciones obligatorias:**

1. **Descripción** — qué problema resuelve el paquete, en qué escenarios usarlo
2. **Instalación** — `PackageReference` con el nombre y versión del paquete; configuración del feed NuGet de APS si es necesaria
3. **Configuración** — cómo registrar en DI (`AddXxx`), qué valores de `appsettings.json` son necesarios, qué variables de entorno requiere
4. **API** — documentación completa de la superficie pública del paquete, con el siguiente formato por elemento:
   - **Interfaces y servicios**: bloque de código con la firma completa de la interfaz; tabla de métodos con nombre, parámetros (nombre, tipo, requerido/opcional, descripción) y tipo de retorno
   - **DTOs y modelos**: tabla de propiedades con nombre, tipo C#, requerida/opcional y descripción; indicar si el modelo es de entrada, salida o ambos
   - **Enums y tipos**: tabla con cada valor, su nombre y su semántica
   - Cada parámetro y propiedad debe tener su tipo C# explícito — nunca `object` ni `dynamic`
   - Los parámetros opcionales deben indicar su valor por defecto
5. **Ejemplos de uso** — fragmentos de código para los casos de uso más frecuentes, listos para copiar y adaptar
6. **Versioning** — tabla de versiones publicadas con cambios relevantes; instrucciones de migración cuando hay breaking changes

**No incluir:** cómo está implementado internamente, cómo desplegar el servicio asociado, decisiones de arquitectura.

### 4.3 README-dev.md

Orientado a desarrolladores que van a **modificar, extender o mantener** el repo. Nunca se embebe en un paquete.

**Índice:** obligatorio si el documento tiene más de tres secciones de primer nivel.

**Regla de no duplicación:** `README-dev.md` no repite ni resume el contenido de `README-sdk.md`. Si el repo tiene `README-sdk.md`, el documento debe abrirse con un enlace explícito a él:

```markdown
> Para la API pública, instalación y ejemplos de uso del paquete, ver [README-sdk.md](README-sdk.md).
```

**Secciones obligatorias:**

1. **Arquitectura** — diagrama o descripción de los proyectos del repo y cómo se relacionan; responsabilidades de cada capa
2. **Dependencias externas** — SDKs de terceros utilizados, para qué se usan y qué abstraen; versiones relevantes
3. **Decisiones de diseño** — patrones aplicados y por qué; alternativas descartadas si son relevantes para el mantenimiento futuro
4. **Configuración del entorno local** — requisitos previos, variables de entorno necesarias para ejecutar o depurar localmente
5. **Tests** — cómo ejecutar los tests; qué tipos de tests existen (unitarios, integración, e2e); mocks y dependencias de test
6. **Cómo extender** — guía para añadir nuevas funcionalidades respetando la arquitectura: nuevos comandos, nuevos providers, nuevas rutas, etc.

**Secciones adicionales para repos con runtime:**

7. **Despliegue** — entornos disponibles, cómo lanzar un despliegue, qué workflows de GitHub Actions están implicados
8. **Variables de entorno en producción** — tabla completa de variables requeridas por entorno, con descripción y ejemplo de valor
9. **Debug y troubleshooting** — cómo conectarse a logs, diagnósticos frecuentes, errores conocidos y su solución

**No incluir:** API pública, ejemplos de uso del paquete como consumidor externo, instrucciones de instalación del NuGet — todo eso pertenece a `README-sdk.md`.

---

## 5. Reglas por tipo de proyecto .csproj

La propiedad `<PackageReadmeFile>` del `.csproj` **solo aplica a proyectos que publican un paquete NuGet consumible externamente.** En cualquier otro caso no debe aparecer.

| Tipo de proyecto | `<PackageReadmeFile>` |
|---|---|
| Publica un paquete NuGet (ServiceGateway, librería SDK) | `README-sdk.md` |
| Runtime desplegable (Function App, Web App, Worker) | — (no se incluye) |
| Soporte interno no publicado (Contracts, Abstractions, Implementations) | — (no se incluye) |

Configuración en el `.csproj` para proyectos publicables:

```xml
<PropertyGroup>
  <PackageId>APS.{NombrePaquete}</PackageId>
  <VersionPrefix>1.0.0</VersionPrefix>
  <PackageReadmeFile>README-sdk.md</PackageReadmeFile>
</PropertyGroup>

<ItemGroup>
  <None Include="..\..\README-sdk.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

> El `README-sdk.md` está en la raíz del repo. La ruta relativa desde el `.csproj` dentro de `src/APS.{Paquete}/` es `..\..\README-sdk.md`. Ajustar si la estructura de carpetas es diferente.

---

## 6. Configuración del mcp-manifest.json

El `mcp-manifest.json` de cada repositorio debe tener dos tools de documentación además de los tools específicos del dominio:

| Tool | Cuándo invocarlo | Ficheros |
|---|---|---|
| `sdk` | Crear o actualizar `README-sdk.md`; generar ejemplos de uso; describir la API del paquete | `README-sdk.md` |
| `dev` | Mantener el repo; entender la arquitectura; configurar el entorno local; depurar; desplegar | `README-dev.md` |

**Regla:** los tools `sdk` y `dev` **solo referencian su fichero respectivo**. No se cruzan. Copilot carga únicamente el contexto necesario para la tarea en curso.

Si el repo es runtime-only y no tiene `README-sdk.md`, el tool `sdk` no se incluye en el manifest.

---

## 7. Plantillas

### 7.1 Plantilla README.md

```markdown
# APS.{NombrePaquete} · {Descripción breve en una línea}

## Documentación

| Documento | Descripción |
|---|---|
| [README-sdk.md](README-sdk.md) | Instalación, configuración y uso del paquete NuGet |
| [README-dev.md](README-dev.md) | Arquitectura, mantenimiento y despliegue del repo |

## Contenido del repositorio

| Proyecto | Descripción |
|---|---|
| `src/APS.{Paquete1}/` | _{descripción}_ |
| `src/APS.{Paquete2}/` | _{descripción}_ |
```

Para repos **runtime-only** (sin NuGet), omitir la fila de `README-sdk.md`:

```markdown
# APS.{NombreServicio} · {Descripción breve en una línea}

## Documentación

| Documento | Descripción |
|---|---|
| [README-dev.md](README-dev.md) | Arquitectura, mantenimiento y despliegue del servicio |
```

### 7.2 Plantilla README-sdk.md

```markdown
# APS.{NombrePaquete}

> {Descripción del paquete: qué problema resuelve y cuándo usarlo.}

## Índice

- [Instalación](#instalación)
- [Configuración](#configuración)
- [API](#api)
  - [{NombreInterfazPrincipal}](#inombreprincipal)
  - [Modelos](#modelos)
  - [Tipos y enums](#tipos-y-enums)
- [Ejemplos de uso](#ejemplos-de-uso)
- [Versioning](#versioning)

---

## Instalación

```xml
<PackageReference Include="APS.{NombrePaquete}" Version="{versión}" />
```

El paquete se distribuye en el feed NuGet privado de APS. Si no tienes el feed configurado,
consulta la [guía de configuración de NuGet](https://github.com/APS-Framework/.github/blob/main/README-nuget.md#2-configurar-credenciales-entorno-local).

---

## Configuración

### Registro en DI

```csharp
builder.Services.Add{NombrePaquete}(builder.Configuration);
```

### appsettings.json

```json
{
  "{NombrePaquete}": {
    "Propiedad1": "{valor}",
    "Propiedad2": "{valor}"
  }
}
```

| Propiedad | Tipo | Requerida | Descripción |
|---|---|---|---|
| `Propiedad1` | `string` | Sí | _{descripción}_ |
| `Propiedad2` | `int` | No | _{descripción}. Default: `{valor}`_ |

---

## API

### `I{NombrePrincipal}`

```csharp
public interface I{NombrePrincipal}
{
    Task<{TResultado}> {Metodo1Async}({TParam1} param1, CancellationToken ct = default);
    Task {Metodo2Async}({TParam2} param2, CancellationToken ct = default);
}
```

| Método | Parámetros | Retorno | Descripción |
|---|---|---|---|
| `{Metodo1Async}` | `param1: {TParam1}` | `Task<{TResultado}>` | _{qué hace}_ |
| `{Metodo2Async}` | `param2: {TParam2}` | `Task` | _{qué hace}_ |

#### Parámetros de `{Metodo1Async}`

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `param1` | `{TParam1}` | Sí | _{descripción}_ |
| `ct` | `CancellationToken` | No | Token de cancelación. Default: `default` |

### Modelos

#### `{NombreDto}` _(entrada / salida / ambos)_

| Propiedad | Tipo | Requerida | Descripción |
|---|---|---|---|
| `Propiedad1` | `string` | Sí | _{descripción}_ |
| `Propiedad2` | `int?` | No | _{descripción}_ |

### Tipos y enums

#### `{NombreEnum}`

| Valor | Descripción |
|---|---|
| `{Valor1}` | _{semántica}_ |
| `{Valor2}` | _{semántica}_ |

---

## Ejemplos de uso

### {Caso de uso frecuente 1}

```csharp
// ejemplo completo y ejecutable
```

### {Caso de uso frecuente 2}

```csharp
// ejemplo completo y ejecutable
```

---

## Versioning

| Versión | Cambios |
|---|---|
| `{versión actual}` | _{descripción}_ |
| `{versión anterior}` | _{descripción}_ |
```

### 7.3 Plantilla README-dev.md

```markdown
# APS.{NombreRepo} · Guía de mantenimiento

> Documentación interna para desarrolladores que mantienen este repositorio.
> Para la API pública, instalación y ejemplos de uso del paquete, ver [README-sdk.md](README-sdk.md).

## Índice

- [Arquitectura](#arquitectura)
- [Dependencias externas](#dependencias-externas)
- [Decisiones de diseño](#decisiones-de-diseño)
- [Configuración del entorno local](#configuración-del-entorno-local)
- [Tests](#tests)
- [Cómo extender](#cómo-extender)
<!-- Si hay runtime, añadir: -->
- [Despliegue](#despliegue)
- [Variables de entorno en producción](#variables-de-entorno-en-producción)
- [Debug y troubleshooting](#debug-y-troubleshooting)

---

## Arquitectura

{Descripción de los proyectos del repo y sus responsabilidades.}

| Proyecto | Tipo | Descripción |
|---|---|---|
| `APS.{Paquete1}` | Librería NuGet | _{descripción}_ |
| `APS.{Paquete2}` | Librería NuGet | _{descripción}_ |
| `APS.{Servicio}.Function` | Runtime | _{descripción}_ |

---

## Dependencias externas

| Dependencia | Versión | Uso en este repo |
|---|---|---|
| `{SDK o paquete}` | `{versión}` | _{para qué se usa y qué abstrae}_ |

---

## Decisiones de diseño

### {Decisión 1}

_{Por qué se tomó esta decisión. Alternativas consideradas si son relevantes.}_

---

## Configuración del entorno local

### Requisitos previos

- {Requisito 1}
- {Requisito 2}

### Variables de entorno

```powershell
$env:{VARIABLE_1} = "{valor de ejemplo}"
$env:{VARIABLE_2} = "{valor de ejemplo}"
```

---

## Tests

```powershell
# Ejecutar todos los tests
dotnet test

# Ejecutar solo tests de integración
dotnet test --filter Category=Integration
```

{Descripción de los tipos de tests existentes y qué cubren.}

---

## Cómo extender

### Añadir {nueva funcionalidad}

_{Pasos y consideraciones para extender el repo respetando la arquitectura.}_

---

<!-- Incluir si el repo tiene runtime -->
## Despliegue

### Entornos

| Entorno | Descripción |
|---|---|
| `dev` | _{descripción}_ |
| `pro` | _{descripción}_ |

### Lanzar un despliegue

```powershell
gh workflow run deploy.yml -f environment=dev
```

---

## Variables de entorno en producción

| Variable | Requerida | Descripción |
|---|---|---|
| `{VARIABLE_1}` | Sí | _{descripción}_ |
| `{VARIABLE_2}` | No | _{descripción. Default: `{valor}`}_ |

---

## Debug y troubleshooting

### {Problema frecuente 1}

_{Síntoma, causa y solución.}_
```

### 7.4 Plantilla mcp-manifest.json

Para repos **NuGet-only** o **Híbridos**:

```json
{
  "name": "APS-Framework/{nombre-repo}",
  "description": "{Descripción del repo y sus paquetes.}",
  "toolNamespace": "{namespace}",
  "replaces": [],
  "packages": [],
  "entryPoints": [],
  "tools": [
    {
      "name": "sdk",
      "description": "Obtener documentación para consumir el paquete APS.{NombrePaquete}: instalación, configuración, API y ejemplos de uso. Llamar SIEMPRE que se añada un PackageReference de APS.{NombrePaquete}, se generen ejemplos de uso, se configure el paquete en DI o se consulte la API disponible.",
      "files": [
        "README-sdk.md"
      ]
    },
    {
      "name": "dev",
      "description": "Obtener documentación de mantenimiento del repo APS.{NombreRepo}: arquitectura interna, dependencias, decisiones de diseño, configuración del entorno local y cómo extender. Llamar SIEMPRE que se vaya a modificar el repo, añadir funcionalidad, configurar el entorno de desarrollo o depurar.",
      "files": [
        "README-dev.md"
      ]
    }
  ]
}
```

Para repos **Runtime-only** (sin paquete NuGet, omitir tool `sdk`):

```json
{
  "name": "APS-Framework/{nombre-repo}",
  "description": "{Descripción del servicio.}",
  "toolNamespace": "{namespace}",
  "replaces": [],
  "packages": [],
  "entryPoints": [],
  "tools": [
    {
      "name": "dev",
      "description": "Obtener documentación de mantenimiento del servicio APS.{NombreServicio}: arquitectura, despliegue, variables de entorno, debug y cómo extender. Llamar SIEMPRE que se vaya a modificar el servicio, configurar el entorno de desarrollo, depurar o lanzar un despliegue.",
      "files": [
        "README-dev.md"
      ]
    }
  ]
}
```

---

## 8. Workflow reutilizable: Sync Vector Store Docs

El workflow `.github/workflows/sync-vector-docs.yml` centraliza la sincronización de documentación operativa en Markdown hacia un vector store compartido en Azure AI Foundry.

### 8.1 Cuándo usarlo

Usarlo cuando un repositorio mantiene documentación bajo una ruta tipo `src/.../ops-docs/**/*.md` y esa documentación debe estar disponible en un vector store para búsqueda semántica o entrenamiento de asistentes.

No sustituye a `README.md`, `README-sdk.md` o `README-dev.md`: este workflow solo sincroniza los ficheros `.md` seleccionados por el caller.

### 8.2 Caller mínimo

Caller manual con `workflow_dispatch`:

```yaml
name: Sync ops-docs

on:
  workflow_dispatch:
    inputs:
      file_filter:
        description: Glob repo-relativo a sincronizar
        required: true
        type: string
        default: src/MyLib/ops-docs/**/*.md

jobs:
  sync-docs:
    uses: APS-Framework/.github/.github/workflows/sync-vector-docs.yml@main
    with:
      file_filter: ${{ inputs.file_filter }}
      docs_prefix: RSB
    secrets: inherit
```

Caller típico con disparo automático en `push` y posibilidad de override manual:

```yaml
name: Sync ops-docs

on:
  push:
    branches:
      - main
    paths:
      - 'src/MyLib/ops-docs/**/*.md'
  workflow_dispatch:
    inputs:
      file_filter:
        description: Override opcional del glob repo-relativo
        required: false
        type: string

jobs:
  sync-docs:
    uses: APS-Framework/.github/.github/workflows/sync-vector-docs.yml@main
    with:
      file_filter: ${{ inputs.file_filter || vars.OPS_DOCS_FILE_FILTER }}
      docs_prefix: RSB
    secrets: inherit
```

### 8.3 Variables y secrets requeridos

El workflow reutilizable consume estas credenciales desde el repositorio caller:

| Origen | Nombre | Requerido | Descripción |
|---|---|---|---|
| `vars` | `FOUNDRY_ENDPOINT_SBX` | Sí | URL base del endpoint Azure AI Foundry |
| `vars` | `VS_TRAINER_ID_SBX` | Sí | ID del vector store de destino |
| `vars` | `OPS_DOCS_FILE_FILTER` | No | Glob repo-relativo por defecto para callers que disparan en `push` |
| `secrets` | `FOUNDRY_API_KEY_SBX` | Sí | API key del endpoint Azure AI Foundry |

El caller debe propagar los secrets con `secrets: inherit`.

### 8.4 Inputs expuestos por el workflow

| Input | Tipo | Requerido | Default | Uso |
|---|---|---|---|---|
| `file_filter` | `string` | Sí | — | Glob repo-relativo de documentos Markdown a sincronizar |
| `docs_prefix` | `string` | No | `RSB` | Prefijo lógico del repositorio dentro del vector store |
| `force_all` | `boolean` | No | `false` | Fuerza una resincronización completa, re-subiendo todos los documentos |
| `migrate_unscoped_legacy` | `boolean` | No | `false` | Migra nombres legacy sin prefijo al formato canónico actual |
| `confirm_large_sync` | `boolean` | No | `false` | Confirmación explícita requerida cuando el glob resuelve más de 200 ficheros |

Restricciones importantes:

- `file_filter` es obligatorio. Si el caller quiere un valor por defecto, debe resolverlo en su propio workflow, por ejemplo con `vars.OPS_DOCS_FILE_FILTER`.
- `docs_prefix` no puede ser vacío.
- Solo se sincronizan ficheros con extensión `.md`, aunque el glob coincida con otros paths.

### 8.5 Semántica de sincronización

El workflow converge el vector store al estado actual del repositorio con estas reglas:

1. Descubre todos los ficheros `.md` que cumplen `file_filter` y calcula su `sha256`.
2. Sube cada documento con nombre canónico `opsdocs::{docs_prefix}/{ruta/relativa}`.
3. Lista los adjuntos actuales del vector store y reconoce varios formatos previos:
   - prefijo canónico actual (`opsdocs::{docs_prefix}/...`)
   - formato antiguo scopeado por repositorio (`opsdocs::{org__repo}::...`)
   - formato legacy sin prefijo, solo si `migrate_unscoped_legacy=true`
4. Elimina del vector store los documentos gestionados por este repositorio que ya no existen en Git.
5. Si ya existe una copia actual con contenido idéntico y nombre canónico, la conserva y elimina duplicados stale.
6. Si el contenido cambió o el nombre antiguo no es canónico, sube primero la nueva copia y solo después elimina la anterior.
7. Deja intactas las entradas no gestionadas por el repositorio caller.

Comportamiento operativo adicional:

- Si `file_filter` resuelve más de 200 documentos y `confirm_large_sync=false`, el workflow falla para evitar sincronizaciones masivas accidentales.
- El resumen final se publica en `GITHUB_STEP_SUMMARY` con conteos de `created`, `updated`, `deleted`, `unchanged`, duración y número de errores.
- Cualquier error de metadata, lectura de contenido, subida o detach hace fallar la ejecución.
