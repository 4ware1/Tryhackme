# CTF Writeup - Poster (TryHackMe)

## 📋 Información General

|Campo|Valor|
|---|---|
|**Plataforma**|TryHackMe|
|**Dificultad**|Easy|
|**OS**|Linux (Ubuntu)|
|**IP**|`10.67.138.72`|
|**Fecha**|03/02/2026|

Exportar a Hojas de cálculo

**Tags:** #ctf #postgresql #bruteforce #privesc #sudo #metasploit

---

## 🎯 Objetivo

- [x] **User Flag**: `THM{postgresql_fa1l_conf1gurat1on}`
    
- [x] **Root Flag**: `THM{c0ngrats_for_read_the_f1le_w1th_credent1als}`
    

---

## 1. Reconocimiento (Enumeración)

### Escaneo de Puertos (Nmap)

El escaneo de servicios identificó los siguientes puertos abiertos:

Plaintext

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp   open  http       Apache httpd 2.4.18 ((Ubuntu))
5432/tcp open  postgresql PostgreSQL DB 9.5.8 - 9.5.10
```

**Análisis:** El servicio PostgreSQL (5432) se identificó como el objetivo principal para la intrusión inicial.

---

## 2. Acceso Inicial

### Fuerza Bruta a PostgreSQL

Se realizó un ataque de fuerza bruta contra el servicio PostgreSQL para identificar credenciales válidas.

- **Credenciales obtenidas:** `postgres:password`
    

### Explotación vía PostgreSQL

Con acceso de superusuario a la base de datos, se utilizó el módulo de Metasploit `admin/postgres/postgres_readfile` para interactuar con el sistema de archivos del servidor.

**Pasos:**

1. Se configuró el módulo con `set RFILE /home/dark/credentials.txt`.
    
2. El archivo reveló las credenciales de un usuario del sistema:
    
    - **Usuario:** `dark`
        
    - **Contraseña:** `qwerty1234#!hackme`
        

---

## 3. Movimiento Lateral

Se estableció una conexión SSH utilizando las credenciales del usuario `dark`.



```
ssh dark@10.67.138.72
```

### Enumeración Post-Explotación

Se ejecutó el script **LinPeas** para identificar vectores de escalada de privilegios. El script localizó un archivo de configuración crítico en el servidor web:

- **Ruta:** `/var/www/html/config.php`
    
- **Contenido:** `$dbpass = "p4ssw0rdS3cur3!#";`
    

Dada la alta probabilidad de reutilización de contraseñas, se probó esta clave para el usuario **alison**, logrando acceso exitoso a su cuenta mediante `su alison`.

---

## 4. Escalación de Privilegios a Root

### Abuso de Sudoers

Tras obtener acceso como `alison`, se verificaron sus privilegios de `sudo`:



```
alison@ubuntu:~$ sudo -l
User alison may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

Al tener permisos totales, se procedió a elevar privilegios a root:


```
sudo su -
whoami # root
```

**Root Flag:** `THM{c0ngrats_for_read_the_f1le_w1th_credent1als}`

---

## 🔑 Resumen de Credenciales

|Usuario|Contraseña|Fuente / Ubicación|
|---|---|---|
|`postgres`|`password`|Fuerza Bruta (DB)|
|`dark`|`qwerty1234#!hackme`|`/home/dark/credentials.txt`|
|`alison`|`p4ssw0rdS3cur3!#`|`/var/www/html/config.php`|

Exportar a Hojas de cálculo

---

## 💡 Lecciones Aprendidas

1. **Seguridad de Contraseñas:** El uso de una contraseña débil (`password`) en un servicio crítico permitió el acceso inicial mediante fuerza bruta.
    
2. **Exposición de Archivos:** Dejar archivos con credenciales en texto claro en el home de los usuarios (`credentials.txt`) facilita el movimiento lateral.
    
3. **Privilegios de Sudo:** Configurar una cuenta de usuario con `(ALL : ALL) ALL` elimina todas las capas de seguridad del sistema operativo.
