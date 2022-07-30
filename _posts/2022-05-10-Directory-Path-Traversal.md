---
title: Directory Traversal Labs
date: 2022-05-10 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt0.jpg
 height: 1100
 width: 500
categories: [Web Security, Portswigger Academy]
tags: [web security, labs, owasp, lfi]
---

Also known as file path traversal allows to read arbitrary files on the servers. in some cases an attacker might be able to write arbitrary files on the server, allowing them to modify application data or behavior.

# Reading arbitrary files via directory traversal 

We can use the `..` characters to access the parent directory, the following strings are several encoding that can help you bypass a poorly implemented filter.

For example the url takes a filename parameter and returns the content of the file, the aplicaciones appends the requested filename to this base directort and uses an API to read the contents, so the application implements no defenses against directory traversal attacks,so an attacker can request the following URL to retrieve an arbitrary file from the server's filesystem: 

`https://insecure-website.com/loadImage?filename=../../../etc/passwd`

The sequence ../ is valid within a file path, and means to step up one level in the directory structure. The three consecutive ../ sequences step up from /var/www/images/ to the filesystem root, and so the file that is actually read is: 

`/etc/passwd`

Here are some encoded `../` values to bypass some wafs.

``` bash
../
..\

%2e%2e%2f
%252e%252e%252f
%c0%ae%c0%ae%c0%af
%uff0e%uff0e%u2215
%uff0e%uff0e%u2216
```

## File path traversal, simple case

Objective: `To solve the lab, retrieve the contents of the /etc/passwd file.`

First we need to find the potential vector to use this vulnerability, so in the web we see an image and when we access the url, we can see a parameter called `filename` next to the name of the image, we intercept the request of the image with Burpsuite.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt1.jpg)

Then we see that it is making a get request to 22.jpg which is the name of the image.
![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt2.jpg)

We need to change the name to the requested file, but as we don't know how many directories back it is, we go back several times with this payload `../../../../../../etc/paswd` and when we send the request we retrieve the file with users.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt3.jpg)

# Common obstacles to exploiting file path traversal vulnerabilities

## File path traversal, traversal sequences blocked with absolute path bypass

Objective: `To solve the lab, retrieve the contents of the /etc/passwd file.`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt4.jpg)
This lab its the same approach, the only change is that the directory where the image loads from is the root, so we only have to browse /etc/passwd and we will get the list of users.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt5.jpg)

We use the payload `etc/passwd` in the filename value.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt6.jpg)

## File path traversal, traversal sequences stripped non-recursively

In some contexts, such as in a URL path or the filename parameter of a multipart/form-data request, web servers may strip any directory traversal sequences before passing your input to the application. 

You can sometimes bypass this kind of sanitization by URL encoding, or even double URL encoding, the ../ characters, resulting in %2e%2e%2f or %252e%252e%252f respectively.

So using the same approach from previous labs, we'll use the payload: `....//....//....//etc/passwd`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt7.jpg)

## File path traversal, traversal sequences stripped with superfluous URL-decode

The application blocks input containing path traversal sequences. It then performs a URL-decode of the input before using it

So maybe we can encode the payload with URL-encoding and pass to the application, so we´ll use this payload: `..%252f..%252f..%252fetc%252fpasswd`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt8.jpg)

## File path traversal, validation of start of path

The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.

So we´ll supply the path with the initial folders to the backend, for example the image is in this path `/var/www/images/13.jpg` so we´ll use this payload: `var/www/images/../../../../../etc/passwd to bypass this control`.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt9.jpg)

## File path traversal, validation of file extension with null byte bypass

The application validates that the supplied filename ends with the expected file extension.

So this can be bypassed appending a null byte at the end of the filename, so when implement null byte in file name passwd%00.png, it will remove .png extension from checking. By injecting a null byte, the extension rule won’t be enforced because everything after the null byte will be ignored.

So the payload will look like this: `../../../etc/passwd%00`

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/pt10.jpg)

# How to prevent a directory traversal attack

The most effective way to prevent file path traversal vulnerabilities is to avoid passing user-supplied input to filesystem APIs altogether. Many application functions that do this can be rewritten to deliver the same behavior in a safer way.

If it is considered unavoidable to pass user-supplied input to filesystem APIs, then two layers of defense should be used together to prevent attacks:

- The application should validate the user input before processing it. Ideally, the validation should compare against a whitelist of permitted values. If that isn't possible for the required functionality, then the validation should verify that the input contains only permitted content, such as purely alphanumeric characters.
- After validating the supplied input, the application should append the input to the base directory and use a platform filesystem API to canonicalize the path. It should verify that the canonicalized path starts with the expected base directory.

Below is an example of some simple Java code to validate the canonical path of a file based on user input:

``` java
File file = new File(BASE_DIRECTORY, userInput);
if (file.getCanonicalPath().startsWith(BASE_DIRECTORY)) {
    // process file
}
```

Thanks for read, happy hacking.