# APS/.github

Repositorio de configuraciones y workflows compartidos de la organización APS.

## Contenido

| Fichero | Descripción |
|---|---|
| `.github/workflows/nuget-ci-publish.yml` | Workflow reutilizable para CI y publicación de paquetes NuGet en GitHub Packages |
| `.github/workflows/azure-functions-deploy.yml` | Workflow reutilizable para compilar y desplegar Azure Functions (Isolated Worker v4) |
| `.github/workflows/container-app-deploy.yml` | Workflow reutilizable para registrar una imagen Docker en ACR y desplegar en Azure Container Apps |
| `.github/workflows/sync-vector-docs.yml` | Workflow reutilizable para sincronizar `ops-docs` Markdown con un vector store compartido |
| `.github/scripts/nuget_publish.py` | Orquestador compartido para publicación multi-paquete NuGet con resolución de dependencias internas |
| `README-nuget.md` | Guía completa de publicación y consumo de paquetes NuGet |
| `README-docs.md` | Convención de documentación APS, plantillas y guía del workflow de sincronización de `ops-docs` al vector store |

## Catálogo de workflows reutilizables

### `.github/workflows/nuget-ci-publish.yml`

Workflow reutilizable para repositorios .NET que:

- ejecuta `restore`, `build` y `test` en `push` y `pull_request`;
- publica paquetes NuGet en GitHub Packages en `workflow_dispatch`;
- soporta uno o varios paquetes mediante el input `packages`;
- resuelve dependencias internas entre paquetes del mismo repositorio;
- crea tags y GitHub Releases tras una publicación satisfactoria.

Referencia completa: [README-nuget.md](README-nuget.md).

---

### `.github/workflows/azure-functions-deploy.yml`

Workflow reutilizable para repositorios que despliegan **Azure Functions** (Isolated Worker v4, .NET 8+). Patrón build → upload artifact → deploy.

**Qué hace:**
- instala el SDK de .NET, hace `restore` (con soporte a feeds NuGet privados), `build`, `test` y `publish`;
- sube el artefacto compilado entre jobs;
- hace login en Azure vía OIDC (Workload Identity Federation) y despliega con `Azure/functions-action`.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | Nombre del [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments) (dev, int, pro…) |
| `function_app_name` | ✅ | Nombre completo de la Function App destino |
| `project_path` | — | Ruta al proyecto (csproj o directorio). Por defecto `.` |
| `dotnet_version` | — | Versión del SDK de .NET. Por defecto `8.x` |

**Secrets requeridos** (propagados con `secrets: inherit`):
`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `APS_NUGET_TOKEN`.

**Caller mínimo** (`.github/workflows/deploy.yml` en el repo consumidor):

```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [dev, int, pro]

jobs:
  deploy:
    uses: APS-Framework/.github/.github/workflows/azure-functions-deploy.yml@main
    with:
      environment:        ${{ inputs.environment }}
      function_app_name:  ${{ vars.FUNCTION_APP_NAME }}
      project_path:       src/My.FunctionApp
    secrets: inherit
```

---

### `.github/workflows/container-app-deploy.yml`

Workflow reutilizable para repositorios que despliegan una **Azure Container App**. Patrón build .NET → docker build & push a ACR → `az containerapp update`.

**Qué hace:**
- instala el SDK de .NET, hace `restore` (con soporte a feeds NuGet privados), `build` y `test`;
- calcula el tag de imagen (input o primeros 7 caracteres del SHA del commit);
- hace login en Azure vía OIDC y en el ACR con `az acr login`;
- construye la imagen Docker y la sube al ACR; `APS_NUGET_TOKEN` se inyecta como `--build-arg` para que el Dockerfile pueda restaurar paquetes NuGet privados durante el build (requiere `ARG APS_NUGET_TOKEN` en el Dockerfile);
- actualiza la revisión activa de la Container App con la nueva imagen.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | Nombre del [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments) (dev, int, pro…) |
| `acr_name` | ✅ | Nombre del ACR sin sufijo `.azurecr.io` (e.g. `acrramblaresbaidev`) |
| `container_app_name` | ✅ | Nombre completo de la Container App (e.g. `ca-rambla-resiberai-dev`) |
| `resource_group` | ✅ | Resource group donde vive la Container App |
| `container_repository` | ✅ | Repositorio de imagen dentro del ACR (e.g. `resiberai`) |
| `image_tag` | — | Tag de imagen. Vacío = SHA corto del commit |
| `dotnet_version` | — | Versión del SDK de .NET. Por defecto `8.x` |
| `dockerfile` | — | Ruta al Dockerfile. Por defecto `./Dockerfile` |
| `docker_build_context` | — | Contexto de construcción Docker. Por defecto `.` |

**Secrets requeridos** (propagados con `secrets: inherit`):
`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `APS_NUGET_TOKEN`.

**Variables de GitHub necesarias** (repo o por entorno):
`ACR_NAME`, `CONTAINER_APP_NAME`, `RESOURCE_GROUP`, `CONTAINER_REPOSITORY`.

**Caller mínimo** (`.github/workflows/deploy.yml` en el repo consumidor):

```yaml
name: Container App CI/CD

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [dev, int, pro]

jobs:
  deploy:
    uses: APS-Framework/.github/.github/workflows/container-app-deploy.yml@main
    with:
      environment:          ${{ inputs.environment }}
      acr_name:             ${{ vars.ACR_NAME }}
      container_app_name:   ${{ vars.CONTAINER_APP_NAME }}
      resource_group:       ${{ vars.RESOURCE_GROUP }}
      container_repository: ${{ vars.CONTAINER_REPOSITORY }}
    secrets: inherit
```

**Variables de ejemplo para el entorno `dev`** del proyecto ResiberAI:

| Variable | Valor |
|---|---|
| `ACR_NAME` | `acrramblaresbaidev` |
| `CONTAINER_APP_NAME` | `ca-rambla-resiberai-dev` |
| `RESOURCE_GROUP` | `RAMBLA-LV-RESIBERAI-RG-DEV` |
| `CONTAINER_REPOSITORY` | `resiberai` |

### `.github/workflows/sync-vector-docs.yml`

Workflow reutilizable para repositorios que mantienen documentación operativa en Markdown y necesitan sincronizarla con un vector store compartido. Este workflow:

- sincroniza ficheros `.md` seleccionados mediante un glob repo-relativo (`file_filter`);
- publica cada documento con un nombre canónico `opsdocs::{docs_prefix}/{ruta/relativa}`;
- converge el vector store al estado del repositorio creando, actualizando y eliminando adjuntos;
- evita sincronizaciones accidentales de más de 200 ficheros salvo confirmación explícita;
- puede migrar nombres legacy al formato canónico sin tocar entradas ajenas al repositorio.

Referencia completa: [README-docs.md](README-docs.md).

## Scripts compartidos

### `.github/scripts/nuget_publish.py`

Script invocado por `nuget-ci-publish.yml` durante la fase de publicación. Se encarga de:

- descubrir proyectos publicables bajo `src/`;
- calcular versiones `stable` o `rc` por paquete;
- ordenar la publicación según dependencias internas (`ProjectReference`);
- transformar temporalmente dependencias internas a `PackageReference`;
- consultar GitHub Packages para resolver la última versión publicada cuando aplica;
- publicar paquetes, crear tags y generar GitHub Releases.

## Uso

Cada repositorio caller invoca el workflow centralizado que necesite con un fichero mínimo en `.github/workflows/`.

- Para CI y publicación NuGet, consulta la sección **6. Configurar un nuevo repositorio SDK** en [README-nuget.md](README-nuget.md).
- Para despliegue de Azure Functions, consulta la sección **`.github/workflows/azure-functions-deploy.yml`** más arriba en este README.
- Para despliegue de Azure Container Apps, consulta la sección **`.github/workflows/container-app-deploy.yml`** más arriba en este README.
- Para sincronización de `ops-docs` al vector store, consulta la sección **8. Workflow reutilizable: Sync Vector Store Docs** en [README-docs.md](README-docs.md).
