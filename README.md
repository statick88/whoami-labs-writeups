# WHOAMI Labs — writeups

Writeups personales de laboratorios resueltos. Los resultados y flags de abajo se
reportan desde los writeups y sus evidencias locales.

## Retos documentados

Hay **13 writeups locales**: 12 ya figuraban en el índice y React Dashboard estaba
omitido. Este registro no acredita por sí solo resolución independiente, envío de
flags ni puntos en el ranking oficial.

| Laboratorio | Categoría | Writeup |
| --- | --- | --- |
| PathRead | LFI / path traversal | [PATHREAD-writeup.md](PATHREAD-writeup.md) |
| OrderLeak | BOLA / IDOR | [ORDERKEAK-writeup.md](ORDERKEAK-writeup.md) |
| Token Forge | JWT | [TOKEN-FORGE-writeup.md](TOKEN-FORGE-writeup.md) |
| ProfilePeek | IDOR / BOLA por header | [PROFILEPEEK-writeup.md](PROFILEPEEK-writeup.md) |
| SigninBleed | SQL injection en login | [SIGNINBLEED-writeup.md](SIGNINBLEED-writeup.md) |
| Guestbook | Stored XSS / bypass de validación | [GUESTBOOK-writeup.md](GUESTBOOK-writeup.md) |
| HACKERS | Reconocimiento y análisis criptográfico | [HACKERS-writeup.md](HACKERS-writeup.md) |
| Bastion Chain | LFI y pivoting entre redes | [BASTION-CHAIN-writeup.md](BASTION-CHAIN-writeup.md) |
| Cerberus | Cadena WAF / BOLA / command injection / JWT | [CERBERUS-writeup.md](CERBERUS-writeup.md) |
| OPERATOR | Escalada mediante backup y grupo disk | [OPERATOR-writeup.md](OPERATOR-writeup.md) |
| JUMPING | Pivoting entre redes Docker y SSH | [JUMPING-writeup.md](JUMPING-writeup.md) |
| PROXY | HTTP request smuggling y escalada con LD_PRELOAD | [PROXY-writeup.md](PROXY-writeup.md) |
| Matrix | JWT / command injection | [MATRIX-writeup.md](MATRIX-writeup.md) |
| LABERINTO | SSH / socket tmux | [LABERINTO-writeup.md](LABERINTO-writeup.md) |
| DevOps React Dashboard | Command injection / RCE | [REACT_LAB-writeup.md](REACT_LAB-writeup.md) |

## Plan de entrenamiento — Comunidad

Plan actualizado contra los writeups locales y las fuentes oficiales consultadas
el **2026-10-31**. La prioridad es cerrar brechas, no acumular flags de nivel Insano.

### Alcance gratuito confirmado

La [página oficial de planes](https://whoami-labs.com/planes), consultada con
navegador porque el HTML estático no muestra la tabla completa, confirma:

- **$0 / siempre gratis**, sin tarjeta ni costos ocultos; entrenamiento local.
- **+150 laboratorios descargables Docker y OVA**, CTF pública y ranking por puntos.
- Base de writeups de la comunidad y **Mentor IA: hasta 15 consultas diarias**.
- La oferta describe niveles Principiante, Intermedio y Avanzado; las fichas del
  catálogo usan Easy, Medium, Hard e Insane. Conservar la dificultad de cada ficha,
  sin inventar una equivalencia entre ambas clasificaciones.
- Cloud en un clic, mentor de 100 consultas/día y simulador de entrevistas son
  prestaciones Pro: **no son dependencias de este plan**.

El [catálogo oficial para IA](https://whoami-labs.com/llms-full.txt) declara
**158 laboratorios**, con fecha de generación **2026-10-26**. Es una instantánea,
no un contador en vivo. No corresponde restarle los 13 writeups para deducir
pendientes: React Dashboard no aparece con ese nombre en ese corpus y falta
conciliar identidades y completados de la cuenta.

### Punto de partida y brechas

| Área | Evidencia local | Qué falta demostrar |
| --- | --- | --- |
| Web/API | JWT, BOLA/IDOR, SQLi, LFI, XSS, RCE y smuggling | Resolver variantes sin conocer previamente rutas, código ni flags |
| Escalada Linux | OPERATOR y PROXY: backups/grupo disk y LD_PRELOAD | Acceso inicial y escalada como usuario sin privilegios; fundamentos SUID/sudo |
| Pivoting | Bastion Chain y JUMPING documentados | Repetir descubrimiento y saltos desde el atacante, sin inspección privilegiada |
| Cripto/reconocimiento | HACKERS registra recuperación local de flag | Resolver el vector previsto; leer el recurso del validador no acredita criptoanálisis |
| Cron y Capabilities | Sin writeup dedicado | Al menos una resolución reproducible de cada categoría |
| Windows | Sin writeup dedicado | Enumeración y escalada en una OVA aislada |

**Calidad de cierre:** HACKERS declara lectura privilegiada del recurso de
validación; Bastion Chain y JUMPING también usan inspección de contenedores.
PathRead localizó la ruta mediante `docker export`, aunque la lectura final fue
por LFI. No borrar esos logros: registrar la ayuda y repetir sin ella para medir
autonomía. React Dashboard documenta una respuesta HTTP con flag, pero no un
resultado del validador ni evidencia archivada. SigninBleed tampoco registra
reejecución del validador. No presentar estos cierres como envíos oficiales.

### Ruta priorizada

Los siguientes son **candidatos de entrenamiento**, no laboratorios iniciados.
Categorías y dificultades provienen del catálogo oficial; comprobar requisitos y
descarga de la ficha antes de desplegar.

| Orden | Trabajo / candidato | Objetivo y criterio de cierre |
| --- | --- | --- |
| 0 | Conciliar los 13 writeups y el perfil | Registrar por reto: ayuda usada, evidencia, resultado del validador y envío oficial solo si fue observado |
| 1 | [ARTEMIS](https://whoami-labs.com/laboratorios/artemis) — Easy, SUID & Sudo | Enumerar permisos y explicar una escalada reproducible desde usuario sin privilegios |
| 2 | [BREACH](https://whoami-labs.com/laboratorios/breach) — Easy, Capabilities | Identificar capabilities y demostrar la precondición, abuso y mitigación |
| 3 | [ACADEMIA](https://whoami-labs.com/laboratorios/academia) — Medium, Cron | Identificar tarea, usuario y permisos; demostrar escalada y por qué funciona |
| 4 | [VULNERABLE 3](https://whoami-labs.com/laboratorios/vulnerable-3) — Medium, Web Exploit | Añadir file upload: acceso inicial, shell y escalada documentados |
| 5 | [BEKEEPER](https://whoami-labs.com/laboratorios/bekeeper) — Medium, Pivoting & Lateral Movement | Descubrir y alcanzar el segmento interno con túnel real; después repetir Bastion Chain/JUMPING |
| 6 | [WinServices](https://whoami-labs.com/laboratorios/winservices) — Medium, Network & Defense Evasion | Enumerar servicios y permisos Windows; verificar formato y usar OVA con red aislada |
| 7 | Revisión HACKERS + simulacro CTF público | Resolver el vector pendiente y practicar bajo tiempo; registrar ayudas, bloqueos y resultados reales |

Avanzar por evidencia: antes de volver a Hard/Insane, explicar acceso inicial,
escalada y mitigación sin leer código o flags del objetivo con privilegios del host.
Si hace falta inspección local, marcar la sesión como guiada y repetirla después.

### Rutina y seguimiento

1. Seleccionar un solo laboratorio; usar Docker local o una OVA aislada según su
   ficha. No exponer servicios vulnerables fuera del entorno de entrenamiento.
2. Enumerar, formular una hipótesis y guardar peticiones/respuestas o salidas
   antes de pedir ayuda. Usar el mentor para orientación contextual, dentro del
   límite de 15 consultas/día, no como sustituto del razonamiento.
3. Consultar writeups comunitarios después del intento propio; anotar qué ayuda
   desbloqueó el reto y qué debe repetirse sin ella.
4. Cerrar con writeup, evidencia redactada, resultado real del validador y
   mitigación. Registrar puntos solo si la plataforma confirma el envío.
5. Detener los contenedores o VM al terminar y actualizar este índice y
   `.progress`. La resolución de los candidatos queda para sesiones posteriores.

### Fuentes y límites

- [Planes y comparativa oficial](https://whoami-labs.com/planes): condiciones
  Comunidad y separación de Pro, verificadas en la página renderizada.
- [Índice oficial](https://whoami-labs.com/): acceso local, writeups, ranking y mentor.
- [Catálogo oficial completo](https://whoami-labs.com/llms-full.txt): nombres,
  categorías, dificultades y prerrequisitos de los candidatos.
- [CTF pública](https://whoami-labs.com/ctf),
  [ranking](https://whoami-labs.com/ranking) y
  [writeups](https://whoami-labs.com/writeups): recursos enlazados por el índice
  oficial; no se verificaron puntos, posición ni completados de la cuenta.

## Para próximos laboratorios

- Usá un [writeup existente](PROFILEPEEK-writeup.md) como referencia para objetivo,
  alcance, pasos, validación y mitigaciones.
- Guardá capturas y respuestas redactadas bajo `evidence/<reto>/` y enlazalas
  desde el writeup; no incluyas cookies ni credenciales personales.
- Trabaja los contenedores sólo en `127.0.0.1`; no los expongas en todas las interfaces.
