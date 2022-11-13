# Secure coding

```php
$pattern = "/^[A-Za-z\s]+$/";

if(preg_match($pattern, $_GET["code"])) {
    $query = "Select * from ports where port_code like '%" . $_REQUEST["code"] . "%'";
    ...SNIP...
}
```

when the query is executed, the `$_REQUEST["code"]` parameters are being used, which may also contain `POST` parameters, `leading to an inconsistency in the use of HTTP Verbs`. In this case, an attacker may use a `POST` request to perform SQL injection, in which case the `GET` parameters would be empty (will not include any bad characters). The request would pass the security filter, which would make the function still vulnerable to SQL Injection.

file upload vulns,

only can be checked by these three:

1. check if validates the content type: image/jpeg -> text/html or image/svg+xml
2. try to change th extension of the image ex: ga.jpg -> ga.php -> ga.jpg.pgp -> ga.jpg .php
3. try to change the header of the file content, if it responds,then its not validating the header of the file and you can inject html inside the jpg file

Try to use upload scanner


Business Logic Vulnerabilities

- Try to change the price to -
- Change the currency from USD to PEN
- Change the units of the product 1 to 0.1
- Add a huge amount items to cart, the total checkout will overflow and became negative for back-end programming language (2,147,483,647)
- if the email for admin users is @wannacry.com and your email is jesus@gmail.com, use  a long string to truncate the email to 255 characteres long-string...........g@wannacry.com.jesus@gmail.com, the verification link will send to jesus@gmail.com
-
# Idor Vulnerabilities
