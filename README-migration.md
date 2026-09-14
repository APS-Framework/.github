# Migración de repos a GitHub Actions

Runbook para migrar repos de la organización (Function Apps, WebApps y SDKs) a GitHub
Actions usando los workflows compartidos de `APS-Framework/.github`. Recoge las
operaciones realizadas en la migración piloto y la resolución de los problemas
conocidos.

> Referencias: [README-deploy.md](README-deploy.md) (pipelines y environments),
> [README-nuget.md](README-nuget.md) (paquetes) y [README-docs.md](README-docs.md) (documentación).

---

## 1. Checklist por repo

### 1.1 Caller de deploy

Crear `.github/workflows/deploy.yml`. Dos opciones:

- **Pipeline completa** (`pipeline-functions.yml`): entornos fijos `int`, `sbx`, `pro`.
- **Composición manual** cuando el caller necesita pasar los nombres de los recursos
  (`pipeline-functions.yml` no acepta nombres): encadenar `dotnet-build.yml` →
  `azure-functions-deploy.yml` → (`azure-slot-swap.yml` en PRO).

En la composición manual, el caller **compone los nombres** como `<prefijo>-<entorno>`
y los pasa como inputs; las credenciales por stage salen de las org vars sufijadas:

```yaml
jobs:
  build:
    uses: APS-Framework/.github/.github/workflows/dotnet-build.yml@main
    with:
      project_path: '<Sln>.sln'
      artifact_name: app-drop
      dotnet_version: '10.x'
    secrets: inherit

  deploy-int:
    needs: build
    uses: APS-Framework/.github/.github/workflows/azure-functions-deploy.yml@main
    with:
      environment: int
      function_app_name: ${{ vars.FUNCTION_<APP> }}-int
      resource_group: ${{ vars.RESOURCE_GROUP_PREFIX }}-INT
      artifact_name: app-drop
      dotnet_version: '10.x'
      key_vault_name: ${{ vars.KEY_VAULT_PREFIX }}-int
      app_config_name: ${{ vars.APP_CONFIG_PREFIX }}-int
      label: ${{ vars.LABEL || '<LABEL>' }}
      url_value_prefix: ${{ vars.URL_VALUE_PREFIX || '<URL.KEY>' }}
      api_key_value_prefix: ${{ vars.API_KEY_VALUE_PREFIX || '<API.KEY>' }}
      azure_client_id: ${{ vars.AZURE_CLIENT_ID_INT }}
      azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
      azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID_INT }}
    secrets: inherit
```

- **El stage `sbx` usa los recursos con sufijo `dev`** (app, Key Vault, App Config y
  RG). El input `environment` sigue siendo `sbx` (es el GitHub Environment).
- PRO va al slot `staging` y `azure-slot-swap.yml` hace swap + config sync.
- Ocultar PRO hasta tener su RBAC: input booleano `deploy_pro` con `if:` en el job.

### 1.2 Tests

- **Unitarios** en `tests/<Proyecto>.UnitTest` (glob `**/*UnitTest*.csproj`).
- **Integración** en `tests/<Proyecto>.IntegrationTest` (glob `**/*IntegrationTest*.csproj`),
  con `[TestCategory("Integration")]` en la clase y ejecución como gate previo al deploy.
- Stack: MSTest v3 + NSubstitute + Shouldly.
- Variables que reciben los ITs: `TEST_STAGE` (`INT`/`SBX`/`PRO`), `APP_CONFIG_PREFIX`
  / `APP_CONFIG_ENDPOINT` (el endpoint se compone `https://{prefijo}-{entorno}.azconfig.io`)
  y la identidad OIDC del job (`DefaultAzureCredential`).
- No bajar el listón de los ITs: el error middleware de las Function Apps mapea
  excepciones a HTTP 200, así que un assert de `<500` puede ocultar fallos de arranque
  (ver sección 2).

### 1.3 Paquetes

- Usar las últimas versiones del feed del propietario (`APS-Framework` / `CS-Level`).
- Versiones mínimas por los fixes de telemetría: ver sección 2.
- `nuget.config` con `%APS_NUGET_TOKEN%` (ver README-nuget).

### 1.4 Runtime de Azure

Al migrar a `net10.0`, subir el runtime de la Function App (Windows):

```bash
az functionapp config set -n <app> -g <rg> --subscription <sub> --net-framework-version v10.0
```

Sin esto, el deploy del build net10 arranca con `HTTP Error 500.30 - ASP.NET Core app
failed to start` (worker `dotnet.exe` sale con `0xE0434352`).

### 1.5 Environments de GitHub

Por repo, environments `int`, `sbx` y `pro` con:

| Nivel | Nombre | Tipo |
|---|---|---|
| environment | `FUNCTION_APP_NAME` / `WEBAPP_NAME` | var (con `pipeline-functions`) |
| environment | `RESOURCE_GROUP`, `KEY_VAULT_NAME`, `APP_CONFIG_NAME`, `LABEL`, `URL_VALUE_PREFIX`, `API_KEY_VALUE_PREFIX` | var |
| environment | `APP_CONFIG_PREFIX` / `APP_CONFIG_ENDPOINT` | var (ITs) |
| environment | `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` | secret (solo si no se usan org vars sufijadas) |
| environment | required reviewers | protection rule |

### 1.6 Org vars (se definen una vez)

- Prefijos COMMON (`FUNCTION_*`, `KEY_VAULT_PREFIX`, `APP_CONFIG_PREFIX`,
  `RESOURCE_GROUP_PREFIX`, …).
- `AZURE_TENANT_ID`, `AZURE_CLIENT_ID_<ENV>`, `AZURE_SUBSCRIPTION_ID_<ENV>`.

### 1.7 Federated credentials

Una App Registration por entorno (o una compartida) con una credential por repo:

```bash
az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
  "name": "<repo>-<env>",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<Org>/<repo>:environment:<env>",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

### 1.8 RBAC del Service Principal

| Necesidad | Rol | Ámbito |
|---|---|---|
| Deploy (config-zip) y swap | Contributor | suscripción o RG/app |
| Config sync (Key Vault) | Key Vault Secrets Officer | suscripción o vault |
| Config sync (App Config) | App Configuration Data Owner | store |
| ITs (leer App Config) | App Configuration Data Owner | store |

> `Owner` de suscripción **no** da acceso data-plane a Key Vault/App Config: hay que
> asignar los roles de datos explícitamente.

### 1.9 Validación

1. `dotnet build` + unit tests en local.
2. Dispatch del workflow; aprobar los gates por entorno.
3. Smoke de los endpoints del entorno (`/warmup`, `/confirm`) y comprobar headers de
   correlación (`X-Transaction-Id`).

---

## 2. Problemas conocidos

### 2.1 `FunctionContext` / `WorkerTelemetryContextAccessor`

**Síntoma**

- Function App con `HTTP Error 500.30` al arrancar, o
- respuestas HTTP **200** con error en el body (el error middleware las enmascara):

```json
{
  "ExceptionMessage": "Unable to resolve service for type 'Microsoft.Azure.Functions.Worker.FunctionContext' while attempting to activate 'APS.Telemetry.Worker.Services.WorkerTelemetryContextAccessor'.",
  "ExceptionType": "InvalidOperationException"
}
```

**Causa**

- `UseTelemetryMiddleware` registraba `ITelemetryRuntimeContextAccessor →
  WorkerTelemetryContextAccessor`, cuyo constructor exigía `FunctionContext` — y el
  runtime del worker **no lo registra en DI** (solo expone `IFunctionContextFactory`).
- Además, `AddCosmosContainer<CosmosPerformanceLogger>` registraba el logger de
  telemetría como **keyed singleton** y el repositorio (singleton) lo capturaba en
  construcción, fuera del scope de la invocación.

**Solución**

Actualizar los paquetes (no requiere cambios de código en el consumidor):

| Paquete | Versión mínima | Fix |
|---|---|---|
| `APS.Telemetry.Worker` | `0.2.1-rc.14` | opciones con fallback + `FunctionContext` opcional en el accessor |
| `APS.Data.Cosmos` | `0.1.1-rc.2` | logger keyed y `ICosmosRepository<T>` **scoped** |
| `APS.Telemetry.AspNetCore` | `0.1.1-rc.16` | opciones con fallback (webapps) |

Con la sección `Telemetry` ausente en App Config ya no hay NRE (los defaults se
aplican solos), y `BeginChildOperation` degrada cuando no hay `FunctionContext`.

**Notas**

- Sigue llamándose solo `UseTelemetryMiddleware()`; no hay registros extra.
- Si se usa `IScopedBag` **sin** el middleware de telemetría, registrar
  `services.AddScopedBag()` (APS.DependencyInjection).
- `ICosmosRepository<T>` pasa a **scoped**: no inyectarlo en singletons.
- El contexto fuera del flujo de invocación (hosted services, trabajo desacoplado) no
  tiene correlación; pasar los IDs explícitamente o degradar a `UNKNOWN`.

**Verificación**

1. Build + unit tests.
2. ITs contra el entorno (idealmente con un endpoint que resuelva el repositorio).
3. `GET /warmup` → 200 y `POST /confirm` con payload inválido → error de validación
   (no el error de DI).

### 2.2 ApplicationInsights 3.x (`ITelemetryProcessor`)

**Síntoma** (arranque de la app):

```
System.TypeLoadException: Could not load type 'Microsoft.ApplicationInsights.Extensibility.ITelemetryProcessor'
from assembly 'Microsoft.ApplicationInsights, Version=3.1.2...'
  at Program.<<Main>$>b__0_1(IServiceCollection services)
```

**Causa**: los paquetes APS ya resuelven `Microsoft.ApplicationInsights` 3.x, pero el
`Program.cs` usaba APIs 2.x (`AddApplicationInsightsTelemetryWorkerService` de
`Microsoft.ApplicationInsights.WorkerService` 2.23 + `ConfigureFunctionsApplicationInsights`
de `Microsoft.Azure.Functions.Worker.ApplicationInsights`).

**Solución**

- Bump de `Microsoft.ApplicationInsights.WorkerService` a `3.1.2`.
- Eliminar el paquete `Microsoft.Azure.Functions.Worker.ApplicationInsights` y la
  llamada `ConfigureFunctionsApplicationInsights()`.

### 2.3 El error middleware devuelve 200

Las Function Apps mapean `<Exception>` a `HTTP 200` para que Adyen/terceros no
reintenten. Consecuencia: los ITs que solo validan `< 500` **no detectan** fallos de
arranque o de DI. Incluir siempre una aserción sobre el cuerpo de error esperado
(`ExceptionType`).

---

## 3. Orden recomendado de migración

1. Ajustar paquetes y `Program.cs` (secciones 2.2 y 2.1).
2. Añadir tests (unitarios e integración) y habilitar los globs en el caller.
3. Crear el caller `deploy.yml` (pipeline o composición manual).
4. Configurar environments, org vars, federated credentials y RBAC.
5. Subir el runtime de las apps a `v10.0`.
6. Probar en `int` (y `sbx`); dejar `pro` para el final (RBAC + slot).
