# SHADOWMAP — WHOAMI-LABS.COM

**Laboratorio:** SHADOWMAP  
**Categoría:** OSINT / mapeo de superficie  
**Dificultad:** insano (8 XP)  
**Estado:** resuelto

## Metodología

El laboratorio proporciona un entorno atacante con DNS, WHOIS y una red interna
aislada. El briefing define la flag como:

```text
SHADOWMAP{subdomains_iana_webtech_openports_hosts_os_distro}
```

### Subdominios

La enumeración con `Domain_OSINT.sh whoami-corp.local` y la transferencia de
zona devolvió 17 nombres con registro A o CNAME válido:

```text
www portal demo api dev backup intranet support cdn mail smtp ftp ssh whois
vpn old-vpn files admin
```

### WHOIS

```text
Registrar IANA ID: 4242
```

### Tecnología web

```text
Server: nginx/1.24.0 (Ubuntu)
```

El segmento requerido es el nombre corto en minúsculas: `nginx`.

### Puertos del host web

El host web es `10.30.3.20`. La enumeración completa encontró:

```text
21/tcp  ftp
22/tcp  ssh
80/tcp  http
```

Por tanto, el total es `3`.

### Hosts activos

La red `10.30.3.0/24` mostró cinco hosts vivos, contando el entorno atacante:

```text
10.30.3.10  DNS
10.30.3.20  web
10.30.3.25  mail/storage
10.30.3.35  WHOIS
10.30.3.50  atacante
```

El segmento `hosts` es `5`.

### Sistema operativo y distribución

Nmap identificó Linux en los servicios del objetivo. Los banners de SSH y HTTP
identificaron Ubuntu:

```text
OpenSSH 9.6p1 Ubuntu
nginx 1.24.0 (Ubuntu)
```

Los segmentos finales son `linux_ubuntu`.

## Flag

```text
SHADOWMAP{17_4242_nginx_3_5_linux_ubuntu}
```
