# SigninBleed — WHOAMI-LABS.COM

**Laboratorio:** SigninBleed (WHOAMI-LABS.COM)
**Categoría:** Inyección SQL en autenticación
**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `SigninBleed{sqli_proxy_decoder}`

## Alcance y preparación local

- Alcance autorizado: imagen local `signinbleed.tar` del laboratorio SigninBleed.
- Servicio local usado: `http://127.0.0.1:8240`; el puerto interno HTTP 80 se publicó
  sólo en loopback. El script oficial del laboratorio usa el puerto local 8220.
- Herramientas: `curl` y Docker.

## Resumen

El formulario `POST /login` construía la consulta SQL concatenando directamente los
valores de `username` y `password`. Una condición siempre verdadera permitió iniciar
sesión sin credenciales válidas y acceder a `/note`, donde la aplicación mostró la
nota privada y la flag.

## Pasos reproducibles

1. Solicitar `/login` para identificar el formulario de autenticación.
2. Enviar una entrada de usuario que cierre la cadena SQL y fuerce una condición
   verdadera:

   ```http
   POST /login HTTP/1.1
   Host: 127.0.0.1:8240
   Content-Type: application/x-www-form-urlencoded

   username=%27+OR+1%3D1+--+
   password=x
   ```

   La petición recibió `302 FOUND` con `Location: /note` y una cookie de sesión.
3. Reutilizar la cookie en `GET /note`. La respuesta `200 OK` mostró el contenido
   de la nota privada y `SigninBleed{sqli_proxy_decoder}`.

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Credenciales incorrectas | Rechazo y mensaje de error | La aplicación normalmente rechaza el login |
| Entrada SQL manipulada | El valor debe tratarse como texto literal | Redirección a `/note` y sesión autenticada |
| `GET /note` con la sesión obtenida | Sólo tras autenticación válida | Nota privada y flag visibles |

## Evidencia

- `evidence/signinbleed/01-login-request.txt` — petición de login manipulada, sin
  cookie ni credenciales reales.
- `evidence/signinbleed/02-login-response.txt` — redirección y sesión creada, con
  el valor de cookie redactado.
- `evidence/signinbleed/03-note-response.txt` — respuesta de `/note` que contiene
  la flag.

## Análisis

La vulnerabilidad es una **inyección SQL (SQLi)** en el flujo de autenticación. La
aplicación construía una consulta equivalente a:

```python
SELECT username, note, flag FROM users
WHERE username = '<entrada>' AND password = '<entrada>' LIMIT 1
```

Con `username=' OR 1=1 -- `, el resto de la condición queda comentado y la consulta
devuelve el primer usuario. La sesión guarda entonces `user`, `note` y `flag`, que
`/note` renderiza sin otra comprobación.

- **Caja negra:** el login manipulado devolvió una redirección a `/note`, y `/note`
  respondió con la nota y la flag.
- **Inspección local privilegiada:** la lectura del código de la imagen confirmó la
  concatenación de SQL y la carga de `/root/flag.txt`; esto explica el resultado,
  pero no sustituye la evidencia HTTP.

## Validación de flag

La flag fue observada en la respuesta HTTP autenticada de `/note`:

```text
SigninBleed{sqli_proxy_decoder}
```

El script del laboratorio compara la entrada con el contenido de
`/root/flag.txt`. En esta sesión no se volvió a ejecutar el validador interactivo
después de documentar la respuesta HTTP.

## Mitigaciones

1. Usar consultas parametrizadas (`?`/bindings) en lugar de concatenar entradas en
   SQL.
2. Mantener mensajes de error de autenticación uniformes y no crear sesión hasta
   que la consulta parametrizada devuelva una coincidencia válida.
3. Añadir pruebas de regresión con comillas, comentarios SQL y operadores booleanos.
4. No guardar ni renderizar secretos en la sesión si la vista no los necesita.
