# ProfilePeek — WHOAMI-LABS.COM

**Laboratorio:** ProfilePeek (WHOAMI-LABS.COM)
**Categoría:** IDOR / BOLA mediante header de identidad
**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `BURP{idor_header_scope}`

## Alcance y preparación local

- Alcance autorizado: imagen local `profilepeek.tar` del laboratorio ProfilePeek.
- Servicio local: `http://127.0.0.1:8230`; el contenedor se publicó sólo en loopback.
- Herramientas: `curl`, Docker y lectura del código de la imagen para comprender el flujo.

## Resumen

La aplicación permitía iniciar sesión con la cuenta demo `alice / alice1`. El endpoint
`/profile` aceptaba el header `X-User-Id` y lo trataba como una identidad confiable:
al enviar `X-User-Id: 17`, la aplicación seleccionaba el perfil de Carol y activaba
el modo elevado, aunque la sesión pertenecía a Alice. El perfil 17 expuso el campo
restringido que contenía la flag.

## Pasos reproducibles

1. Abrir `/login` y enviar `username=alice&password=alice1`.
2. Conservar la cookie de sesión y solicitar:

   ```http
   GET /profile?uid=1 HTTP/1.1
   Host: 127.0.0.1:8230
   ```

   La respuesta confirmó `Sesión actual: alice`.
3. Repetir la petición con el header controlado:

   ```http
   GET /profile?uid=1 HTTP/1.1
   Host: 127.0.0.1:8230
   X-User-Id: 17
   ```

4. La respuesta cambió al perfil 17, mostró `Modo elevado activo (header)` y reveló
   `BURP{idor_header_scope}` en el campo restringido.

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Sesión Alice, sin header | Sólo el perfil solicitado y sin privilegios elevados | Perfil 1, sesión Alice |
| Sesión Alice, `X-User-Id: 17` | El servidor debe ignorar el identificador no autenticado o rechazarlo | Perfil 17 de Carol y campo restringido visible |

## Evidencia

- `evidence/profilepeek/01-login.txt` — petición de login y redirección observada, sin cookie.
- `evidence/profilepeek/02-header-response.txt` — respuesta relevante con el header de identidad y la flag.
- `evidence/profilepeek/03-validation.txt` — aceptación de la flag por el script del laboratorio.

## Análisis

La vulnerabilidad es un **IDOR/BOLA** combinado con una confianza incorrecta en
headers de identidad. `resolve_target()` toma `X-User-Id` directamente, sustituye
el `uid` solicitado y marca la petición como elevada. No existe una comprobación de
que el usuario autenticado pueda actuar como el UID indicado ni de que el proxy que
debería establecer ese header sea confiable.

- **Caja negra:** el login con la cuenta demo y la consulta HTTP con
  `X-User-Id: 17` devolvieron el perfil de Carol y el campo restringido.
- **Inspección local privilegiada:** la lectura del código de la imagen confirmó la
  lógica que convierte ese header en modo elevado; esto explica el comportamiento,
  pero no sustituye la evidencia HTTP.

## Validación de flag

Tras iniciar sesión con la cuenta verificada `alice / alice1`, se introdujo la flag
en el validador del laboratorio. El script respondió:

```text
¡Felicitaciones! Has comprometido ProfilePeek.
```

La flag validada fue `BURP{idor_header_scope}`.

## Mitigaciones

1. No aceptar headers de identidad enviados por el cliente; establecerlos sólo en un
   proxy autenticado y eliminar cualquier valor entrante antes de reenviarlo.
2. Derivar el usuario objetivo de la sesión autenticada y comprobar autorización
   objeto por objeto antes de devolver perfiles o campos restringidos.
3. No usar un header controlable por el cliente para activar privilegios elevados.
4. Añadir pruebas que intenten leer el perfil 17 desde la sesión de Alice y esperen
   una respuesta de denegación o ausencia del campo secreto.
