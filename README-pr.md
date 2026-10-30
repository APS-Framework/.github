# Pull Request

> Procedimiento del comando `/pr`. Crea una PR de la rama actual contra la rama por defecto. Uso: `/pr 8125, 7995`.

## Reglas

- No hacer merge ni usar `--force`.
- Sin líneas `Co-Authored-By`.
- Si un paso falla, parar y explicar qué falta.

## Pasos

1. **Work items.** Separar por comas o espacios. Se aceptan `8125`, `#8125` o `AB#8125`, y deben ser números. Si no hay ninguno, pedirlo.
2. **Rama.** Si es la rama por defecto, parar y proponer un nombre de rama. Si hay cambios sin commit, avisar y sugerir `/commit`. Si la rama no está en el remoto, subirla con `git push -u origin HEAD`.
3. **PR existente.** Si `gh pr view` encuentra una abierta, enseñarla y preguntar si se actualiza la descripción.
4. **Validaciones.** Revisar todo el diff contra la rama por defecto, no solo el último commit, con la sección `review` y con `REVIEW.md` si existe. Ejecutar build y tests.
5. **Documentación.** Aplicar la sección `git_ops`.
6. **Alcance.** Si el diff incluye algo que el título no refleja (workflows, CI, despliegue, dependencias), avisar antes de continuar.
7. **Título.** Conventional Commits, generado a partir de todos los commits.
8. **Descripción.** Usar `.github/PULL_REQUEST_TEMPLATE.md` si existe, rellenarla y quitar sus comentarios. Incluir en la descripción una línea por work item con el formato `AB#8125`: `#8125` no enlaza con Azure Boards desde GitHub, y `AB#` en el título o en un comentario tampoco crea el enlace.
9. **Crear.** `gh pr create` contra la rama por defecto y mostrar la URL.
