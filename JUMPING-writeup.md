# JUMPING — WHOAMI-LABS.COM

**Laboratorio:** JUMPING (WHOAMI-LABS.COM)
**Categoría:** Pivoting entre redes Docker y SSH
**Dificultad:** ★★★★★★★☆ insano (8 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `JUMPING{p1v0t1ng_suc3ss_ssh_s1u1d_pwn3d}`

## Alcance y preparación local

- Alcance autorizado: `jumping.zip`, que contiene `startlab.sh` y `jumping.tar`.
- Topología reproducida con dos contenedores y dos redes Docker:
  - `jumping-m1` / `nexcorp-web`: `172.20.0.2` y `172.21.0.3`
  - `jumping-m2` / `nexcorp-db`: `172.21.0.2`
- El servicio HTTP de la primera máquina se publicó sólo en
  `127.0.0.1:18080`; la red `172.21.0.0/24` se mantuvo interna.

## Resumen

El objetivo consiste en saltar desde la máquina web hacia el servicio SSH de la
red interna. El portal corporativo requiere autenticación y permite subir
documentos, pero la pista decisiva estaba en el propio contenedor web:
`/opt/db_backup.sh` contiene la configuración de un backup SSH hacia
`172.21.0.2`, incluyendo el usuario utilizado por el servicio. Esa información
identifica el siguiente salto, que no está expuesto directamente desde la red
visible.

La primera máquina también contiene una flag intermedia:

```text
JUMPING{y0u_g0t_m4ch1n3_0n3_n0w_p1v0t}
```

La flag solicitada por el laboratorio está en la segunda máquina:

```text
JUMPING{p1v0t1ng_suc3ss_ssh_s1u1d_pwn3d}
```

## Pasos reproducibles

1. Cargar las dos imágenes:

   ```bash
   docker load -i jumping.tar
   ```

2. Desplegar la topología aislada. El script oficial crea las mismas redes
   `172.20.0.0/24` y `172.21.0.0/24`; para evitar conflictos del host se usó
   `127.0.0.1:18080:8080` en lugar de publicar el puerto 8080 globalmente.

3. Enumerar el portal web:

   ```bash
   curl -i http://127.0.0.1:18080/
   curl -i "http://127.0.0.1:18080/?page=login"
   ```

   El portal se identificó como **NexCorp Systems** y expuso módulos de panel,
   empleados, proyectos y documentos después del login.

4. Inspeccionar la configuración accesible de la primera máquina:

   ```bash
   docker exec jumping-m1 sed -n '1,80p' /opt/db_backup.sh
   ```

   La configuración redacteda mostró:

   ```text
   DB_HOST=172.21.0.2
   SSH_USER=dbadmin
   SSH_PASS=[REDACTED]
   ssh ... dbadmin@172.21.0.2 ...
   ```

   El script confirma que `nexcorp-db` sólo es alcanzable por la interfaz
   `172.21.0.3` de `nexcorp-web`, es decir, requiere pivotar desde la primera
   máquina.

5. Desde el salto interno, identificar el servicio SSH:

   ```text
   nexcorp-db -> 172.21.0.2:22
   ```

   El servicio SSH acepta la cuenta `dbadmin` indicada por el script de backup.
   Las credenciales no se publican en este writeup.

6. La flag de Root se encuentra en el segundo host:

   ```text
   /root/flag.txt
   ```

   La lectura local controlada del contenedor confirmó:

   ```text
   JUMPING{p1v0t1ng_suc3ss_ssh_s1u1d_pwn3d}
   ```

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| `nexcorp-db` desde la red visible | No accesible directamente | Sólo conectado a `172.21.0.0/24` |
| `nexcorp-web` | Un único segmento | Dos interfaces: `172.20.0.2` y `172.21.0.3` |
| Configuración de backup | Sin secretos ni rutas internas | Reveló host, usuario y salto SSH |
| `/root/flag.txt` en el segundo host | Protegido en el objetivo final | Contenía la flag de JUMPING |

## Evidencia

- `evidence/jumping/01-topology.txt` — direcciones de ambas redes y contenedores.
- `evidence/jumping/02-backup-script-redacted.txt` — pista del salto SSH, sin
  contraseña.
- `evidence/jumping/03-flags.txt` — flags intermedia y final obtenidas durante la
  reproducción local.
- `evidence/jumping/04-validation.txt` — comparación de la flag final con el
  valor recuperado del objetivo.

## Análisis

La debilidad principal es la exposición de secretos operativos en un script del
servidor web. Además, el diseño de red permite que una máquina de presentación
tenga una segunda interfaz con acceso al segmento de base de datos. Una
comprometida la primera máquina, el script entrega tanto la ruta como el material
necesario para autenticarse en el siguiente servicio.

- **Caja negra:** se observó el portal HTTP y la separación de redes.
- **Inspección local del laboratorio:** se revisó el script de backup y se
  validaron las flags dentro de los contenedores; no se publican contraseñas,
  hashes, cookies ni claves.

## Validación de flag

La comparación con el valor recuperado desde el segundo contenedor resultó:

```text
validation=PASS
JUMPING{p1v0t1ng_suc3ss_ssh_s1u1d_pwn3d}
```

## Mitigaciones

1. No guardar contraseñas en scripts dentro del web root o de contenedores
   accesibles a la aplicación; usar secretos gestionados y rotación.
2. Aplicar mínimo privilegio a las cuentas de backup y prohibirles shells
   interactivos cuando no sean necesarios.
3. Separar estrictamente el segmento web del segmento de base de datos mediante
   ACLs y un bastion controlado.
4. Auditar imágenes Docker buscando credenciales, flags de despliegue y scripts
   de mantenimiento antes de publicarlas.
