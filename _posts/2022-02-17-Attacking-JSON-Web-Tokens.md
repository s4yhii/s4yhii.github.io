Some acronyms:
- JOSE: 
    - Javascript Object Signing and Encryption
    - The name of the working group
- JWT: JSON Web TOKEN
- JWE: JSON Web Encryption 
- JWS: JSON Web Signature
- JWK: JSON Web Key
- JWA: JSON Web Algorithm

"Encryption gives you confidentiality but signature gives you integrity"

JWT has 3 parts separated by a dot:
- Header (base 64 url encoded without padding(no '/', '+', '='))
  - Contain an algorithm "alg" attribute to tell how the token was signed
  - Support a lot of different algorithms (HS256, HS384, HS512, None, ...)
- Payload (base 64 url encoded without padding no '/', '+', '='))
  - May contain anything
  - Use registered claims
    - "iss": issuer
    - "sub": subject
    - "aud": audience
    - "jti": claim id
    - "exp": expiration time
    - "nbf": not before
    - "iat": issued at
- Signature (base 64 encoded)

The JWT Format: Algorithms

HMAC: All services need to know the secret

Example: One client talks to multiple services (application, microservices), if you use HMAC, if one of the server get compromised and the secret code is compromised you can send tampered token to everyone else and you can perform critical actions with that token.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/streams/jwt.jpg)
  
Asymmetric: you share the private key only to trusted services(login, registration, pass reset)

If one of your lower security system get popped nothing happens because the server doesn't has the private key

