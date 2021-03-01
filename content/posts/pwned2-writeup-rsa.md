+++
date = "2021-03-01"
title = "pwnED 2 CTF Writeup - scuffed_rsa"
description = "Writeup of the scuffed_rsa challenge in pwnED 2 CTF."
math = true
authors = ["Atte Niemi"]
+++

In this challenge, we are given source code for a bad RSA algorithm. We are also given output (the ciphertext and $n$).
We start by observing a part of the source code:
```
e = 65536
def gen_key():
    p = getPrime(1024)
    q = getPrime(1024)
    n = p # q
    t = (p-1) * (q-1)
    d = inverse(e, t)
    return n, d
```
Two things stand out here. 
We see that $n$ ends up being a prime, which means that the totient function becomes $n-1$.
Secondly, $e$ is not prime (albeit very close to the standard 65537). Since we are given $n$, we can observe that $e$ is not coprime to the totient $n-1$. Normally in RSA, $e$ being coprime to the totient allows for us to uniquely decrypt a message. That is, $ed\equiv 1 \mod \phi(n) \implies m^{ed} \equiv m \mod n$.
We obtain $d$:
```
d = inverse(e, n-1)
```
Since we have $ed \equiv 2 \mod (n-1)$, decrypting the ciphertext normally will leave us with $m^2 \mod n$ instead of the plaintext $m$. Therefore, we need to perform a modular square root to find the two possible plaintexts. In this case, I'm using [crypto-commons](https://github.com/p4-team/crypto-commons/) for their implementation of the Tonelli-Shanks algorithm.
```
m2 = pow(c, d, n)
c1 = modular_sqrt(m2, n) # roots are +-c1 mod (n)
c2 = n - c1 # find -c1 mod(n)
print(long_to_bytes(c1), long_to_bytes(c2))
..., b'pwnEd{t0nell1_sh4nk5_x16_s4v3s_th3_d
4y}'
```
