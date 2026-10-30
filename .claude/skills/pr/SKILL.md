---
name: pr
description: "Crea una Pull Request de la rama actual con sus work items. Sigue el procedimiento del MCP aps-framework. Úsala solo cuando el usuario invoque /pr."
---

Crea una Pull Request de la rama actual. Uso: `/pr 8125, 7995`

Work items: $ARGUMENTS (si ves ese texto sin sustituir, tómalos del mensaje del usuario).

Consulta la tool `aps_framework__github`, sección `pr`, y sigue ese procedimiento.

Siempre, aunque el MCP no esté disponible:

- Nunca crees la PR desde la rama por defecto.
- Nunca uses `--force` ni hagas merge.
- Nunca subas secretos ni credenciales.

Si el MCP no responde, avísalo y para.
