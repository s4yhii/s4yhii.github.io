---
title: Tickets and Popcorn please - The Day main(dot)js Became the Key Vault
date: 2025-08-20 08:00:00 -0500
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/cover.png
 height: 1400
 width: 700
categories: [Writeup, Bug Bounty]
tags: [Writeup, Bug Bounty]
---

# Act I — The Setup

It all started on a lazy evening in April. I wasn’t trying to hack anything major, just poking around a movie ticketing site with DevTools open. As I added a ticket to my cart, something odd caught my eye: a POST request carrying a mysterious parameter named encInfo.

`“Why would a frontend encrypt its own traffic before sending it to its backend?”`

Pro Tip: In Caido, the first thing I do is filter the HTTP History with HTTPQL to cut out analytics noise and static requests. For example:

```javascript
req.method.cont:"POST" and not req.host.cont:"analytics"
```

That was the spark. What began as casual curiosity turned into a journey where I ended up pulling strangers’ receipts and breaking AES encryption in the browser.

`“The checkout flow looked ordinary — until I noticed encInfo.”`

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/1.png)


# Act II — The First Discovery: Ghost Receipts

It all began while observing **network activity via Caido HTTP History** . A specific request caught my eye:

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/4.png)

At first glance it felt too simple. My instinct was: surely the backend must cross-check this against a logged-in session or some signature. To confirm, I stripped the cookies and replayed the request. It still worked. That’s when I realized this endpoint was completely unauthenticated.

Pro Tip: In Caido I like to replay with headers removed one by one (auth tokens, cookies, referers). This quickly reveals which ones actually matter. In this case, none did.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/2.png)

Next, I wondered how resilient it was against tampering. I changed the bookingId slightly, swapping the last character. Half-expecting a 403 or error, I instead got back a massive Base64 blob in the response.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/3.png)

A full movie ticket receipt for a user I had no relationship with that includes the following info: Full Name, Movie Title, Cine, Seat reserver, Date of visit, total price paid.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/6.png)


The invoice retrieval relied entirely on a `bookingId` string — a 7-character alphanumeric identifier starting with `W`. I tried to reverse engineer this string, but was not created in the front, instead in the back, so it was random. Through light fuzzing and guesswork, I retrieved several valid receipts. But I needed scale, with a few lines of Python, I wrote a brute-forcer — and within seconds, my terminal was spitting out dozens of receipts.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/masterhttps://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/5.png)

Some of the booking IDs I brute-forced returned perfectly valid, usable tickets, while others came back as expired or invalid. If the showtime was scheduled for the same day, the receipt was essentially “live” and could be used to claim entry. Anything older would still return a receipt, but one that no longer held any real-world value.

This meant that, in practice, an attacker could target today’s active IDs and potentially walk into a cinema with someone else’s booking. The combination of weak identifiers, no authentication, and the time-sensitive nature of these receipts turned what looked like “just a privacy leak” into a real access control issue with direct financial and reputational impact.

# Act III — The Cipher in the Browser

Even after pulling receipts, something still bugged me. Every sensitive request — adding tickets, concessions, even returns — had that weird encInfo blob attached. It was like a secret note passed between the frontend and the backend, except the note was just a mess of hex characters.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/8.png)

At first, I tried poking at it. Change a byte, send it back, watch what happens. Every time I did, the server threw me either a 400 Bad Request or a 500 Internal Server Error. That told me one thing: this blob wasn’t just noise. The backend really cared about it.

So I switched gears. If the backend cared so much, maybe the frontend could tell me why. I opened Chrome DevTools, jumped into Sources, and started scrolling through the minified spaghetti that was main.js.

When hunting for crypto in JS, I’ve learned a trick: search for obvious strings like "AES" or "encrypt", or just regex for anything that looks like a key. Thirty-two characters, all numbers? Suspicious. Sixteen characters of lowercase letters? Even more suspicious.

And there it was. Jackpot. Right in the middle of the bundle:

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/7.png)


```javascript
AES.encrypt(pad(JSON.stringify(data), 16), KEY, { iv: IV });
```


No obfuscation. No key rotation. Just the crypto equivalent of leaving your house key under the doormat.

Lesson: **Never trust the client to encrypt or validate anything important.**

At this point, the puzzle pieces clicked together. If I had the key and the IV, then that big scary encInfo blob wasn’t scary at all — it was just encrypted JSON waiting to be freed.

So I copied one out of a real request, fired up a quick Python script with PyCryptodome, and hit run:

```javascript
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
import binascii

KEY = b"22021509147968334581420394558985"
IV  = b"ibfxivitgrpewzgj"

data = binascii.unhexlify("37B0E9B8...")  # sample encInfo
cipher = AES.new(KEY, AES.MODE_CBC, IV)
plaintext = unpad(cipher.decrypt(data), 16)
print(plaintext.decode())
```

And out came a neat little JSON:

```json
{
  "UserSessionId": "995ec229c07cd2adc79289936e12f8fa",
  "CinemaId": "0000000001",
  "Concessions": [
    {"ItemId": "2624", "Quantity": 2, "PriceInCents": 6300}
  ],
  "ReturnOrder": true,
  "FirstRequest": false
}
```

Suddenly, the “mysterious blob” was just plain business logic. Prices. Sessions. Flags for returns and refunds. All editable.

I couldn’t resist testing it. I decrypted one of my own ticket purchase payloads, scrolled down to the Concessions section, and spotted my popcorn order sitting there at 6300 cents.

```json
"Concessions": [
  {"ItemId": "2624", "Quantity": 2, "PriceInCents": 0}
]
```

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/9.png)


Re-encrypted the blob with the same key and IV, dropped it back into the request, and hit Replay.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/10.png)

The backend didn’t blink. No error, no integrity check, no “are you kidding me?” 

This means an attacker could:

- Forge tickets and concession orders.   
- Abuse the order flow (`ReturnOrder`, `ProcessOrderValue`).
- Change prices or claim refunds. 

That’s when it hit me: the browser wasn’t just handling presentation; it was acting like the bank vault for the entire ordering process. And with the AES key and IV lying around in main.js, I hadn’t broken in — they’d handed me the vault combination.

## Summary

The ticketing platform exposed two critical flaws in its web application:

An unauthenticated invoice API that returned Base64-encoded PDF receipts when given only a bookingId and email, allowing enumeration of customers’ tickets.

A frontend that implemented AES-CBC encryption with hardcoded key and IV in main.js, enabling attackers to decrypt, modify, and re-encrypt sensitive request payloads like session IDs, ticket orders, and refunds.

Together, these issues could have allowed attackers to exfiltrate customer PII, enumerate active tickets, and manipulate order flows. Once I confirmed the vulnerability, I stopped testing and reported the issues responsibly.

## Lessons Leardned

- Never trust the client for security. Cryptographic operations and secrets should live on the server, not in JavaScript.

- Use integrity checks. Encrypted blobs must be signed (e.g., HMAC, AEAD) to prevent tampering.

- Protect sensitive APIs with authentication and authorization. A booking receipt is personal data — it should never be accessible unauthenticated.

- Avoid predictable identifiers. Short, sequential booking codes make brute-forcing feasible; use long, random identifiers.

- Validate and rate-limit aggressively. Even “read-only” endpoints can leak massive amounts of data without proper controls.

## Timeline

|Date|Action|
|---|---|
|April, 30, 2025| Initial report sent to the company|
|May, 07, 2025|Initial response sent to the company|w
|June, 30, 2025|Vulnerability fixed, unable to reproduce|
|August 21, 2025|Company give the rights to publish|
|August 30, 2025|Blog post released|

## Thanks

I hope this writeup are useful and help you make money. Thanks for reading, sharing and everything that I don't know. Huge thanks to the following people for helping write and review this post:

- Carlos Aguilar
- Cesar Neira
- Jonathan Huallanca
- Jose Londres
- Eduardo Sarria


