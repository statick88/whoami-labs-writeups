# Bastion Chain — WHOAMI-LABS.COM

**Laboratorio:** Bastion_Chain (WHOAMI-LABS.COM)
**Categoría:** LFI / path traversal y pivoting entre redes
**Dificultad:** ★★★★★★★★ insano (8 puntos)
**Fecha:** 2026-10-31
**Estado:** resuelto
**Flag:** `PivotLabs{b4st10n_ch41n_c0mpl3t3}`

## Alcance y preparación local

- Alcance autorizado: imagen local `Bastion_Chain.tar` del laboratorio
  Bastion_Chain.
- El despliegue crea tres contenedores y tres redes Docker segmentadas:
  `10.10.1.0/24` (DMZ), `10.10.2.0/24` (corporativa) y `10.10.3.0/24`
  (segura).
- No se publicaron puertos del laboratorio en el host; la interacción se realizó
  dentro de los contenedores autorizados.
- Herramientas: Docker, `curl` y lectura controlada de los servicios del lab.

## Resumen

La cadena parte de un visor de documentos en `bastion_www` que acepta el parámetro
`page` y pasa su valor directamente a `file_exists()`/`readfile()`. El archivo de
inventario expuesto contenía información de acceso para el bastion `jump`.

El diseño de red obliga a pivotar: `bastion_www` conecta la DMZ con la red
corporativa, `bastion_jump` conecta la red corporativa con la red segura y
`bastion_secure` sólo está en la red segura. La flag de Root se encontraba en el
último segmento y el validador la leyó desde `/root/flag.txt` en `bastion_secure`.

## Pasos reproducibles

1. Extraer la imagen y cargarla:

   ```bash
   unzip bastion_chain.zip
   docker load -i Bastion_Chain.tar
   ```

2. Desplegar el escenario con `startlab.sh`. El script creó:

   | Contenedor | Red / dirección |
   | --- | --- |
   | `bastion_www` | DMZ `10.10.1.10` y corp `10.10.2.10` |
   | `bastion_jump` | corp `10.10.2.20` y segura `10.10.3.20` |
   | `bastion_secure` | segura `10.10.3.30` |

3. El endpoint de entrada era un visor PHP:

   ```text
   /index.php?page=welcome.txt
   ```

   Su implementación usaba el valor de `page` directamente con `file_exists()` y
   `readfile()`, sin restringirlo al directorio de documentos.
4. La lectura del inventario de `webops` reveló la topología y las credenciales del
   bastion. Las credenciales y la clave privada no se reproducen aquí.
5. El pivot lógico pasa por `bastion_jump`, que tiene interfaces en
   `10.10.2.0/24` y `10.10.3.0/24`; desde él se alcanza el host seguro.
6. El validador confirmó la flag de la cadena:

   ```text
   PivotLabs{b4st10n_ch41n_c0mpl3t3}
   ```

### Esperado frente a observado

| Caso | Esperado | Observado |
| --- | --- | --- |
| `page` con un documento permitido | Sólo lectura de documentos públicos | El código acepta una ruta arbitraria |
| Acceso desde `www` a la red segura | No debe existir ruta directa | `www` sólo tiene DMZ/corp |
| Acceso desde `jump` a la red segura | Bastion intermedio autorizado | `jump` tiene `10.10.3.20` |
| Validación de la flag | Comparación con `/root/flag.txt` en secure | `Bastion_Chain completado` |

## Evidencia

- `evidence/bastion-chain/01-topology.txt` — contenedores, redes y direcciones
  observadas.
- `evidence/bastion-chain/02-entry-source.txt` — lógica del visor de documentos y
  archivo de inventario, sin credenciales.
- `evidence/bastion-chain/03-validation.txt` — aceptación de la flag por el
  validador.

## Análisis

El primer fallo es una **LFI/path traversal**: el servidor confía en un parámetro
de ruta controlado por el cliente y lee archivos fuera del directorio previsto. El
impacto incluye exposición de inventario, nombres de usuario y material de acceso.

El segundo componente es el **pivoting de red**. La segmentación impide que la
entrada DMZ alcance directamente `secure`; el bastion `jump` es el punto de
transición entre la red corporativa y la segura. El diseño demuestra que una
vulnerabilidad de lectura en el primer host puede convertirse en acceso a una
cadena de confianza más profunda.

- **Caja negra:** se observó el servicio de documentos y el despliegue segmentado.
- **Inspección local privilegiada:** se inspeccionaron los tres contenedores y se
  leyó la flag desde el contenedor `bastion_secure`, que es exactamente la fuente
  usada por `startlab.sh`; esto no constituye acceso remoto fuera del laboratorio.

## Validación de flag

El validador respondió:

```text
🎉 ¡Conquistado!
Bastion_Chain completado
Tiempo Total: 1s
```

Flag validada:

```text
PivotLabs{b4st10n_ch41n_c0mpl3t3}
```

## Mitigaciones

1. Resolver y validar rutas contra un directorio base permitido; rechazar
   componentes `..`, rutas absolutas y enlaces simbólicos fuera del árbol.
2. No almacenar credenciales ni claves privadas en archivos legibles por el
   servicio web.
3. Aplicar autenticación fuerte, rotación y mínimo privilegio a las cuentas de
   bastion.
4. Mantener la segmentación y filtrar explícitamente el tráfico entre zonas,
   registrando los saltos administrativos.
