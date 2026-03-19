# Operaciones git · Revisión de documentación

> Guía para evaluar si la documentación debe actualizarse antes de consolidar cambios
> mediante commit, push o PR.

---

## Cuándo aplicar esta revisión

Aplicar **siempre** que el desarrollador pida en el chat alguna de estas operaciones:

- Hacer un commit (`commit`, `git commit`, "consolida los cambios", "guarda los cambios")
- Hacer un push (`push`, `git push`, "sube los cambios", "sube al remoto")
- Crear una PR o MR ("crea una PR", "abre una pull request", "genera un PR")
- Cualquier combinación de las anteriores ("commitea y sube", "push y PR", etc.)

---

## Flujo de revisión

### 1. Identificar ficheros con cambios relevantes

Revisar los ficheros modificados (staged o unstaged). Los cambios relevantes para documentación son:

| Tipo de fichero | Afecta a |
|---|---|
| Interfaces públicas (`.cs` en `Contracts/`, `Abstractions/`) | `README-sdk.md` |
| DTOs, modelos o enums públicos | `README-sdk.md` |
| Métodos con cambio de firma, parámetros o tipo de retorno | `README-sdk.md` |
| Extensiones de DI (`ServiceCollectionExtensions`, `IoCExtensions`) | `README-sdk.md` |
| Configuración (`appsettings`, opciones de DI, variables de entorno) | `README-sdk.md` y/o `README-dev.md` |
| Arquitectura interna, nuevos proyectos, dependencias externas | `README-dev.md` |
| Workflows de despliegue, scripts de infraestructura | `README-dev.md` |
| Ejemplos de uso (`examples/`) | `README-sdk.md` |
| Ficheros internos sin superficie pública (helpers, utilities) | Sin impacto en doc |
| Tests | Sin impacto en doc |
| Refactorizaciones internas sin cambio de API | Sin impacto en doc |

### 2. Verificar si existe documentación

Si el repo tiene `README-sdk.md` o `README-dev.md` y hay cambios relevantes según la tabla anterior:

1. Leer los ficheros modificados relevantes
2. Leer el documento de documentación correspondiente
3. Comparar: identificar qué secciones están desactualizadas o incompletas

### 3. Actuar según el resultado

**Si la documentación está al día:** continuar con la operación git sin interrupciones.

**Si la documentación está desactualizada o incompleta:**
- Indicar qué secciones necesitan actualizarse y por qué (qué cambió en código que no está reflejado)
- Preguntar al desarrollador si quiere actualizar la documentación antes de continuar
- Si confirma: aplicar solo los cambios mínimos necesarios en los docs afectados, luego continuar con el git
- Si decide omitirlo: continuar con la operación git

**Si el repo no tiene `README-sdk.md` ni `README-dev.md`:** continuar con la operación git sin revisión de documentación.

---

## Reglas

- **No bloquear** operaciones git por cambios que no afecten a la superficie pública ni a la documentación (refactorizaciones internas, tests, correcciones de bugs sin cambio de API).
- **No regenerar** documentos completos — solo actualizar las secciones afectadas.
- **No preguntar** si no hay documentación que revisar o si los cambios son claramente internos.
- La decisión final siempre es del desarrollador: si decide no actualizar la doc, proceder con el git.
