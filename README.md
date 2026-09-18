# APS/.github

Repositorio de configuraciones y workflows compartidos de la organización APS.

## Contenido

| Fichero | Descripción |
|---|---|
| `.github/workflows/dotnet-build.yml` | Build reusable: restore + build + tests unitarios + publish + artifact (build once) |
| `.github/workflows/azure-functions-deploy.yml` | Deploy de Azure Functions desde artifact: integration tests → deploy → config sync (slot opcional) |
| `.github/workflows/azure-webapp-deploy.yml` | Deploy de Azure Web App desde artifact: integration tests + deploy (slot opcional) |
| `.github/workflows/azure-slot-swap.yml` | Swap de slot staging → production para Functions/WebApp (re-run = rollback) |
| `.github/workflows/azure-functions-config-sync.yml` | Sincroniza App Configuration y Key Vault con la URL base y la API key de la Function App (uso standalone; el deploy ya lo ejecuta en su job) |
| `.github/workflows/container-app-build.yml` | Build de Container App: tests unitarios + docker build & push a ACR |
| `.github/workflows/container-app-deploy.yml` | Deploy de Container App desde imagen: integration tests + revisión (staging label opcional) |
| `.github/workflows/container-app-promote.yml` | Promote de tráfico por label/revisión (equivalente al swap en Container Apps) |
| `.github/workflows/nuget-ci-publish.yml` | CI y publicación de paquetes NuGet en GitHub Packages |
| `.github/workflows/sync-vector-docs.yml` | Sincronización de `ops-docs` Markdown con un vector store compartido |
| `.github/scripts/nuget_publish.py` | Orquestador compartido para publicación multi-paquete NuGet |
| `README-nuget.md` | Guía completa de publicación y consumo de paquetes NuGet |
| `README-docs.md` | Convención de documentación APS y guía del workflow de sincronización |
| `README-deploy.md` | Guía de despliegue: pipelines, environments, vars/secrets por nivel, federated credentials y Terraform |
| `README-migration.md` | Runbook de migración de repos a GitHub Actions: checklist y problemas conocidos (FunctionContext, ApplicationInsights 3.x) |

## Flujo build once / promote

Patrón equivalente a las pipelines ADO `CS.Level.*` (`APSRepo/APS.Templates`): un único build con
tests unitarios, promoción del mismo artifact por todos los entornos, integration tests como gate
previo a cada deploy, y swap de slot en PRO.

### Composición desde el caller

Un solo run encadena todas las fases; las aprobaciones pausan el run (no lo relanzan).
El caller declara qué entornos hay, en qué orden se promocionan y con qué gates; los
bloques del shared hacen el trabajo de cada paso:

```
deploy.yml (caller)
  build (dotnet-build: build + UT)
    → deploy int (azure-functions-deploy: ITs → deploy → config sync)
    → deploy sbx (azure-functions-deploy: ITs → deploy → config sync)
    → deploy pro (azure-functions-deploy: ITs → deploy slot staging)
    → swap        (azure-slot-swap: swap + config sync)
```

| Tecnología | Bloques a encadenar |
|---|---|
| Functions | `dotnet-build` → `azure-functions-deploy` ×N → `azure-slot-swap` |
| Web App | `dotnet-build` → `azure-webapp-deploy` ×N → `azure-slot-swap` |
| Container App | `container-app-build` → `container-app-deploy` ×N → `container-app-promote` |

Caller de referencia (build → validate → int → sbx → pro staging → swap + publish NuGet):
`CS.Level.Booking/.github/workflows/deploy.yml`. Forma canónica de los gates y ejemplo
mínimo en [README-deploy.md](README-deploy.md) §2.

Reglas del flujo:

- **Build**: los tests unitarios son obligatorios. El artifact se sube una sola vez y todos los
  deploys consumen el mismo binario (no hay rebuild por entorno).
- **Deploy**: los integration tests corren en el mismo job, antes del step de deploy, con las
  vars/secrets del GitHub Environment. Se ejecutan con `--filter "TestCategory=Integration"`
  (misma convención que `integration-tests.yaml` de APS.Templates) y reciben `TEST_STAGE`
  (environment en mayúsculas por defecto), `APP_CONFIG_CONNECTION` y `APP_CONFIG_ENDPOINT`.
  Los tests usan la identidad OIDC del job (`DefaultAzureCredential` → `AzureCliCredential`).
- **PRO**: el deploy se hace al slot `staging` y el workflow de swap lo promueve a `production`.
  Re-ejecutar el swap invierte la operación: rollback sin redeploy.
- **Config sync (opcional)**: tras el deploy (int/sbx) o el swap (pro),
  `azure-functions-config-sync.yml` publica la URL base y la function key en App Configuration y
  Key Vault (equivalente a `update-values.yaml` de APS.Templates).
- **Aprobaciones**: se configuran como protection rules de cada GitHub Environment (required
  reviewers, wait timer). El run queda en `Waiting` y reanuda al aprobar.
- **Configuración**: los nombres de recursos salen de las vars del environment y los valores de
  Azure de org vars sufijadas por entorno (`AZURE_CLIENT_ID_INT`, `AZURE_SUBSCRIPTION_ID_INT`, …).
  Detalle completo en [README-deploy.md](README-deploy.md).

### Composición manual (avanzado)

Los bloques (`dotnet-build.yml`, `azure-*-deploy.yml`, `azure-slot-swap.yml`,
`azure-functions-config-sync.yml`, `container-app-*.yml`) siguen siendo reutilizables para
encadenarlos a mano con `needs`; el catálogo con inputs y secrets está más abajo.

---

## Catálogo de workflows

### Orquestadores retirados

`pipeline-functions.yml`, `pipeline-webapp.yml` y `pipeline-container-app.yml` han sido
eliminados. Encadenaban los bloques por ti, pero mantenían dentro del shared información
que es del caller: qué entornos existen, el orden de promoción, los gates (`needs` que
cruzan `environment:`), el mapeo de sufijos (`sbx → -dev`) y la convención
`AZURE_CLIENT_ID_<ENV>`. El caller encadena ahora los bloques directamente.

Referencia completa: [README-deploy.md](README-deploy.md).

---

### `.github/workflows/dotnet-build.yml`

Build once para Functions y Web Apps: restore, tests unitarios, `dotnet publish` y upload del
artifact que consumen todos los deploys.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `project_path` | — | Ruta o glob MSBuild al `.sln`/`.slnx`/`.csproj`. Por defecto `**/*.sln` |
| `publish_project` | — | Glob al csproj de la app. Vacío = se detecta desde `project_path` (si es solución, excluye tests) |
| `artifact_name` | ✅ | Nombre del artifact que consumen los deploys |
| `unit_test_project` | — | Glob de tests unitarios. Por defecto `**/*UnitTest*.csproj`. Vacío = no ejecutar |
| `dotnet_version` | — | SDK de .NET. Por defecto `8.x`; acepta multilinea (`8.x` + `10.x`) |
| `configuration` | — | Por defecto `Release` |
| `retention_days` | — | Retención del artifact. Por defecto `7` |

**Secrets:** `APS_NUGET_TOKEN` (obligatorio), `NUGET_EXTERNAL_TOKEN` (opcional).

---

### `.github/workflows/azure-functions-deploy.yml`

Deploy de una Function App (Isolated Worker v4) desde el artifact de `dotnet-build.yml`.
**No compila.** En el mismo job: validación de configuración → integration tests → deploy → config sync
(omitido si se despliega a un slot; el swap ejecuta el suyo). Los pasos están en composite actions.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | GitHub Environment (int, dev, pro…) |
| `function_app_name` | — | Nombre completo de la Function App. Vacío = var `FUNCTION_APP_NAME` del environment |
| `artifact_name` | — | Artifact generado por `dotnet-build.yml`. Por defecto `app-drop` |
| `slot` | — | Slot destino. Vacío = `production` |
| `integration_test_project` | — | Glob de ITs. Por defecto `**/*IntegrationTest*.csproj`. Vacío = no ejecutar |
| `integration_test_stage` | — | Valor de `TEST_STAGE`. Vacío = environment en mayúsculas |
| `dotnet_version` | — | SDK para los integration tests. Por defecto `8.x` |
| `configuration` | — | Por defecto `Release` |
| `resource_group` | — | RG. Vacío = var `RESOURCE_GROUP` del environment o se resuelve por nombre |
| `key_vault_name` / `app_config_name` / `label` / `url_value_prefix` / `api_key_value_prefix` | — | Config sync. Vacío = vars del environment (`KEY_VAULT_NAME`, `APP_CONFIG_NAME`, `LABEL`, `URL_VALUE_PREFIX`, `API_KEY_VALUE_PREFIX`). Sin prefijos, el sync se omite |
| `api_key_vault_name` / `function_key_name` | — | Secreto en KV / function key a publicar. Por defecto `api_key_value_prefix` con `.`→`-` y `default` |

**Secrets:** `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `APS_NUGET_TOKEN`
(obligatorios); `NUGET_EXTERNAL_TOKEN` (opcional). Los ITs reciben `TEST_STAGE` y las vars
`APP_CONFIG_PREFIX` / `APP_CONFIG_ENDPOINT` del environment (el endpoint se compone
`https://{prefijo}-{entorno}.azconfig.io`).

> Los deployment slots de Functions requieren plan Premium o Dedicated; no existen en Consumption.

---

### `.github/workflows/azure-webapp-deploy.yml`

Deploy de una App Service Web App desde el artifact de `dotnet-build.yml`. Mismos inputs que el
deploy de Functions salvo los de config sync (no aplica en Web Apps), con `webapp_name` en lugar
de `function_app_name`. Usa `az webapp deploy --type zip` (con `--slot` opcional).

---

### `.github/workflows/azure-slot-swap.yml`

Swap del slot `staging` a `production` para Functions o Web Apps. Es la última etapa del flujo y
también el rollback: re-ejecutarlo invierte el swap sin redeploy. Para Functions ejecuta el
config sync tras el swap en el mismo job (si hay prefijos configurados).

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | GitHub Environment del swap |
| `app_type` | ✅ | `function` o `webapp` |
| `app_name` | ✅ | Nombre completo del recurso |
| `source_slot` | — | Por defecto `staging` |
| `target_slot` | — | Por defecto `production` |
| `preserve_vnet` | — | Solo Web Apps. Por defecto `false` |
| `resource_group` | — | Vacío = se resuelve por nombre |

**Secrets:** `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.

---

### `.github/workflows/azure-functions-config-sync.yml`

Sincroniza Azure App Configuration y Key Vault con la URL base y la function key de una Function
App (equivalente a `update-values.yaml`). Se ejecuta después del deploy en INT/DEV o después del
swap en PRO.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | GitHub Environment |
| `function_app_name` | ✅ | Nombre completo de la Function App |
| `key_vault_name` | ✅ | Key Vault donde se guarda la API key |
| `app_config_name` | ✅ | App Configuration a actualizar |
| `label` | ✅ | Label de App Configuration |
| `url_value_prefix` | ✅ | Key de App Config para la URL base |
| `api_key_value_prefix` | ✅ | Key de App Config para la API key |
| `api_key_vault_name` | — | Nombre del secreto. Vacío = `api_key_value_prefix` con `.` → `-` |
| `function_key_name` | — | Function key a publicar. Por defecto `default` |
| `resource_group` | — | Vacío = se resuelve por nombre |

**Secrets:** `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.

> La identidad OIDC necesita permiso de escritura en Key Vault (Key Vault Secrets Officer) y en
> App Configuration (App Configuration Data Owner).

---

### `.github/workflows/container-app-build.yml`

Build once de una Container App: tests unitarios, docker build y push al ACR. Devuelve
`image_tag` y `acr_login_server` para que los deploys compongan la referencia de imagen.

**Inputs principales:** `acr_name` ✅, `container_repository` ✅, `image_tag`, `dockerfile`,
`docker_build_context`, `unit_test_project`, `dotnet_version`, `configuration`.

**Outputs:** `image_tag`, `acr_login_server`.

---

### `.github/workflows/container-app-deploy.yml`

Deploy de una revisión de Container App a partir de una imagen ya publicada. **No construye la
imagen.** Container Apps no tiene deployment slots: el equivalente implementado es multiple
revision mode + labels de tráfico.

- `staging_label` vacío: update directo (`az containerapp update`).
- `staging_label` definido (ej. `staging`): la nueva revisión queda con 0% de tráfico (el 100%
  continúa en la revisión anterior) y accesible en `https://<app>---<label>.<fqdn>`.
  El paso a producción es `container-app-promote.yml`.

**Inputs principales:**

| Input | Obligatorio | Descripción |
|---|---|---|
| `environment` | ✅ | GitHub Environment (dev, int, pro…) |
| `container_app_name` | ✅ | Nombre completo de la Container App |
| `resource_group` | ✅ | Resource group del recurso |
| `image` | ✅ | Referencia completa `acr.azurecr.io/repo:tag` |
| `staging_label` | — | Label para validar la revisión sin tráfico (ej. `staging`) |
| `revision_suffix` | — | Sufijo opcional de la revisión (minúsculas, números, guiones) |
| `integration_test_project` | — | Glob de ITs. Por defecto `**/*IntegrationTest*.csproj` |

**Outputs:** `revision_name`, `previous_revision`.

---

### `.github/workflows/container-app-promote.yml`

Mueve el 100% del tráfico a un label o a una revisión concreta: el equivalente al swap.
Re-ejecutarlo con `revision` = revisión anterior = rollback.

**Inputs principales:** `environment` ✅, `container_app_name` ✅, `resource_group` ✅,
`label` o `revision` (excluyentes), `weight` (default 100).

**Ejemplo Container Apps:**

```yaml
jobs:
  build:
    uses: APS-Framework/.github/.github/workflows/container-app-build.yml@main
    with:
      acr_name:             ${{ vars.ACR_NAME }}
      container_repository: ${{ vars.CONTAINER_REPOSITORY }}
    secrets: inherit

  deploy-pro:
    needs: build
    uses: APS-Framework/.github/.github/workflows/container-app-deploy.yml@main
    with:
      environment:        pro
      container_app_name: ${{ vars.CONTAINER_APP_NAME }}
      resource_group:     ${{ vars.RESOURCE_GROUP }}
      image:              ${{ needs.build.outputs.acr_login_server }}/${{ vars.CONTAINER_REPOSITORY }}:${{ needs.build.outputs.image_tag }}
      staging_label:      staging
    secrets: inherit

  promote-pro:
    needs: deploy-pro
    uses: APS-Framework/.github/.github/workflows/container-app-promote.yml@main
    with:
      environment:        pro
      container_app_name: ${{ vars.CONTAINER_APP_NAME }}
      resource_group:     ${{ vars.RESOURCE_GROUP }}
      label:              staging
    secrets: inherit
```

---

### `.github/workflows/nuget-ci-publish.yml`

Workflow reutilizable para repositorios .NET que:

- ejecuta `restore`, `build` y `test` en `push` y `pull_request`;
- publica paquetes NuGet en GitHub Packages en `workflow_dispatch`;
- soporta uno o varios paquetes mediante el input `packages`;
- resuelve dependencias internas entre paquetes del mismo repositorio;
- crea tags y GitHub Releases tras una publicación satisfactoria.

Referencia completa: [README-nuget.md](README-nuget.md).

---

### `.github/workflows/sync-vector-docs.yml`

Workflow reutilizable para repositorios que mantienen documentación operativa en Markdown y necesitan
sincronizarla con un vector store compartido:

- sincroniza ficheros `.md` seleccionados mediante un glob repo-relativo (`file_filter`);
- publica cada documento con un nombre canónico `{docs_prefix}/{ruta/relativa}`;
- converge el vector store al estado del repositorio creando, actualizando y eliminando adjuntos;
- evita sincronizaciones accidentales de más de 200 ficheros salvo confirmación explícita.

Referencia completa: [README-docs.md](README-docs.md).

---

## Scripts compartidos

### `.github/scripts/nuget_publish.py`

Script invocado por `nuget-ci-publish.yml` durante la fase de publicación. Se encarga de:

- descubrir proyectos publicables bajo `src/`;
- calcular versiones `stable` o `rc` por paquete;
- ordenar la publicación según dependencias internas (`ProjectReference`);
- transformar temporalmente dependencias internas a `PackageReference`;
- consultar GitHub Packages para resolver la última versión publicada cuando aplica;
- publicar paquetes, crear tags y generar GitHub Releases.

## Migración desde los workflows combinados

`azure-functions-deploy.yml` y `container-app-deploy.yml` ya no compilan: son etapas de deploy del
flujo build once / promote. Los callers deben migrar a:

- Functions/WebApp: `dotnet-build.yml` → `azure-functions-deploy.yml` / `azure-webapp-deploy.yml`
  → (`slot: staging` + `azure-slot-swap.yml` en PRO). En Functions, el config sync corre dentro
  del job de deploy (int/sbx) o del swap (pro); ya no hay que orquestarlo como job aparte.
- Container Apps: `container-app-build.yml` → `container-app-deploy.yml`
  → (`staging_label` + `container-app-promote.yml` en PRO).

## Uso

Cada repositorio caller invoca los workflows centralizados con un fichero en `.github/workflows/`.

- Para CI y publicación NuGet, consulta la sección **6. Configurar un nuevo repositorio SDK** en [README-nuget.md](README-nuget.md).
- Para el flujo build once / promote, consulta los ejemplos de esta página.
- Para migrar un repo a GitHub Actions, consulta [README-migration.md](README-migration.md).
- Para sincronización de `ops-docs` al vector store, consulta la sección **8. Workflow reutilizable: Sync Vector Store Docs** en [README-docs.md](README-docs.md).
