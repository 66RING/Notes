---
title: What Is Https
date: 2020-12-01
tags: web, http
mathjax: true
---

## What is Https

Cryptography


### Symmetric encryption

The classic style is: $abc \overset{+3}{\underset{-3}{\longleftrightarrow}} def$.
+3 is **PSK**(pre shared key)

But how to let the receiver know what the PSK is?


### Asymmetric encryption

Which can solve listening problem.

Encryption with public key, decryption with private key. 
- Public key
    - $A_0+public\ key \overset{en}{=} A_1$
- Private key
    - $A_1+private\ key \overset{de}{=} A_0$

Such as, there are two guys:

``` 
A: What is you public key?
B: pk
Than A can encryption with public and than send the infomation to B.
And B can use the his key to get the right infomation.

What if the hacker has the public?
So hacker can fake the infomation
```


#### Private key encryption

Encryption with private key, decryption with public key. 

``` 
A: Here is my public key
B: get
A: message =private encryption=> cipher
B: cipher =public decryption=> message

But if hacker has your public key hacker can get you message too.
hacker: cipher =public decryption=> message.
```


#### USE BOTH

``` 
A: What is your public key?
B: bk
A: Ok. We are going to use Symmetric Encryption. The PSK is:psk.

hacker don't have the private the so he don't know what to do.
```

Actually asymmetric encryption is not safe enough. What **if B IS NOT B**?


#### Modern HTTPs mode

In real life we may choose to trust police station. So it may help that police station tell you that B is B in the following way.

The police station here which we call it CA institution.
- Signature algorithm
    - encryption with CA's private key
- Check algorithm
    - decryption with CA's public key
- the whole process can be received but can't be faked

(we trust browser, browser trust CA) 
| Roles   | browser(user), web, CA                                                                                             |
|---------|--------------------------------------------------------------------------------------------------------------------|
| web     | through a serise of process to get the CA certification.                                                           |
| CA      | signature this web with its private key and broadcast the public key, which we can use it to get infomation of web |
| browser | varify the web's public key with CA's public key and mark web is safe. (B is B)                                    |

After we know B is B, we can encryption message with B's public key.And tell B we are going to use PSK.And than we are safe.

### Identity 

- authentication
    - to prove you is you
- authornization
    - to prove you have the permissions

As we know, a web app can use cookie implementate a identity function.
But a cookie is like a tag that server make on you. And in your next request, browser send send cookie with your request.
Which means **you can choose to send it or not with technical means!**. Such as delete local cookie to fake a unvoted tags.


## Hash

### Database Security(tips)

We need to login, but we don't need to save user's password into database.
We can save this `hash(password)` and compare it.\o/


### Use Hash on Web App

``` sequence-diagrams
cache->browser:
browser->nginx: request(Img)
nginx-->browser: Img & ETag(hash) 200
nginx->server: 
server-->nginx: 
browser->cache: save Img

browser->nginx: request(Img & hash equals?)
nginx->server: hash equals?
server-->nginx: yes
nginx-->browser: not modified 206
browser->cache: search your cache
cache->browser: here
```

