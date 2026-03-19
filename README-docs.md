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
- [3. Contenido de cada documento](#3-contenido-de-cada-documento)
  - [3.1 README.md](#31-readmemd)
  - [3.2 README-sdk.md](#32-readme-sdkmd)
  - [3.3 README-dev.md](#33-readme-devmd)
- [4. Reglas por tipo de proyecto .csproj](#4-reglas-por-tipo-de-proyecto-csproj)
- [5. Configuración del mcp-manifest.json](#5-configuración-del-mcp-manifestjson)
- [6. Plantillas](#6-plantillas)
  - [6.1 Plantilla README.md](#61-plantilla-readmemd)
  - [6.2 Plantilla README-sdk.md](#62-plantilla-readme-sdkmd)
  - [6.3 Plantilla README-dev.md](#63-plantilla-readme-devmd)
  - [6.4 Plantilla mcp-manifest.json](#64-plantilla-mcp-manifestjson)

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

## 3. Contenido de cada documento

### 3.1 README.md

Documento mínimo. No contiene documentación propia — su único propósito es orientar a quien llega al repo.

**Secciones obligatorias:**
- Título y descripción de una línea de qué hace el repo
- Tabla de contenido con los ficheros clave del repo
- Enlace a `README-sdk.md` si existe (con descripción: "para consumidores del paquete")
- Enlace a `README-dev.md` (con descripción: "para mantenedores del repo")

**No incluir:** ejemplos de código, instrucciones de instalación, arquitectura interna, variables de entorno.

### 3.2 README-sdk.md

Orientado exclusivamente a desarrolladores externos que quieren **usar** el paquete. Este documento se embebe dentro del `.nupkg` y es lo que muestra GitHub Packages y el gestor de NuGet de Visual Studio.

**Secciones obligatorias:**

1. **Descripción** — qué problema resuelve el paquete, en qué escenarios usarlo
2. **Instalación** — `PackageReference` con el nombre y versión del paquete; configuración del feed NuGet de APS si es necesaria
3. **Configuración** — cómo registrar en DI (`AddXxx`), qué valores de `appsettings.json` son necesarios, qué variables de entorno requiere
4. **API principal** — interfaz o clase de entrada, métodos más importantes con firma y descripción breve
5. **Ejemplos de uso** — fragmentos de código para los casos de uso más frecuentes, listos para copiar y adaptar
6. **Versioning** — tabla de versiones publicadas con cambios relevantes; instrucciones de migración cuando hay breaking changes

**No incluir:** cómo está implementado internamente, cómo desplegar el servicio asociado, decisiones de arquitectura.

### 3.3 README-dev.md

Orientado a desarrolladores que van a **modificar, extender o mantener** el repo. Nunca se embebe en un paquete.

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

**No incluir:** ejemplos de uso del paquete como consumidor externo, instrucciones de instalación del NuGet.

---

## 4. Reglas por tipo de proyecto .csproj

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

## 5. Configuración del mcp-manifest.json

El `mcp-manifest.json` de cada repositorio debe tener dos tools de documentación además de los tools específicos del dominio:

| Tool | Cuándo invocarlo | Ficheros |
|---|---|---|
| `sdk` | Crear o actualizar `README-sdk.md`; generar ejemplos de uso; describir la API del paquete | `README-sdk.md` |
| `dev` | Mantener el repo; entender la arquitectura; configurar el entorno local; depurar; desplegar | `README-dev.md` |

**Regla:** los tools `sdk` y `dev` **solo referencian su fichero respectivo**. No se cruzan. Copilot carga únicamente el contexto necesario para la tarea en curso.

Si el repo es runtime-only y no tiene `README-sdk.md`, el tool `sdk` no se incluye en el manifest.

---

## 6. Plantillas

### 6.1 Plantilla README.md

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

### 6.2 Plantilla README-sdk.md

```markdown
# APS.{NombrePaquete}

> {Descripción del paquete: qué problema resuelve y cuándo usarlo.}

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

| Propiedad | Requerida | Descripción |
|---|---|---|
| `Propiedad1` | Sí | _{descripción}_ |
| `Propiedad2` | No | _{descripción}. Default: `{valor}`_ |

---

## API principal

### `I{NombrePrincipal}`

```csharp
// Operación principal
Task<{Resultado}> {Metodo1Async}({Param} param, CancellationToken ct = default);

// Operación secundaria
Task {Metodo2Async}({Param} param, CancellationToken ct = default);
```

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

### 6.3 Plantilla README-dev.md

```markdown
# APS.{NombreRepo} · Guía de mantenimiento

> Documentación interna para desarrolladores que mantienen este repositorio.
> Para consumir el paquete, ver [README-sdk.md](README-sdk.md).

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

### 6.4 Plantilla mcp-manifest.json

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
