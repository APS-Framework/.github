---
name: commit
description: "Crea un commit de los cambios preparados y lo sube a la rama remota. Sigue el procedimiento del MCP aps-framework. Úsala solo cuando el usuario invoque /commit."
---

Crea un commit de los cambios preparados y súbelo a la rama remota.

Consulta la tool `aps_framework__github`, sección `commit`, y sigue ese procedimiento.

Siempre, aunque el MCP no esté disponible:

- Nunca hagas commit en la rama por defecto: propón un nombre de rama.
- Nunca uses `--force` ni `--no-verify`.
- Nunca subas secretos ni credenciales.

Si el MCP no responde, avísalo y para.
