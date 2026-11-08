# Máquina Fawn

#### Tier: Starting Point
#### Released on: 30/09/2021
#### IP de ejemplo: 10.129.109.33

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con la herramienta `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 10.129.109.33
```
```
Not shown: 65529 closed tcp ports (reset), 5 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack ttl 63
```
vemos un único puerto abierto. Tratemos de obtener más información del mismo, usando los scripts predeterminados de nmap (-sC) y enumerando la versión del servicio (-sV)
```
nmap -sCV -p21 -T4 10.129.109.33
```
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.223
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
Service Info: OS: Unix

```
podemos ver bastante información interesante; por ejemplo, el `anonymous login` de FTP está activado, por lo que podemos entrar como el usuario `anonymous` sin proporcionar contraseña. Además, podemos ver que la flag.txt está dentro del servicio FTP gracias al script `ftp-anon.nse` de nmap, que se usa por el argumento -sC. 

Procedemos a entrar con el usuario anonymous, sin proporcionar contraseña: `ftp 10.129.109.33`
Cuando salga el prompt para poner usuario pondremos anonymous, y en el campo de contraseña simplemente daremos a la tecla enter
```
Connected to 10.129.109.33.
220 (vsFTPd 3.0.3)
Name (10.129.109.33:w1s4): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
veamos los archivos dentro de este servicio con el comando `ls`, al igual que en Linux
```
229 Entering Extended Passive Mode (|||16742|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```
solo falta leer la flag.txt, para ello la traemos a nuestro equipo con `get flag.txt`. Posteriormente escribimos `exit` para salir. La flag debería estar en nuestro directorio actual de trabajo
```
❯ cat flag.txt
<REDACTED>
```
