---
title: Tickets and Popcorn please - The Day main(dot)js Became the Key Vault
date: 2025-08-30 08:00:00 -0500
image: 
 path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/ciname/cover.png
 height: 1400
 width: 700
categories: [Writeup, Bug Bounty]
tags: [Writeup, Bug Bounty]
---

# Act I — The Setup

It all started on a lazy evening in April. I wasn’t trying to hack anything major, just poking around a movie ticketing site with DevTools open. As I added a ticket to my cart, something odd caught my eye: a POST request carrying a mysterious parameter named encInfo.

`“Why would a frontend encrypt its own traffic before sending it to its backend?”`

That was the spark. What began as casual curiosity turned into a journey where I ended up pulling strangers’ receipts and breaking AES encryption in the browser.

`“The checkout flow looked ordinary — until I noticed encInfo.”`

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/1.png)


# Act II — The First Discovery: Ghost Receipts

It all began while observing **network activity via Caido HTTP History** . A specific request caught my eye:

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/4.png)

No authentication. No CSRF token. Just a bookingId and an email.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/2.png)

So I replayed it with another valid guessed bookingId , half expecting an error. Instead, I got back a long Base64 string. A quick base64 decode later and... voilà!

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/3.png)

A full movie ticket receipt for a user I had no relationship with that includes the following info: Full Name, Movie Title, Cine, Seat reserver, Date of visit, total price paid.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/6.png)


The invoice retrieval relied entirely on a `bookingId` string — a 7-character alphanumeric identifier starting with `W`. I tried to reverse engineer this string, but was not created in the front, instead in the back, so it was random. Through light fuzzing and guesswork, I retrieved several valid receipts. But I needed scale, with a few lines of Python, I wrote a brute-forcer — and within seconds, my terminal was spitting out dozens of receipts.

![alt text](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/Writeup/cinema/5.png)



