---
title: Broken Authentication
date: 2022-03-15 12:00:00 -0400
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Portswigger/auth1.jpg
 height: 1100
 width: 500
categories: [Web Security, Portswigger Academy]
tags: [web security, theory, owasp, auth]
---

Authentication is the process of verifying the identity of a given user or client. In other words, it involves making sure that they really are who they claim to be, there are three authentication factors:
- Something you **know**, such as password or security question, known as "knowledge factors"
- Something you **have**, a physical object like a mobile phone or security token, known as "possession factors"
- Something you **are**, for example biometrics or patterns of behavior, known as "inherence factors"

**What is the difference between authentication and authorization?**
Authentication is the process of verifying that a user **is who they claim to be**, whereas authorization involves verifying whether a user **is allowed to do something**

# Vulnerabilities in authentication mechanisms
A website consists of several mechanisms where vulnerabilities may occur, some of them are broadly applicable across all of these context, we will look the vulnerabilities in the following areas:
## Vulnerabilities in password based login
### Brute force usernames and passwords
- Its common to see business logins in the format `firstname.lastname@company.com`, however even if there is no obvious pattern, sometimes high privileged accounts are `admin` or `administrator`
- During auditing you should check HTTP responses to see if any email addresses are disclosed, sometimes responses contain emails addresses of high-privileged users like administrators and IT support
- In case where the policy requires users to change their passwords regularly, is common to just make minor. predictable changes, for example `Mypassword1` becomes `Mypassword1?`

### Username enumeration
Username enumeration is when an attacker is able to observe changes in the website's behavior in order to identify if a given username is valid, usually occurs at the login page, when you are attempting to brute-force a login page, you should pay attention to any differences in: 
- **Status codes**: During a brute force, the returned status code is likely to be the same in the majority of the guesses, if a guess returns a different status code, maybe the username was correct, is the best practice to always return the same status code regardless of the outcome.
- **Error messages**: Sometimes the returned error message is different on whether both username and password are incorrect or only the password was incorrect, the best practice is use identical, generic messages in both cases.
- **Response times**: When the requests were handled with a different response times, for example a website might only check whether the password is correct if the username is valid. This extra step might cause a slight increase in the response time. This may be subtle, but an attacker can make this delay more obvious by entering an excessively long password that the website takes noticeably longer to handle.

### Flawed brute-force protection
The two most common ways of preventing brute-force attacks are:
- Locking the account that the remote user is trying to access if they make too many failed login attempts
- Blocking the remote user IP address if they make too many login attempts in succession
- In some cases the counter of the failed attempts resets if the IP owner logs in successfully, so an attacker would simply have to log in to their account every few attempts to prevent this limit, so put your credential in the wordlist and you will bypass this.
- We can send multiple credential per request in json format if the web is vulnerable to brute-force

### Account locking
- If the number of login attempts exceed, responses from the server indicating that the account is locked can help an attacker to enumerate usernames

### User rate limiting
Making too many login requests within a short period of time causes your IP to be blocked, the IP can be unblocked in this cases:
- Automatically after a certain period of time has elapsed
- Manually by an administrator
- Manually by the user after successfully completing a CAPTCHA

As the limit is based in the rate of HTTP requests sent from the user IP address, it sometimes also possible to bypass this defense if you guess multiple passwords with a single request.
### HTTP basic authentication
`Authorization: Basic base64(username:passwrord)`
This is not considered a safe authentication method, it involves repeatedly sending the user credential in every request, that can leads to MITM attacks, HTTP basic authentication is also particularly vulnerable to session-related exploits, notably [CSRF](https://portswigger.net/web-security/csrf), against which it offers no protection on its own.
## Vulnerabilities in multi-factor authentication

### Flawed two-factor verification logic
- Means that after user has completed the login step, the website not adequately verify that the same user is completing the second step
For example:
- The user logins with their normal credentials in the first step
 ```bash
POST /login-steps/first HTTP/1.1 
Host: vulnerable-website.com 
... 
username=carlos&password=qwerty
```
- Then he are assigned with a cookie related to his account before going to the second step
- When submitting the verification code, the request need the account cookie to determine the user who is trying to access
```bash
`POST /login-steps/second 
HTTP/1.1 Host: vulnerable-website.com 
Cookie: account=carlos 
... 
verification-code=123456`
```
- An attacker could use any other username when submitting the verification code
- This is dangerous because an attacker is able to brute-force the verification code as it would allow them to log in to any user based only in the username

### Brute-forcing 2FA verification codes
- Some websites implement the login out the user if they enter certain number of incorrect verification codes, this is ineffective, because it can be automated with multi-step process by using Turbo Intruder plugin.
## Vulnerabilities in other authentication mechanisms

# Preventing attacks on your own authentication mechanisms
## Take care with user credentials
- Do not send any login data over unencrypted connections, although you may have implemented HTTPS for your login requests, make sure to redirect any attempted HTTP request to HTTPS as well
- Audit your website to make sure that no username or email addresses are disclosed either through publicly accessible profiles or reflected in HTTP responses

## Don't count on users for security
- Implement an effective password policy, not the traditional, instead implement a simple password checker, for example the Javascript library `zxcvbn` 
- By only allowing passwords which are rated highly by the password checker, you can enforce the use of secure passwords more effectively than you can with traditional policies

## Prevent username enumeration
- Regardless of whether an attempted username is valid, it is important to use identical, generic error messages, and make sure they are really identical
- Your should always return the same HTTP status code with each login request and, finally make the response time in different scenarios as indistinguishable as possible

## Implement robust brute-force protection
- Implement strict, IP-based user rate limiting, this should involve measures to prevent attacker from manipulating their apparent IP address, ideally you should require to complete a Captcha test with every login attempt
- This is not guaranteed to eliminate the threat, however making the process tedious for the attacker

## Triple-check your verification logic
- Is easy for simple logic flaws to creep into code which, in case of authentication, have the potential to completely compromise your web an users
- Auditing any verification or validation logic thoroughly to eliminate flaws is absolutely key to robust authentication

## Implement proper multi-factor authentication
- SMS-based 2FA is technically verifying two factors, however the potential for abuse through SIM swapping, instead use a dedicated app or device that generates the verification code directly
- Make sure that the logic in your 2FA checks is sound so that it cannot be easily bypassed

