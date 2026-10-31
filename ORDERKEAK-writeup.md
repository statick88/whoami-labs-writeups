# ORDERKEAK — Whoami-Labs.COM

**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Categoría:** Web Exploit / Broken Access Control (BOLA/IDOR)
**Plataforma:** Linux (Docker)

---

## Resumen

Laboratorio centrado en la detección y explotación de una vulnerabilidad **BOLA (Broken Object Level Authorization)** en una API de pedidos. El endpoint `/api/orders/<id>` devolvía los datos de cada pedido sin verificar que el solicitante tuviera permiso sobre él, exponiendo un pedido "confidential" (id 15) que contenía la flag en un campo adicional no documentado.

**Flag:** `BURP{bola_comparer_api}`

---

## Análisis del entorno

Al desplegar el laboratorio se obtiene un contenedor Docker con una API Flask en `http://172.17.0.2:5000` (mapeado localmente al puerto `8240`).

Endpoints descubiertos:

| Método | Ruta | Comentario |
|--------|------|------------|
| `GET` | `/` | Página de inicio con documentación y herramientas recomendadas. |
| `GET` | `/docs` | Documentación de la API (endpoints y respuesta típica). |
| `GET` | `/status` | Healthcheck → `{"service":"OrderLeak","status":"ok"}`. |
| `GET` | `/api/orders/<id>` | Devuelve el JSON del pedido `id`. Alias: `/api/order/<id>`. |
| `GET` | `/api/orders` | Indica la sintaxis correcta para consultar un pedido. |

---

## Vulnerabilidad

### Clasificación

**BOLA / IDOR (Broken Object Level Authorization)** — OWASP API Security Top 10 (API1:2023).

### Descripción

La aplicaciónalmacenaba todos los pedidos en un diccionario Python (`ORDERS`) y el endpoint `GET /api/orders/<int:order_id>` devolvía directamente la entrada correspondiente sin aplicar ningún control de autorización por usuario. Cualquier visitante podía solicitar cualquier `id` y recibir su contenido completo.

Además, el pedido con `id = 15` (propietario `carol`) tenía `status: "confidential"` y un campo extra `flag` que **no figuraba en la documentación ni en la respuesta típica** del endpoint. Al solicitar ese id, el JSON devuelto incluía la flag sin enmascurar.

### Evidencia

Respuesta típica (documentada):

```json
{ "id": 1, "owner": "alice", "item": "Teclado mecanico RGB", "total": 79.90, "status": "shipped", "notes": "Envio estandar." }
```

Respuesta del pedido confidencial (no documentada):

```json
{ "flag": "BURP{bola_comparer_api}", "id": 15, "item": "Servidor Rack 1U + Firewall NGFW Licencia Anual", "notes": "Adquisicion clasificada infraestructura central CISO.", "owner": "carol", "status": "confidential", "total": 4500.0 }
```

---

## Explotación

No se requería autenticación ni manipulación de tokens. Basta con iterar los identificadores de pedido y observar cuál contiene datos sensibles.

```bash
# Consulta directa al pedido confidencial
curl -s http://localhost:8240/api/orders/15
# → {"flag":"BURP{bola_comparer_api}", ...}
```

Técnica recomendada por el laboratorio (Burp Suite → **Repeater** o **Intruder** con fuzzing numérico de `id`): environment el parámetro de ruta `1` al `20` y comparar las respuestas en el **Comparer** para detectar la diferencia entre un JSON normal y el JSON confidencial que incluye la flag.

---

## Flag

```
BURP{bola_comparer_api}
```

---

## Lección aprendida

- **Nunca confiar en que el cliente solo verá sus propios recursos.** Los endpoints que devuelven objetos por identificador numérico deben verificar que el solicitante esté autorizado sobre ese objeto específico.
- Los campos internos (como `flag`, `notes` internas o metadatos) no deben incluirse en respuestas genéricas; el modelo debe excluir lo que no corresponda exponer.
- La documentación es una guía, no un contrato: siempre conviene enumerar identificadores y comparar respuestas (Burp **Comparer**) para descubrir datos no documentados.