# Máquina Dancing

#### OS: Windows
#### Tier: Starting Point
#### Released on: 30/09/2021
#### IP de ejemplo: 10.129.108.235

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con la herramienta `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 10.129.108.235
```
```
Not shown: 65486 closed tcp ports (reset), 38 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE      REASON
135/tcp   open  msrpc        syn-ack ttl 127
139/tcp   open  netbios-ssn  syn-ack ttl 127
445/tcp   open  microsoft-ds syn-ack ttl 127
5985/tcp  open  wsman        syn-ack ttl 127
47001/tcp open  winrm        syn-ack ttl 127
49664/tcp open  unknown      syn-ack ttl 127
49665/tcp open  unknown      syn-ack ttl 127
49666/tcp open  unknown      syn-ack ttl 127
49667/tcp open  unknown      syn-ack ttl 127
49668/tcp open  unknown      syn-ack ttl 127
49669/tcp open  unknown      syn-ack ttl 127
```
Parece que hay bastantes puertos abiertos, aunque todos los que están en el rango de 47XXX-49XXX parecen puertos genéricos. Tratemos de obtener más información de todos ellos, usando los scripts predeterminados de nmap (-sC) y enumerando la versión del servicio (-sV). 
Para agilizar el proceso y no copiar todos los puertos manualmente, usamos una simple herramienta en Python3: extractports, que podéis encontrar en mi [github](https://github.com/wh1t3w1s4/extractports)
```
Discovered open ports: 135,139,445,5985,47001,49664,49665,49666,49667,49668,49669
```

```
nmap -sCV -p135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 -T4 10.129.108.235
```
```
PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 3h57m55s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-08T22:28:19
|_  start_date: N/A
```
identificamos servicios típicos de una máquina Windows, como el Service Message Block (SMB), usualmente asociado al puerto 445 o el Remote Procedure Call (RPC) en el 135.

El puerto 445 parece el más prometedor, vamos a enumerarlo usando la herramienta `smbclient` sin proporcionar credenciales usando los parámetros `[-N|--no-pass]` y `[-L|--list=HOST]`
```
smbclient -N -L //10.129.108.235
```
```
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	WorkShares      Disk     
```
la única share que no es predeterminada aquí es WorkShares, vamos a enumerarla de manera interactiva
```
smbclient -N //10.129.108.235/WorkShares
```
en SMB interactivo el uso de comandos es bastante simple, pudiendo usar tanto ls (Linux) como dir (de Windows) para ver lo que se comparte en esta share. 
```
smb: \> ls
  .                                   D        0  Mon Mar 29 10:22:01 2021
  ..                                  D        0  Mon Mar 29 10:22:01 2021
  Amy.J                               D        0  Mon Mar 29 11:08:24 2021
  James.P                             D        0  Thu Jun  3 10:38:03 2021
```
en el directorio de Amy.J hay un worknotes.txt, sin información interesante. 
```
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: worknotes.txt
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ - start apache server on the linux machine
   2   │ - secure the ftp server
   3   │ - setup winrm on dancing 
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Sin embargo en el de James.P encontramos la flag. Si lo traemos a nuestro equipo con `get flag.txt`, salimos con `exit` para leerlo localmente y lo abrimos obtenemos la flag
```
❯ cat flag.txt
<REDACTED>
```
