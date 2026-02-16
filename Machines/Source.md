Maquina Source

Nmap

<img width="318" height="137" alt="image" src="https://github.com/user-attachments/assets/7d92167f-a75b-4678-8e18-dd0d27b43126" />

`Puertos: 10000,22`

Búsqueda de Exploits (Searchsploit)

Utilice searchsploit para buscar vulnerabilidades conocidas asociadas a la versión de Webmin:

searchsploit webmin 1.890

<img width="649" height="367" alt="image" src="https://github.com/user-attachments/assets/9f1bc58e-548d-4f10-8eff-0988a0e83f3c" />

Se identificó la vulnerabilidad CVE-2019-15107, un backdoor intencional en el parámetro password_change.cgi que permite la ejecución remota de comandos (RCE) sin autenticación previa.
3. Explotación

Implementación en Metasploit

Se seleccionó el módulo correspondiente para explotar el backdoor de Webmin:


`msfconsole
use exploit/linux/http/webmin_backdoor`

Al ejecutarse, el exploit aprovecha la vulnerabilidad para devolver una sesión con privilegios de root de forma directa.
Estabilización de la Shell


<img width="79" height="30" alt="image" src="https://github.com/user-attachments/assets/a7fabef5-e525-4695-b8ae-e38c815359c6" />


4. Post-Explotación y Flags

Dado que el acceso inicial proporcionó directamente privilegios de superusuario, no fue necesario realizar una escalada de privilegios adicional. Se procedió a la recolección de los objetivos:

Localización de Flags:

    User Flag: Ubicada en el directorio personal del usuario.
<img width="347" height="22" alt="image" src="https://github.com/user-attachments/assets/ca72244c-4119-4ed8-ba56-07c189bc0e92" />


cat /home/dark/user.txt

    Root Flag: Ubicada en el directorio root.
<img width="175" height="33" alt="image" src="https://github.com/user-attachments/assets/d217bb85-e908-4f68-9016-ca25a84a93d7" />


cat /root/root.txt

Conclusión

La resolución de la máquina Source destaca la importancia de mantener actualizado el software de administración. El uso de versiones comprometidas con backdoors conocidos permite un compromiso total del sistema en pocos pasos. Este laboratorio refuerza el flujo de trabajo básico: reconocimiento preciso, investigación de versiones y explotación dirigida.
