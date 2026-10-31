# Guestbook — WHOAMI-LABS.COM

**Laboratorio:** Guestbook (WHOAMI-LABS.COM)
**Categoría:** Stored XSS / bypass de validación del cliente
**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flags:** `user{XSS_1s_n0t_d34d_y3t}`, `root{Byp4ss_Fr0nt3nd_w1th_Burp}`

## Alcance y preparación local

- Alcance autorizado: imagen local `guestbook.tar` del laboratorio Guestbook.
- Servicio local usado para la reproducción: `http://127.0.0.1:8280`; el contenedor se
  publicó sólo en loopback. El script oficial valida también SSH en el puerto 22 y
  HTTP en el puerto 80 dentro del contenedor.
- Herramientas: `curl` y Docker.

## Resumen

El guestbook bloqueaba los caracteres `<` y `>` únicamente con JavaScript antes del
envío. El endpoint PHP no repetía esa validación y almacenaba el campo `message`
sin escapar. Una petición POST directa permitió guardar HTML/JavaScript controlado,
que el bot administrativo cargaba periódicamente con su cookie de administrador.

La página administrativa expuso credenciales de emergencia para el servidor. El
validador del laboratorio aceptó posteriormente las flags de User y Root.

## Pasos reproducibles

1. Abrir la página principal y observar que `validateForm()` rechaza `<` y `>` en
   el navegador.
2. Omitir esa validación y enviar directamente el formulario:

   ```http
   POST /index.php HTTP/1.1
   Host: 127.0.0.1:8280
   Content-Type: application/x-www-form-urlencoded

   name=auditor&message=%3Cscript%3Edocument.body.dataset.xss%3D%22stored%22%3C%2Fscript%3E
   ```

3. La respuesta `200 OK` mostró el contenido almacenado sin escapar:

   ```html
   <div style="margin-top: 5px;"><script>document.body.dataset.xss="stored"</script></div>
   ```

4. El bot administrativo visitaba la página con `admin_session`. La cookie fue
   reutilizada dentro del laboratorio para consultar `/admin.php`, que respondió
   `Authentication: SUCCESS` y mostró las credenciales de emergencia.
5. El script de validación aceptó:

   ```text
   user{XSS_1s_n0t_d34d_y3t}
   root{Byp4ss_Fr0nt3nd_w1th_Burp}
   ```

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| Mensaje con `<` o `>` desde el navegador | Rechazo antes del envío | El filtro sólo existía en JavaScript |
| POST directo con HTML | Entrada rechazada o escapada en servidor | HTML/JavaScript almacenado y reflejado sin escapar |
| Acceso a `/admin.php` sin cookie | `403 Forbidden` | Con la cookie del bot, `Authentication: SUCCESS` |
| Validación de flags | Rechazo de valores incorrectos | User aceptada y el validador mostró `¡Felicitaciones! Has comprometido Guestbook a nivel Fácil.` al aceptar Root |

## Evidencia

- `evidence/guestbook/01-post-request.txt` — POST directo que evita la validación
  del navegador.
- `evidence/guestbook/02-stored-response.txt` — respuesta que muestra el payload
  almacenado sin escapar.
- `evidence/guestbook/03-admin-response.txt` — acceso administrativo observado,
  con credenciales redactadas.
- `evidence/guestbook/04-validation.txt` — aceptación de User y Root por el
  validador del laboratorio.

## Análisis

La vulnerabilidad principal es un **stored XSS** causado por confiar en una
validación del cliente. `index.php` aplicaba `htmlspecialchars()` al nombre, pero
insertaba `message` sin escaping y después lo imprimía directamente en la página.
El bot administrativo visitaba el guestbook con una cookie privilegiada, por lo que
un payload persistente podía ejecutarse en su contexto.

El acceso a `/admin.php` dependía de una cookie estática predecible y daba acceso a
credenciales operativas, lo que ampliaba el impacto desde ejecución JavaScript en
el navegador del bot hasta acceso al servidor. Las credenciales y cookies no se
incluyen en este writeup ni en la evidencia.

- **Caja negra:** POST directo, HTML persistido, respuesta administrativa y
  aceptación de las flags.
- **Inspección local privilegiada:** lectura de `index.php`, `admin.php`, `bot.py` y
  los archivos de flags del contenedor para entender el escenario. No es evidencia
  de acceso remoto por sí sola.

## Validación de flags

El validador respondió que la flag de User era correcta y después mostró:

```text
[+] Flag de usuario correcta. ¡Buen trabajo!
¡Felicitaciones! Has comprometido Guestbook a nivel Fácil.
```

Flags validadas:

```text
user{XSS_1s_n0t_d34d_y3t}
root{Byp4ss_Fr0nt3nd_w1th_Burp}
```

## Mitigaciones

1. Validar y sanear entradas en el servidor; escapar el contenido al renderizarlo
   con el contexto correcto.
2. Aplicar una política CSP estricta y evitar que el bot administrativo navegue con
   cookies privilegiadas en páginas con contenido no confiable.
3. No usar cookies administrativas estáticas ni predecibles; rotarlas y protegerlas
   con `HttpOnly`, `Secure` y `SameSite`.
4. No mostrar credenciales de emergencia en una página web; usar un almacén de
   secretos y limitar las cuentas y permisos del servicio.
