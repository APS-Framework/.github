# APS Framework

Infraestructura técnica compartida para aplicaciones .NET sobre Azure.

APS-Framework actúa como organización de plataforma: proporciona librerías, workflows CI/CD y herramientas de IA disponibles para cualquier organización de producto. Apoyarse en estos recursos permite a los equipos **centrarse en el desarrollo de negocio** sin tener que mantener infraestructura técnica transversal propia.

Además, a través del [sdk-mcp-server](https://github.com/APS-Framework/sdk-mcp-server#readme) y su mecanismo de autodiscovery, cualquier organización puede **incorporar asistencia con IA directamente en el entorno de sus desarrolladores**: configurando el servidor con los topics de sus propios repositorios, los agentes de IA (GitHub Copilot, Claude…) acceden al contexto real del código — contratos, documentación y ejemplos — sin salir del IDE.

Cada organización puede también optar por mantener sus propios feeds de paquetes, workflows CI/CD o herramientas, ya sea como alternativa o como extensión de lo que ofrece este repositorio.

---

## Infraestructura compartida

| Repositorio | Descripción |
|---|---|
| [.github](https://github.com/APS-Framework/.github#readme) | Workflows reutilizables (`nuget-ci-publish`, `azure-functions-deploy`), scripts de publicación NuGet y convenciones de documentación para toda la organización. |

---

## Bibliotecas

### Fundamentos

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-common](https://github.com/APS-Framework/sdk-common#readme) | `APS.Common` | Utilidades transversales: jerarquía de excepciones de dominio, helpers y contratos base compartidos por el resto de los SDKs. |
| [sdk-dependency-injection](https://github.com/APS-Framework/sdk-dependency-injection#readme) | `APS.DependencyInjection` | Extensiones de registro en el contenedor de inyección de dependencias. Simplifica el bootstrapping de servicios APS en ASP.NET Core y Azure Functions Isolated Worker. |

### Autenticación

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-auth](https://github.com/APS-Framework/sdk-auth#readme) | `APS.Auth` · `APS.Auth.Google` | Validación de identidad y claims. Incluye soporte para Google ID Tokens sin dependencias externas pesadas. |

### Datos

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-data-cosmos](https://github.com/APS-Framework/sdk-data-cosmos#readme) | `APS.Data.Cosmos` | Cliente de acceso a Azure Cosmos DB: repositorios genéricos, paginación, consultas fluidas y manejo consistente de errores. |
| [sdk-data-blobs](https://github.com/APS-Framework/sdk-data-blobs#readme) | `APS.Data.Blob` | Acceso a Azure Blob Storage a través de `IBlobService`/`BlobService`: carga, descarga, listado y eliminación de blobs. |
| [sdk-data-kusto](https://github.com/APS-Framework/sdk-data-kusto#readme) | `APS.Data.Kusto` | Cliente fluido para Azure Data Explorer (Kusto): ejecución de queries KQL, hydration de resultados y gestión de conexión integrada. |

### Mensajería

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-messaging-mail](https://github.com/APS-Framework/sdk-messaging-mail#readme) | `APS.Messaging.Mail` | Envío de correo con contratos reutilizables y proveedor Twilio SendGrid. Desacopla la lógica de negocio del proveedor de entrega. |
| [sdk-messaging-evengrid](https://github.com/APS-Framework/sdk-messaging-evengrid#readme) | `APS.Messaging.EventGrid` | Publicación de eventos en Azure Event Grid desde aplicaciones .NET. Ofrece un publisher tipado que abstrae el SDK nativo. |

### Integración HTTP

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-service-gateway](https://github.com/APS-Framework/sdk-service-gateway#readme) | `APS.ServiceGateway` | Integración HTTP tipada con Refit, `HttpClientFactory`, Polly (retry/circuit-breaker), Managed Identity, propagación de headers y rutas dinámicas. |

### Observabilidad y resiliencia

| Repositorio | Paquete | Descripción |
|---|---|---|
| [sdk-telemetry](https://github.com/APS-Framework/sdk-telemetry#readme) | `APS.Telemetry` | SDK de telemetría para ASP.NET Core y Azure Functions Isolated Worker: structured logging, trazas distribuidas y métricas sobre Application Insights. |
| [sdk-worker](https://github.com/APS-Framework/sdk-worker#readme) | `APS.Worker` | Middleware de manejo de errores y mapeo de excepciones de dominio a HTTP status codes para Azure Functions Isolated Worker y ASP.NET Core. |

---

## Uso desde otras organizaciones

### Consumir paquetes NuGet

El feed de APS-Framework es público para todas las organizaciones de la plataforma. Para añadirlo como fuente en cualquier repositorio, agregar un `nuget.config` en la raíz:

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

En CI, el secret `APS_NUGET_TOKEN` debe ser un Classic PAT con scope `read:packages` sobre APS-Framework, configurado como org secret en la organización consumidora.

### Usar los workflows reutilizables

Los workflows de [APS-Framework/.github](https://github.com/APS-Framework/.github) son referenciables directamente desde cualquier repositorio de cualquier organización sin necesidad de copiarlos:

```yaml
jobs:
  ci:
    uses: APS-Framework/.github/.github/workflows/nuget-ci-publish.yml@main
    secrets: inherit
```

Cada organización puede también definir los suyos propios o extender los de APS-Framework adaptándolos a sus convenciones.

---

## MCP Server — Asistencia con IA

El [sdk-mcp-server](https://github.com/APS-Framework/sdk-mcp-server#readme) expone las librerías APS.\* a agentes de IA (GitHub Copilot, Claude…) directamente desde el IDE, permitiendo consultar contratos, documentación y ejemplos de cualquier SDK sin salir del chat.

### Cómo funciona

El servidor se instala **una sola vez por máquina** y descubre automáticamente cualquier repositorio que tenga un `mcp-manifest.json` en su raíz, independientemente de la organización a la que pertenezca. Cada repo de producto puede conectar sus propios repos junto con los de APS-Framework en una única sesión de asistencia.

```powershell
npm install -g @aps-framework/sdk-mcp-server
sdk-mcp-server  # escucha en http://127.0.0.1:7512/mcp
```

### Configurar en un repositorio

Añadir `.vscode/mcp.json` en la raíz del repo consumidor. El ejemplo siguiente descubre los SDKs de APS-Framework, los repositorios propios de la organización de producto y excluye el repo actual para que no aparezca en su propia lista de tools:

```json
{
  "servers": {
    "aps-framework": {
      "type": "http",
      "url": "http://127.0.0.1:7512/mcp?discovery=APS-Framework:sdk&discovery=mi-org:mi-topic&exclude=mi-org/nombre-repo-actual"
    }
  }
}
```

El parámetro `discovery` acepta uno o varios topics de GitHub por organización (`org:topic1,topic2`). Para incluir repos de varias organizaciones, repetir el parámetro tantas veces como sea necesario. El repo desde el que se trabaja debe excluirse siempre con `&exclude=org/repo`.

Referencia completa: [sdk-mcp-server README](https://github.com/APS-Framework/sdk-mcp-server#readme).

---

## Publicación NuGet y gestión de versiones

Los paquetes se publican en **GitHub Packages** (`https://nuget.pkg.github.com/APS-Framework/index.json`) siguiendo un flujo de versiones con RC previo a estable:

| Fase | Acción | Resultado |
|---|---|---|
| Desarrollo | `ProjectReference` entre repos | Sin publicación |
| Pre-release | `workflow_dispatch` → modo `rc` | `1.2.0-rc.1`, `1.2.0-rc.2`… |
| Estable | `workflow_dispatch` → modo `stable` | `1.2.0`, tag + GitHub Release |
| Siguiente ciclo | Incrementar `<VersionPrefix>` en `.csproj` | Prepara `1.3.0` |

El workflow [`nuget-ci-publish.yml`](https://github.com/APS-Framework/.github/blob/main/.github/workflows/nuget-ci-publish.yml) gestiona todo el ciclo: resolución de dependencias internas entre paquetes del mismo repo, cálculo automático del número de RC, empaquetado, publicación, creación de tags y generación de GitHub Releases.

### Credenciales necesarias

| Secret | Scope | Propósito |
|---|---|---|
| `APS_NUGET_TOKEN` | Org secret — Classic PAT `read:packages` | Restaurar paquetes de feeds corporativos durante el build y en local. Debe ser Classic PAT porque los fine-grained solo cubren una organización. |
| `NUGET_PUBLISH_TOKEN` | Org secret — Classic o Fine-grained PAT `write:packages` | Publicar paquetes en el feed. Cada organización gestiona el suyo de forma independiente. |
| `NUGET_EXTERNAL_TOKEN` | Secret opcional — `read:packages` del proveedor | Solo necesario si el repo consume paquetes de feeds externos a la empresa. |

Ambos secrets se configuran a nivel de organización y se propagan automáticamente a todos los repos mediante `secrets: inherit` en el workflow caller. `GITHUB_TOKEN` (token automático del runner) **no** funciona para consumir paquetes de otros repositorios de la organización.

### `nuget.config`

Cada repositorio debe incluir un `nuget.config` en la raíz con las fuentes de las que tiene dependencias reales:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="NuGet official package source" value="https://api.nuget.org/v3/index.json" />
    <add key="APS-Framework" value="https://nuget.pkg.github.com/APS-Framework/index.json" />
    <add key="mi-org" value="https://nuget.pkg.github.com/mi-org/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <APS-Framework>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </APS-Framework>
    <mi-org>
      <add key="Username" value="x" />
      <add key="ClearTextPassword" value="%APS_NUGET_TOKEN%" />
    </mi-org>
  </packageSourceCredentials>
</configuration>
```

El `key` en `<packageSourceCredentials>` debe coincidir exactamente con el `key` de `<packageSources>`. Ambos feeds corporativos comparten la misma variable `%APS_NUGET_TOKEN%` — un único Classic PAT con `read:packages` cubre múltiples organizaciones.

En local, `APS_NUGET_TOKEN` se puede obtener reutilizando el token de la sesión activa de `gh`: `gh auth token`.

**Guía completa:** [README-nuget.md](https://github.com/APS-Framework/.github/blob/main/README-nuget.md) — credenciales locales, configuración de un nuevo repo SDK, referencia técnica del workflow y scripts compartidos.

---

## Workflows reutilizables

Los workflows se consumen con `uses: APS-Framework/.github/.github/workflows/<nombre>.yml@main` desde cualquier repositorio de cualquier organización.

### `nuget-ci-publish.yml`

Workflow para repositorios .NET que publican paquetes NuGet en GitHub Packages.

- En cada `push` a `main` o `pull_request`, ejecuta restore, build y test automáticamente.
- La publicación se lanza de forma manual con `workflow_dispatch`, eligiendo entre `rc` (pre-release) o `stable`.
- Soporta repositorios con uno o varios paquetes: el input `packages` acepta nombres separados por comas.
- Calcula versiones automáticamente a partir del `<VersionPrefix>` del `.csproj`, resuelve dependencias internas entre paquetes del mismo repo y genera tags y GitHub Releases tras cada publicación satisfactoria.

> **Publicar con Copilot (recomendado)**
>
> La forma más segura y cómoda de disparar una publicación es pedírselo directamente a GitHub Copilot en modo Agent:
>
> _"Publica una versión RC de los paquetes `MiOrg.Pagos.Proveedor1` y `MiOrg.Pagos.Proveedor2`"_ (dos paquetes del mismo repo)
> _"Publica la versión estable del paquete de autenticación"_ (repo con un solo paquete, sin necesidad de especificar nombre)
>
> Con el MCP server configurado, Copilot conoce los paquetes disponibles en el repositorio, sus versiones actuales y el estado de las últimas publicaciones. A partir de esa información, lanza el workflow con los parámetros correctos, sin que el desarrollador tenga que recordar nombres exactos, inputs ni comandos `gh workflow run`.
>
> El input `packages` solo es necesario en repositorios con varios paquetes — si el repositorio tiene uno solo, puede omitirse.

```yaml
jobs:
  ci-publish:
    uses: APS-Framework/.github/.github/workflows/nuget-ci-publish.yml@main
    with:
      release_type: ${{ inputs.release_type || '' }}
      packages: ${{ inputs.packages || '' }}
    secrets: inherit
```

Referencia completa: [README-nuget.md](https://github.com/APS-Framework/.github/blob/main/README-nuget.md).

---

### `azure-functions-deploy.yml`

Workflow para construir y desplegar Azure Functions (.NET 8, Isolated Worker v4) en Azure.

- Compila y testea el proyecto de Functions, genera los artefactos de publicación y los sube como artifact del workflow.
- Despliega en el entorno indicado usando OIDC (Workload Identity Federation) — sin secretos de credenciales almacenados en el repositorio consumidor.
- Soporta los inputs `environment`, `function_app_name`, `project_path` y `dotnet_version`.

```yaml
jobs:
  deploy:
    uses: APS-Framework/.github/.github/workflows/azure-functions-deploy.yml@main
    with:
      environment: dev
      function_app_name: ${{ vars.FUNCTION_APP_NAME }}
      project_path: src/MiProyecto.API
    secrets: inherit
```

Para que el despliegue funcione correctamente, la identidad de servicio (Service Principal) usada en Azure debe tener una **federated credential** configurada con el subject `repo:<org>/<repo>:environment:<nombre-entorno>`.

Referencia completa: [`azure-functions-deploy.yml`](https://github.com/APS-Framework/.github/blob/main/.github/workflows/azure-functions-deploy.yml).

---

## Recursos

- [Publicación NuGet y gestión de versiones](https://github.com/APS-Framework/.github/blob/main/README-nuget.md)
- [Convenciones de documentación](https://github.com/APS-Framework/.github/blob/main/README-docs.md)
- [Flujo de trabajo con Git](https://github.com/APS-Framework/.github/blob/main/README-git.md)
