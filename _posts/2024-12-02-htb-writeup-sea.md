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

