---
title: HackTheBox Traverxec
date: 2021-07-08 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/banner.jpg 
  height: 1100
  width: 500
categories: [HTB Writeups, Linux Easy]
tags: [less, scripts, sudo, journalctl, ssh, nostromo]
---

***

**Machine IP** : 10.10.10.165

**DATE**  : 08/07/2021

***

## Matriz de la maquina


Esta matriz nos muestra las características de explotación de la maquina.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/matrix.png)

## Reconocimiento


Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes.

## Nmap 

Usamos el siguiente comando para escanear todos los puertos de una manera rapida.

```bash
nmap -p- --open -T5 -v -n -Pn 10.10.10.165
```

Posteriormente utilizamos este comando con los puertos del anterior escaneo para saber las versiones de cada servicio.

```bash
nmap -sC -sV -n -p22,80 -Pn 10.10.10.165
```

Nos arroja este resultado:

```bash
~/HTB/traverxex 59s ❯ nmap -sC -sV -n -p22,80 -Pn 10.10.10.165                                     59s
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-05 05:16 EDT
Nmap scan report for 10.10.10.165
Host is up (1.1s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 73.06 seconds
```

Podemos observar dos puertos abiertos, el 22 que pertenece a ssh y el 80 que es un http con la version `nostromo 1.9.6`, procederemos a revisar la web que corre en el servicio http y haremos una busqueda de directorio con `gobuster`.

Luego de fuzzear la web con `gobuster`, no encontré nada interesante, por eso busqué la version y teconología web que utiliza esta web con el comando

```bash
whatweb 10.10.10.165
```

Me arroja el siguiente resultado

```bash
http://10.10.10.165 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[nostromo 1.9.6], IP[10.10.10.165], JQuery, Script, Title[TRAVERXEC]
```

Luego de buscar la version nostromo 1.9.6 en google encontre un cve de una vulnerabilidad `RCE(Remote Code Execution)`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/exploitdb.png)

Copié el codigo a un archivo .py y ejecuté el script, luego me dio una shell como `www-data` y haciendole el tratamiento de la `tty`, quedaría así.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/wwwdata.png)

Luego de un momento de analizar directorio por directorio encontré archivos de configuracion que tenian el `hash` de `david`, un usuario del sistema, y tenia rutas de directorios ocultos pertenecientes a david.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/nhttpd.png)

Con el hash encontré utilizé `john` para poder crackearlo y obtuve `Nowonly4me` como password, que seria la clave de acceso a la pagina `protected-file-area`, que está dentro del directorio `www_public` del usuario `david`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/davidhash.png)

Luego de desencriptarlo...

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/deshashed.png)

Luego de acceder a la pagina con el usuario `david` y contraseña `Nowonly4me`, pude descargar el comprimido que contenia la `clave privada` de david. 

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/protected-file.png)

Luego hize el mismo procedimiento para obtener la password, usé `ssh2john` para crackear la password y con el `id_rsa` pude logearme via `ssh` y obtener el `user.txt`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/user.png)


## Escalamiento de privilegios

Luego de obtener acceso como el usuario david, probé con `sudo -l` para listar los binarios que se pueden ejecutar como sudo, pero no encontré nada, luego inspeccioné los binarios con permisos `SUID`, pero tampoco encontré un vector de ataque.

Pero inspeccionando los directorios de david, dentro de la carpeta `bin` encontré dos archivos, uno `server-stats.head` y el otro `server-stats.sh`, dentro del head no habia más que  un banner, y en el otro habian comandos llamando a `binarios`, en la ultima linea llamaba a `sudo`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/davidbin.png)

El script devuelve las últimas 5 líneas de los registros del servicio `nostromo` usando `journalctl`. Esto es explotable porque `journalctl` invoca el buscador predeterminado, que probablemente sea `less`. El comando `less` muestra la salida en la pantalla del usuario y espera la entrada del usuario una vez que se muestra el contenido. Este puede explotarse ejecutando un comando de `shell`.

Ejecutamos el comando para invocar a `less`

```bash
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
```
Y en esta ventana de input ingresamos `!/bin/bash` para invocar una shell como `root`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/traverxec/root.png)


## Conclusiones


Fue una maquina agradable con debilidades que, por desgracia, siguen siendo muy comunes por ahí.

Algunas reflexiones y conclusiones de este pentest:

Usar una política de contraseñas fuerte y monitorear diariamente si se publican vulnerabilidades para las aplicaciones/bibliotecas/plugins/firmware que tienes en producción.

utiliza herramientas de código abierto que todavía se mantienen y que están respaldadas por una comunidad fuerte.

Almacene sus claves privadas en un repositorio protegido por una autenticación multifactorial. No las dejes "en línea" si no es necesario y no pienses que nadie las encontrará sólo porque crees que las escondiste bien.

Evita jugar con la configuración de sudo si no estás seguro de lo que haces y de cómo se puede abusar de ello. Echa un vistazo a GTFObins para empezar...

Gracias por leer, Happy hacking and always try harder!