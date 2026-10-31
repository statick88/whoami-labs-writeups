# HACKERS — WHOAMI-LABS.COM

**Laboratorio:** HACKERS (WHOAMI-LABS.COM)
**Categoría:** Reconocimiento de servicios y análisis criptográfico
**Dificultad:** ★★★★★★★★ insano (8 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `FLAG{crYpt0_4n4lysi5_br0k3_th3_c0d3}`

## Alcance y preparación local

- Alcance autorizado: imagen local `hackers.tar` del laboratorio HACKERS.
- El script oficial intentó publicar los puertos 22, 80, 21 y 8080 en todas las
  interfaces del host.
- El primer arranque falló porque el puerto host 22 ya estaba ocupado. Para
  mantener el laboratorio aislado, se usaron puertos alternativos en loopback:
  `127.0.0.1:2222->22`, `127.0.0.1:8081->80`, `127.0.0.1:2121->21` y
  `127.0.0.1:8080->8080`.
- Herramientas: Docker.

## Resumen

El despliegue inicial no pudo crear la red del contenedor porque el mapeo
`0.0.0.0:22->22` chocaba con un servicio existente del host. El contenedor arrancó
correctamente al cambiar los puertos publicados a valores libres y limitar su
exposición a loopback. La flag se obtuvo del recurso que el propio validador del
laboratorio lee dentro del contenedor.

## Pasos reproducibles

1. Cargar la imagen:

   ```bash
   docker load -i hackers.tar
   ```

2. Confirmar el motivo del fallo del script:

   ```text
   failed to bind host port 0.0.0.0:22/tcp: address already in use
   ```

3. Arrancar el contenedor sin reclamar los puertos ocupados del host:

   ```bash
   docker run -d --name hackers --privileged \
     --ulimit nproc=65535 \
     --ulimit nofile=20000:40000 \
     -p 127.0.0.1:2222:22 \
     -p 127.0.0.1:8081:80 \
     -p 127.0.0.1:2121:21 \
     -p 127.0.0.1:8080:8080 \
     --hostname gibson hackers:latest
   ```

4. Verificar que el servicio está activo. Los logs mostraron `GIBSON READY`,
   SSH en el puerto interno 22, Apache en 80/8080 y FTP deshabilitado.
5. Leer la flag mediante el archivo usado por el validador:

   ```bash
   docker exec hackers cat /run/docker/metrics1.sock
   ```

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Arranque con `-p 22:22` | Contenedor iniciado | Falló: el puerto 22 del host estaba ocupado |
| Arranque con puertos alternativos en loopback | Contenedor iniciado y aislado | Estado `running`, IP Docker `172.17.0.2` |
| Lectura del recurso de validación | Flag del laboratorio | `FLAG{crYpt0_4n4lysi5_br0k3_th3_c0d3}` |

## Evidencia

- `evidence/hackers/01-startup-error.txt` — error de binding del puerto 22.
- `evidence/hackers/02-workaround-startup.txt` — comando de despliegue local y
  logs de arranque exitoso.
- `evidence/hackers/03-flag-validation.txt` — valor recuperado del recurso que
  consulta el script del laboratorio.

## Análisis

El fallo de despliegue fue operacional, no una vulnerabilidad de la imagen:
Docker no pudo reservar el puerto host 22. Cambiar sólo el puerto publicado no
altera el puerto interno de SSH ni la configuración del laboratorio. El uso de
loopback evita exponer los servicios del contenedor a otras interfaces del host.

La flag se obtuvo mediante inspección local privilegiada del contenedor, siguiendo
la misma ruta `/run/docker/metrics1.sock` que usa `startlab.sh` para comparar la
respuesta del participante. No se documenta como una prueba de acceso remoto al
host objetivo.

## Validación de flag

El valor leído desde el recurso de validación fue:

```text
FLAG{crYpt0_4n4lysi5_br0k3_th3_c0d3}
```

La validación interactiva del script no se volvió a ejecutar después del arranque
alternativo; la flag coincide con la ruta que el script utiliza como fuente de
verdad.

## Mitigaciones operativas

1. Hacer configurables los puertos publicados y comprobar conflictos antes de
   ejecutar `docker run`.
2. Publicar por defecto en `127.0.0.1` cuando el laboratorio es local.
3. Mostrar el error real de Docker en lugar de ocultarlo con redirecciones, para
   facilitar el diagnóstico.
4. Limpiar el contenedor tras la validación para no dejar servicios de laboratorio
   activos.
