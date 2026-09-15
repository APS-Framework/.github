# Despliegue · Pipelines build once / promote

Guía de las pipelines completas de la organización: un solo run encadena build,
tests unitarios, entornos con aprobación, integration tests, slot de staging y swap.

## 1. Pipelines disponibles

| Pipeline | Stages |
|---|---|
| `pipeline-functions.yml` | build (UT) → deploy int (ITs → deploy → config sync) → deploy sbx (ITs → deploy → config sync) → deploy pro (ITs → deploy slot `staging`) → swap (+ config sync) |
| `pipeline-webapp.yml` | build (UT) → deploy int (ITs) → deploy sbx (ITs) → deploy pro (ITs, slot `staging`) → swap |
| `pipeline-container-app.yml` | build+push (UT) → deploy int → deploy sbx → deploy pro (label `staging`, 0% tráfico) → promote |

- **Un solo run**: si una fase falla, las siguientes se saltan (`needs`) y el run queda fallido.
- **Build once**: el artifact/imagen se construye una vez; todos los deploys consumen lo mismo.
- **Aprobaciones**: cada GitHub Environment con required reviewers pausa el run en esa fase.
- **Entornos fijos**: `int`, `sbx`, `pro`, con selección opcional en `pipeline-functions.yml`
  (`deploy_int`/`deploy_sbx`/`deploy_pro`; por defecto, promoción completa).
- **Mismo job**: los unit tests corren tras el build; y en cada entorno los integration tests → deploy → config sync corren en el mismo job (una sola aprobación por entorno). En pro el config sync se ejecuta dentro del job del swap.
- **Bloques**: los pasos comunes viven en composite actions (`.github/actions/integration-tests`, `function-deploy`, `config-sync`) que usan los workflows reutilizables.

## 2. Caller mínimo

En cada repo de aplicación, `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  pipeline:
    uses: APS-Framework/.github/.github/workflows/pipeline-functions.yml@main
    secrets: inherit
```

Los bloques (`dotnet-build.yml`, `azure-functions-deploy.yml`, `azure-slot-swap.yml`,
`azure-functions-config-sync.yml`, `container-app-*.yml`) siguen siendo reutilizables
para composición avanzada (encadenar a mano con `needs`). Los pasos comunes están en
composite actions (`.github/actions/integration-tests`, `function-deploy`, `config-sync`),
de modo que el deploy de Functions y el config-sync standalone comparten implementación.

## 3. Environments requeridos en cada repo

Cada repo caller define los environments `int`, `sbx` y `pro` con:

| Nivel | Nombre | Tipo | Ejemplo |
|---|---|---|---|
| environment | `FUNCTION_APP_NAME` | var | `booking-func-int` |
| environment | `RESOURCE_GROUP` | var | `RAMBLA-LV-BOOKING-RG-INT` |
| environment | `KEY_VAULT_NAME` | var | `kv-booking-int` |
| environment | `APP_CONFIG_NAME` | var | `appcs-booking-int` |
| environment | `LABEL` | var | `BOOKING-CLIENT` |
| environment | `URL_VALUE_PREFIX` | var | `RAMBLA.Booking.Client.UrlService` |
| environment | `API_KEY_VALUE_PREFIX` | var | `RAMBLA.Booking.Client.Header.api-key` |
| environment | `APP_CONFIG_PREFIX` | var (opcional) | prefijo del App Config para los ITs; el endpoint se compone `https://{prefijo}-{entorno}.azconfig.io` (si el sufijo del recurso no coincide con el entorno, usar `APP_CONFIG_ENDPOINT`) |
| environment | `APP_CONFIG_ENDPOINT` | var (opcional) | `https://appcs-booking-int.azconfig.io` (override explícito para los ITs) |
| environment | `AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_SUBSCRIPTION_ID` | secret | solo si **no** usás las org vars sufijadas |

WebApps usan `WEBAPP_NAME` en lugar de `FUNCTION_APP_NAME` (y no tienen config sync).
Container Apps usan `CONTAINER_APP_NAME` + `RESOURCE_GROUP`, y su build usa las
repo/org vars `ACR_NAME` y `CONTAINER_REPOSITORY` (ACR compartido).

> Si un environment no define la var esperada, el job falla con un mensaje explícito
> indicando qué variable falta.

## 4. Valores de Azure a nivel organización

Los valores compartidos por todos los repos **no se repiten** en cada repo:

| Nivel | Nombre | Tipo | Varia por entorno |
|---|---|---|---|
| org secret | `APS_NUGET_TOKEN` | secret | No |
| org var | `AZURE_TENANT_ID` | var | No |
| org var | `AZURE_CLIENT_ID_INT`, `AZURE_CLIENT_ID_SBX`, `AZURE_CLIENT_ID_PRO` | var | Sí (sufijo) |
| org var | `AZURE_SUBSCRIPTION_ID_INT`, `AZURE_SUBSCRIPTION_ID_SBX`, `AZURE_SUBSCRIPTION_ID_PRO` | var | Sí (sufijo) |

Las pipelines pasan la var correspondiente a cada stage; si no existe, el bloque cae
al secret homónimo del environment. Si un repo necesita una excepción, define una
repo var/secret con el mismo nombre y pisa la de organización.

GitHub no soporta secrets/vars de organización con valores distintos por entorno:
por eso se usan **nombres sufijados** (`_INT`, `_SBX`, `_PRO`) y las pipelines
referencian el correcto en cada stage (los stages son fijos).

## 5. Federated credentials (OIDC)

Una App Registration **por entorno**, con una federated credential por repo:

| App registration | Subject de cada credential |
|---|---|
| `sp-cslevel-int` | `repo:CS-Level/<repo>:environment:int` |
| `sp-cslevel-sbx` | `repo:CS-Level/<repo>:environment:sbx` |
| `sp-cslevel-pro` | `repo:CS-Level/<repo>:environment:pro` |

```bash
az ad app federated-credential create \
  --id <APP_OBJECT_ID> \
  --parameters '{
    "name": "booking-int",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:CS-Level/CS.Level.Booking:environment:int",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

Con 5-7 repos × 3 entornos quedás cómodo respecto al límite de credenciales por app.
(Con muchos más repos habría que usar custom sub claims de GitHub Enterprise o un SP
por repo.)

## 6. Aprobaciones con teams de la organización

- `Required reviewers` acepta hasta **6 usuarios o teams**; basta con que **uno** apruebe.
- El team debe tener al menos **read access** al repo.
- Opción **Prevent self-review** para que quien disparó no pueda aprobar.
- **Plan**: required reviewers en repos privados requiere **GitHub Enterprise**; en
  Free/Pro/Team solo está disponible en repos públicos.
- La asignación del team al environment es por repo, pero se automatiza (ver Terraform).

## 7. Terraform (environments + reviewers + vars)

Ejemplo con el provider `integrations/github`:

```hcl
locals {
  environments = ["int", "sbx", "pro"]
  deploy_repos = ["CS.Level.Booking", "CS.Level.Payment", "CS.Level.Webhooks"]
}

data "github_team" "approvers" {
  for_each = toset(local.environments)
  slug     = "cs-level-approvers-${each.value}"
}

# Environments + reviewers
resource "github_repository_environment" "booking" {
  for_each = toset(local.environments)

  repository  = "CS.Level.Booking"
  environment = each.value

  reviewers {
    teams = [data.github_team.approvers[each.value].id]
  }
}

# Variables propias de la app, por entorno
resource "github_actions_environment_variable" "booking_function_app" {
  for_each = toset(local.environments)

  repository    = "CS.Level.Booking"
  environment   = each.value
  variable_name = "FUNCTION_APP_NAME"
  value         = "booking-func-${each.value}"
}

# Repetir el patron para RESOURCE_GROUP, KEY_VAULT_NAME, APP_CONFIG_NAME, LABEL,
# URL_VALUE_PREFIX y API_KEY_VALUE_PREFIX (o generalizar con for_each sobre un mapa
# de repos y sufijos).

# Valores de Azure a nivel organizacion (vars sufijadas)
resource "github_actions_organization_variable" "azure_client_id" {
  for_each = {
    INT = var.azure_client_id_int
    SBX = var.azure_client_id_sbx
    PRO = var.azure_client_id_pro
  }

  variable_name           = "AZURE_CLIENT_ID_${each.key}"
  visibility              = "selected"
  selected_repository_ids = [for repo in local.deploy_repos : data.github_repository.deploy[repo].repo_id]
  value                   = each.value
}
```

Para `APS_NUGET_TOKEN` usar `github_actions_organization_secret`. Con Bicep no es
posible gestionar GitHub: los outputs de la infraestructura hay que llevarlos a las
environment vars con `gh variable set --env <entorno>` o Terraform.

## 8. Notas

- **Retención del artifact**: `retention_days` (default 7) debe cubrir la espera entre
  aprobaciones; si PRO puede tardar más, subirlo (input de la pipeline).
- **Functions slots**: requieren plan Premium o Dedicated; no existen en Consumption. El config sync de pro se ejecuta dentro del job del swap (el deploy va al slot `staging`).
- **Tests**: los unitarios corren en el job de build (tras compilar) y los de integración como gate previo al deploy de cada entorno; los globs por defecto son `**/*UnitTest*.csproj` y `**/*IntegrationTest*.csproj`. Si no hay proyectos, el build falla salvo que se pase el patrón vacío.
- **WebApps**: sin config sync (la function key no aplica).
- **Container Apps**: no tiene slots; el "staging" es una revisión con label y el
  promote mueve el tráfico. Rollback = re-ejecutar `container-app-promote.yml` con la
  revisión anterior.
- **Overrides**: las pipelines exponen inputs (`project_path`, `artifact_name`,
  `dotnet_version`, `unit_test_project`, `integration_test_project`, `retention_days`)
  con defaults; no hace falta pasarlos.
