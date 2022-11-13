---
title: Vulnerabilities in Python Code
date: 2022-07-05 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/codeflag/ci0_1.png
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

We can see the `hostname` appended to the command and executed on a subshell with the paratmeter `shell=true`, an attacker could stack another command with `;` in the GET parameter to inject other commands for example `cat /etc/paswd` .

## Prevention 
The recommended approach to execute commands is using the `subprocess` API, with the option shell set to `False`.

### Safe example

```python
cmd= ['ping', '-c', '3', address]
p=Popen(cmd, shell=False, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)

```
@app.route("/dns")
def page():

    hostname = request.values.get(hostname)
    cmd = 'nslookup ' + hostname

    return subprocess.check_output(cmd, shell=True)
```
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

The user input data is concatenated to the template text, allowing an attacker to inject template code, for example {% raw %}`{{5*5}}`{% endraw %} will be rendered as 25.

{% raw %}

```liquid
$ curl -g 'http://localhost:5000/page?name={{7*7}}'
Hello 49!
```
{% endraw %}

Depending on the template engine, advanced payloads can be used to escape the template sandbox and gain `RCE` in the system, for example this snippet run a system command that add a malicious script in the tmp folder.

{% raw %}

```liquid
$ curl -g 'http://localhost:5000/page?name={{''.__class__.mro()[1].__subclasses__()[46]("touch /tmp/malicious.sh",shell=True)}}'
```
{% endraw %}

## Prevention

{% raw %}
```liquid
#Jinja2
import Jinja2
Jinja2.from_string("Hello {{name}}!").render(name=name)
```
{% endraw %}

### Safe example

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



#### Content Security Policy (CSP)

Content Security Policy (CSP) is an added layer of security that helps to detect and mitigate certain types of attacks, including Cross-Site Scripting (XSS) and data injection attacks. CSP via a special HTTP header instructs the browser to only execute or render resources from those sources.

For example:

```
Content-Security-Policy: default-src: 'self'; script-src: 'self' static.domain.tld
```

The above CSP will instruct the web browser to load all resources only from the page's origin and JavaScript source code files from `static.domain.tld`. For more details on Content Security Policy, including what it does and how to use it, see [this](https://content-security-policy.com/) article.
notice how the `motd` variable is inserted into the HTML page using the `safe` Jinja filter, which disables HTML escaping of the content and introduces a reflected XSS vulnerability.

{% raw %}
```liquid
#In vanilla Python, this can be escaped by this html method
html.escape('USER-CONTROLLED-DATA')

# In jinja everything is escaped by default except for values with |safe tag
<li><a href=" \{{ url }}">{{ text }}</a></li>
```
{% endraw %}


### Safe example:

{% raw %}
 ```liquid
 <h2> welcome {{ username }}.</h2>
 !{% if motd %}
 <p>{{motd|e}}</p>
 {% endif %}
```
{% endraw %}

# SQL Injection

## Vulnerable Example

This Flask applicatin checks the user creentials against the SQL database.

```python
@app.route("/login")
def login():

  username = request.values.get('username')
  password = request.values.get('password')

  # Prepare database connection
  db = pymysql.connect("localhost")
  cursor = db.cursor()

  # Execute the vulnerable SQL query concatenating user-provided input.
  cursor.execute("SELECT * FROM users WHERE username = '%s' AND password = '%s'" % (username, password))

  # If the query returns any matching record, consider the current user logged in.
  record = cursor.fetchone()
  if record:
    session['logged_user'] = username

  # disconnect from server
  db.close()
```

This concatenates `username` and `password`, so an attacker could manipulate this to bypass the login mechanism.

Injecting `' OR '1'='1';--` in the username, the query becomes:

```sql
SELECT * FROM users WHERE username = '' OR 'a'='a';-- AND password = '';
```

So this query return any entry in the `users` table thas has an empty username, so the attacker can log in as the first user in the table.

## Prevention

- Scrutinize all the SQL queries that use user-provided input from the HTTP request, such as from sources like request.args.get, request.args.args, and request.args.forms
- User parameterized queries, specifying placeholders for parameters
- Escape inputs before adding them to the query, query concatenation should be avoided


Some python libraries provides the function to use parameterized queries on all type of databases.

### PyMySQL, MySQL-python

```sql
cursor.execute("SELECT * FROM users WHERE username = %s AND password = %s", (username, password))
```

### Safe example


```python
    sql_statement = "SELECT username FROM users WHERE username='%s' and password_hash='%s'", (username, password_hash, )
```

# XML Entity Expansion (XXE)

## Vulnerable Example

This flask snippet pases XML and returns the parsed content in html

```python
@tools.route("/is_xml", methods=['POST'])
def tools_is_xml():
    try:
        # read data from POST
        xml_raw = request.files['xml'].read()

        # create the XML parser
        parser = etree.XMLParser()

        # parse the XML data
        root = etree.fromstring(xml_raw, parser)

        # return a string representation
        xml = etree.tostring(root, pretty_print=True, encoding='unicode')
        return jsonify({'status': 'yes', 'data': xml})
    except Exception as e:
        return jsonify({'status': 'no', 'message': str(e)})
```

When the `etree.fromstring` method is called, it parses and expands with the external entity.

```xml
<!DOCTYPE d [<!ENTITY e SYSTEM "file:///etc/passwd">]><t>&e;</t>
```

In this example the entity `&e;` is expanded with the content of `/etc/passwd` file.

## Prevention

The safest way to prevent XXE is always to disable DTDs (External Entities) completely. 

Depending on the parser, the method should be similar to the following:

```python
parser = etree.XMLParser(resolve_entities=False, no_network=True) 
```

Disabling DTDs (Document Type Definitions) also makes the parser secure against denial of services (DOS) attacks such as Billion Laughs.  

If external entities are necessary then:
- Use XML processor features, if available, to authorize only required protocols (eg: https).
- Use an entity resolver (and optionally an XML Catalog) to resolve only trusted entities.

### Safe example

```python
    # create the XML parser
    parser = etree.XMLParser(resolve_entities=False, no_network=True)
    # parse the XML data
    root = etree.fromstring(xml_raw, parser)
```
