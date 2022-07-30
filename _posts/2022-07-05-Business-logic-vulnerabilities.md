---
title: Business logic vulnerabilites
date: 2022-06-30 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/bl0.jpg
 height: 1100
 width: 500
categories: [Web Security, Portswigger Academy]
tags: [web security, labs, owasp, injection]
---

OS command injection allows an attacker to execute arbitrary operating system (OS) commands on the server that is running an application, and typically fully compromise the application and all its data.

# OS command injection, simple case

This lab contains an [OS command injection](https://portswigger.net/web-security/os-command-injection) vulnerability in the product stock checker.

The application executes a shell command containing user-supplied product and store IDs, and returns the raw output from the command in its response.

To solve the lab, execute the `whoami` command to determine the name of the current user.

Solution: 

file upload vulns,

only can be checked by these three:

1. check if validates the content type: image/jpeg -> text/html or image/svg+xml
2. try to change th extension of the image ex: ga.jpg -> ga.php -> ga.jpg.pgp -> ga.jpg .php
3. try to change the header of the file content, if it responds,then its not validating the header of the file and you can inject html inside the jpg file

Try to use upload scanner