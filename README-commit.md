# Commit y push

> Procedimiento del comando `/commit`. Crea un commit y lo sube a la rama remota.

## Reglas

- Nunca en la rama por defecto.
- Sin `--no-verify`, `--force` ni `--amend`.
- Sin líneas `Co-Authored-By`.
- Si un paso falla, parar y explicar qué falta.

## Pasos

1. **Rama.** Obtener la rama por defecto con `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`. Si es la actual, no hacer commit: proponer un nombre `tipo/descripcion-corta` a partir de los cambios y crear la rama solo si el usuario lo confirma.
2. **Cambios.** Revisar `git status` y `git diff --staged`. Si no hay nada preparado, enseñar los ficheros modificados y preguntar cuáles incluir. No usar `git add -A`.
3. **Validaciones.** Revisar el diff con la sección `review` y con `REVIEW.md` si existe. Un problema bloqueante detiene el commit; los avisos se enseñan y decide el usuario.
4. **Documentación.** Aplicar la sección `git_ops`.
5. **Build y tests.** Ejecutar los comandos del repo indicados en `AGENTS.md` o `README-dev.md`. Si fallan, parar.
6. **Mensaje.** Generarlo a partir del diff, en formato Conventional Commits (`tipo(ámbito): resumen`), con el idioma de los últimos commits del repo. Explicar el porqué en el cuerpo si no es evidente.
7. **Commit y push.** `git commit`, y después `git push -u origin HEAD`. Si el remoto va por delante, parar y avisar: no reescribir el historial.
8. **Resultado.** Mostrar hash, mensaje, rama y si ya hay una PR abierta.
