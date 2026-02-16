Nmap

```
nmap
# Nmap 7.95 scan initiated Fri Feb 13 17:03:29 2026 as: /usr/lib/nmap/nmap --privileged -Pn -n -sV -sC -O -p21,22,80 -oN recon_10.64.167.152/full_enum.txt 10.64.167.152
Nmap scan report for 10.64.167.152
Host is up (0.17s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 70:9a:32:7d:23:03:9e:a2:1f:61:ef:9b:92:42:7c:97 (RSA)
|   256 1b:27:d1:e3:5a:8c:20:15:72:99:13:b3:9d:c9:20:7c (ECDSA)
|_  256 6e:14:f0:c1:bc:27:84:70:66:21:b9:a6:92:b9:47:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works! If you see this add 'te...
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|specialized|phone|storage-misc
Running (JUST GUESSING): Linux 4.X|5.X|3.X (91%), Crestron 2-Series (86%), Google Android 10.X|11.X|12.X (85%), HP embedded (85%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:crestron:2_series cpe:/o:linux:linux_kernel:3 cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/h:hp:p2000_g3
Aggressive OS guesses: Linux 4.15 - 5.19 (91%), Linux 4.15 (90%), Linux 5.4 (90%), Crestron XPanel control system (86%), Linux 3.8 - 3.16 (86%), Android 10 - 12 (Linux 4.14 - 4.19) (85%), HP P2000 G3 NAS device (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Feb 13 17:03:48 2026 -- 1 IP address (1 host up) scanned in 18.54 seconds

```

## Puerto 80

## Informacion del Servicio

```
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

## HTTP Headers

```
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works! If you see this add 'te...
```

![[Pasted image 20260214145225.png]]


###### Gobuster salida
```
/index.html           
/images                
/scripts              
/assets              
/robots.txt           
/server-status     
```
![[Pasted image 20260214145351.png]]
###### wfuzz Subdomains
```
"www"                      
"dev"                      
"www.dev"    
```

Endpoint con posible lfi
![[Pasted image 20260214145517.png]]

Confirmacion de LFI
![[Pasted image 20260214145551.png]]

Hice otro wfuzz para confirmar que archivos puedo ver en el sistema; este es el mas interesantes
```
 "/etc/ssh/sshd_config"     
```
Dentro del archivo se encuentra un id_rsa que le pertence al usuario dale
![[Pasted image 20260214145812.png]]
Despues de ingresar como dale buscamos la primera flag
![[Pasted image 20260214150020.png]]

Podemos ver que dale tiene permisos para ejecutar ese script
```shell
sudo -l
   (gyles) NOPASSWD: /home/gyles/admin_checks
```


Podemos ver que es lo que tiene dentro el script, y podemos observar un problema en la variable "error" nos permite injectar comandos como como el usuario gyale entonces nos lanzamos una shell /bin/bash/ y tenemos accesos yo me mande una revshell directamente para que sea mas interactiva la consola
```shell
dale@ip-10-64-167-152:/home/gyles$ cat /home/gyles/admin_checks
#!/bin/bash

printf "Reading stats.\n"
sleep 1
printf "Reading stats..\n"
sleep 1
read -p "Enter name of person backing up the data: " name
echo $name  >> /var/stats/stats.txt
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null

date_save=$(date "+%F-%H-%M")
cp /var/stats/stats.txt /var/stats/stats-$date_save.bak

printf "Stats have been backed up\n"

```

gyale pertence al grupo de admin y puede modificar los archivos mencionados, existe una crontab que ejecuta main_backup.sh cada un minuto entonces procedo a modificar el archivo me mando una revshell
```
 Group admin:
/usr/local/bin                                                                               
/usr/local/bin/main_backup.sh
/opt/admin_stuf
```

![[Pasted image 20260214150917.png]]


###### Aprendizaje

Importante siempre wfuzz para subdominios, es lo que siempr me falta de aplicar en la metodologia para buscar entradas siempre tener en cuenta, y despues el priv escalado fue sencillo, detecte otra forma para escalar en el PATH= estaba mal configurado usr/local/bin/ entonces podia hacer un pathhijacking dentro de esas carpetas todo se ejecutaria como root