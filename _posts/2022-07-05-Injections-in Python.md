---
title: Vulnerabilities in Python Code
date: 2022-07-05 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/codeflag/ci0.jpg
 height: 3000
 width: 550
categories: [Web Security, Secure Coding]
tags: [web security, coding, python]
---

# OS Command Injection

## Vulnerable Example

The following snippet contains a Flask web application written in Python that executes the `nslookup` command to resolve the host supplied by the user.

```python
@app.route("/dns")
def page():

    hostname = request.values.get(hostname)
    cmd = 'nslookup ' + hostname

    return subprocess.check_output(cmd, shell=True)
```

We can see the `hostname` appended to the command and executed on a subshell with the paratmeter `shell=true`, an attacker could stack another command with `;` in the GET parameter to inject other commands for example `cat /etc/paswd`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/codeflag/ci2.jpg)

## Prevention 
The recommended approach to execute commands is using the `subprocess` API, with the option shell set to `False`.

```python
subprocess.Popen('nslookup ' + hostname, ... , shell=True) # WRONG


subprocess.Popen([ 'nslookup', hostname ], ... , shell=False) # RIGHT 
```

## Lab Solution
Default: 
```python
cmd= 'ping -c 3 %s' %(address)
p=Popen(cmd, shell=True, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)
```

Solution:

```python
cmd= ['ping', '-c', '3', address]
p=Popen(cmd, shell=False, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)
```
# Server Site Template Injection (STTI)

## Vulnerable Example
This snippet contains a `Flask webapp` written in Python, which `concatenates` user input data with a template string.

```python
@app.route("/page")
def page():

    name = request.values.get('name')

    output = Jinja2.from_string('Hello ' + name + '!').render()

    return output
```

The user input data is concatenated to the template text, allowing an attacker to inject template code, for example `\{\{5\*5\}\}` will be rendered as 25.

```bash
$ curl -g 'http://localhost:5000/page?name={{7*7}}'
Hello 49!
```

Depending on the template engine, advanced payloads can be used to escape the template sandbox and gain `RCE` in the system, for example this snippet run a system command that add a malicious script in the tmp folder.

```bash
$ curl -g 'http://localhost:5000/page?name={{''.__class__.mro()[1].__subclasses__()[46]("touch /tmp/malicious.sh",shell=True)}}'
```

## Prevention

### Jinja2

```python
import Jinja2
Jinja2.from_string("Hello {{name}}!").render(name=name)
```

### Mako

```python
from mako.template import Template
Template("Hello ${name}!").render(name=name)
```

### Tornado

{% raw %}

```liquid
template.Template("Hello {{ name }}!").generate(name=name)
```
{% endraw %}

## Lab Solution
Default:

```python 
def page_not_found(e):
	return render_template_string(
	'404 page not found error: the %s resource does not exist.', % request.path), 404
```

Solution:

```python 
def page_not_found(e):
	return render_template_string(
	'404 page not found error: the {{path}} resource does not exist.', path=request.path), 404
```

# Reflected Cross-Site Scripting in MOTD

## Prevention

### Input Validation

-   Exact Match: Only accept values from a finite list of known values.
-   Allow list: If a list of all the possible values can't be created, accept only known good data and reject all unexpected input.
-   Deny list: If an allow-list approach is not feasible (on free form text areas, for example), reject all known bad values.

This chart details a list of critical output encoding methods required to mitigate Cross-Site Scripting

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/codeflag/ci1.jpg)


#### Content Security Policy (CSP)

The Content Security Policy (CSP) is a browser mechanism that enables the creation of source allow lists for client-side resources of web applications, e.g., JavaScript, CSS, images, etc. CSP via a special HTTP header instructs the browser to only execute or render resources from those sources.

For example:

```
Content-Security-Policy: default-src: 'self'; script-src: 'self' static.domain.tld
```

The above CSP will instruct the web browser to load all resources only from the page's origin and JavaScript source code files from `static.domain.tld`. For more details on Content Security Policy, including what it does and how to use it, see [this](https://content-security-policy.com/) article.
notice how the `motd` variable is inserted into the HTML page using the `safe` Jinja filter, which disables HTML escaping of the content and introduces a reflected XSS vulnerability.

```python
#In vanilla Python, this can be escaped by thius html method
html.escape('USER-CONTROLLED-DATA')
```


```python 
# In jinja everything is escaped by default except for values with |safe tag
<li><a href=" \{{ url }}">{{ text }}</a></li>
```

## Lab Solution

Default:

{% raw %}

 ```liquid
 <h2> welcome {{ logged_user }}.</h2>
 !{% if motd %}
 <p>{{motd|safe}}</p>
 {% endif %}
```
{% endraw %}

Solution test 2:

{% raw %}

 ```liquid
 <h2> welcome {{ logged_user }}.</h2>
 !{% if motd %}
 <p>{{motd|e}}</p>
 {% endif %}
```

{% endraw %}