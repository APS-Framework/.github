# Revisión de cambios

> Reglas generales que aplican `/commit` y `/pr`. Las propias de cada repo están en su `REVIEW.md`.

## Bloquean

El comando para hasta que se corrijan:

- Secretos, credenciales o tokens en el código.
- Datos personales o de pago en logs.
- Commit directo en la rama por defecto.
- Build o tests en rojo.
- PR sin work item real (`AB#` en la descripción).

## Avisan

Se enseñan y decide el usuario:

- Comportamiento nuevo o corregido sin test.
- SDK públicos de Azure donde corresponde un paquete `APS.*`, salvo en el propio SDK.
- Cambios de CI, despliegue o dependencias mezclados con otra cosa: van en su propia PR.
- Código comentado o `TODO` sin referencia.
- Cambios en contratos públicos que rompan a los consumidores.
- Operaciones que no se pueden reintentar o que pueden quedar a medias.
