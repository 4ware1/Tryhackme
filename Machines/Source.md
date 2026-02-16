
**Resumen corto de la pwned máquina "Source" (TryHackMe):**

- Escaneo Nmap: Puerto 22 (SSH OpenSSH 7.6p1 Ubuntu) y puerto 10000 (Webmin MiniServ 1.890).
- Vulnerabilidad crítica: **Webmin 1.890** tenía un **backdoor intencional** (CVE-2019-15107), insertado maliciosamente en las descargas oficiales.
- Explotación: Usaste el módulo de Metasploit exploit/linux/http/webmin_backdoor con:
    - SSL true (porque corre sobre HTTPS),
    - ForceExploit true (para saltar chequeos),
    - Payload cmd/unix/reverse_python (o similar),
    - Target 0 (Unix In-Memory).

- Resultado: Shell **root** directo en una sola ejecución (el backdoor ejecuta comandos como root sin autenticación).
- Acceso ganado: /root/root.txt y /home/dark/user.txt (flag de usuario + root).