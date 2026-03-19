# APS/.github

Repositorio de configuraciones y workflows compartidos de la organización APS.

## Contenido

| Fichero | Descripción |
|---|---|
| `.github/workflows/nuget-ci-publish.yml` | Workflow reutilizable para CI y publicación de paquetes NuGet en GitHub Packages |
| `.github/scripts/nuget_publish.py` | Orquestador compartido para publicación multi-paquete NuGet con resolución de dependencias internas |
| `README-nuget.md` | Guía completa de publicación y consumo de paquetes NuGet |
| `README-docs.md` | Convención de documentación APS: README-sdk.md, README-dev.md, reglas por tipo de repo y plantillas |

## Catálogo de workflows reutilizables

### `.github/workflows/nuget-ci-publish.yml`

Workflow reutilizable para repositorios .NET que:

- ejecuta `restore`, `build` y `test` en `push` y `pull_request`;
- publica paquetes NuGet en GitHub Packages en `workflow_dispatch`;
- soporta uno o varios paquetes mediante el input `packages`;
- resuelve dependencias internas entre paquetes del mismo repositorio;
- crea tags y GitHub Releases tras una publicación satisfactoria.

Referencia completa: [README-nuget.md](README-nuget.md).

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

Cada repositorio SDK invoca el workflow centralizado con un caller mínimo. Consulta la sección **6. Configurar un nuevo repositorio SDK** en [README-nuget.md](README-nuget.md).
