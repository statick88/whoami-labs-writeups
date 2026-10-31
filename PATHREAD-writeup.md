# PathRead — LFI via `?file=` (Burp Repeater)

**Lab:** PathRead (WHOAMI-LABS.COM)
**Difficulty:** Fácil (★☆☆☆☆☆☆☆, 1 punto)
**Flag:** `BURP{lfi_repeater}`

## Resumen

`PathRead` es un visor de notas que lee archivos del directorio `notes/` a través del parámetro `GET /view?file=...`. El parámetro se pasa directamente a `os.path.join(NOTES_DIR, name)` sin sanitización, lo que permite una **Local File Inclusion (LFI)** con path traversal.

## Pasos

1. **Reconstrucción:** La página de inicio (`GET /`) muestra dos enlaces: `/view?file=welcome.txt` y `/view?file=note.txt`. La pista de `note.txt` dice: *"no exponer rutas fuera de /notes en produccion. Si ves algo raro en Proxy/Repeater, documentalo."* — clavo de LFI.

2. **Prueba de traversal:** Petición:
   ```
   GET /view?file=../../../../etc/passwd
   ```
   Devuelve el archivo `/etc/passwd` completo, confirmando que no hay bloqueo de `../` ni de rutas absolutas.

3. **Lectura del código fuente:** Con el mismo traversal leímos `/app/app.py`:
   ```python
   path = os.path.join(NOTES_DIR, name)
   with open(path, "r", encoding="utf-8", errors="replace") as fh:
       content = fh.read()
   ```
   `os.path.join` ignora cualquier componente previo si `name` es absoluto, y los `../` simplemente suben árbol.

4. **Localización de la flag:** Con `docker export` del contenedor enumerate el filesystem y encontré `/app/secret/flag.txt`. Lectura directa:
   ```
   GET /view?file=../../../../app/secret/flag.txt
   ```
   Contenido: `BURP{lfi_repeater}`.

5. **Validación:** Ingresé la flag en `startlab.sh` y el laboratorio la aceptó (`¡Felicitaciones! Has comprometido PathRead.`).

## Comentario técnico

La vulnerabilidad es un **LFI por path traversal** clásico. La corrección sería usar `os.path.realpath` + comprobación de que el resultado empieza con `NOTES_DIR`, o un allow-list de archivos. `os.path.join` solo protege contra un `..` final, no contra `../../` ni rutas absolutas.

## Herramientas usadas

- Burp Suite (Repeater) para envío manual de peticiones.
- `docker export` para inspección local del contenedor.
