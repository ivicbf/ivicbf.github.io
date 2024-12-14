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
