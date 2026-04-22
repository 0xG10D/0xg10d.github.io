---
title: DiceCTF 2026 - Plane-or-Exchange [ Cryptography ]
date: 2026-03-08 17:00:00 +0800
categories: [DiceCTF2026, Cryptography]
tags: [ctf, crypto, dicectf, algebra, writeup]
description: Writeup for the DiceCTF cryptography challenge "plane-or-exchange".
render_with_liquid: false
toc: true
image:
  path: https://ctftime.org/media/cache/8d/dd/8ddda663b7eafd93fcd0271cc10a1214.png
---

This post explains how to solve the **DiceCTF cryptography challenge _plane-or-exchange_**.  
The challenge involves analyzing a custom cryptographic protocol built around permutations and polynomial invariants.

---

## Challenge Description

The challenge description:

> Alice and Bob had a brief exchange, and now they know something which I do not. Would you please help me to drop some Eves?

We are given:

- `protocol.py` (implementation of the protocol)
- `public.txt` containing:
  - Alice's public key
  - Bob's public key
  - Public parameters
  - Ciphertext

Our goal is to **recover the shared secret and decrypt the ciphertext**.

---

## Understanding the Protocol

Inside `protocol.py`, the key functions are:

- `connect()`
- `calculate()`
- `normalize()`

The important transformation is:

```

F(X) = normalize(calculate(X))

```

This converts a grid diagram structure into a **polynomial invariant**.

The key mathematical property is:

```

F(connect(A,B)) = F(A) × F(B)

```

So connecting two objects corresponds to **multiplying their polynomials**.

---

## Key Exchange Mechanism

The protocol builds public keys like this:

```

PubA = connect(P, PrivA)
PubB = connect(P, PrivB)

```

Applying the invariant:

```

F(PubA) = F(P) × F(PrivA)
F(PubB) = F(P) × F(PrivB)

```

The shared secret computed by Alice is:

```

Shared = F(PrivA) × F(PubB)

```

---

## The Cryptographic Weakness

Since we know:

```

F(PubA) = F(P) × F(PrivA)

```

We can isolate the private component:

```

F(PrivA) = F(PubA) / F(P)

```

Now substitute into the shared secret equation:

```

Shared = (F(PubA) / F(P)) × F(PubB)

```

All values on the right side are **public**, so we can compute the shared secret directly.

This means the protocol leaks a **multiplicative invariant**, which completely breaks the key exchange.

---

## Computing the Shared Secret

First compute the polynomials:

### Alice Public Polynomial

```

t^14 - 11t^13 + 61t^12 - 217t^11 + 548t^10 - 1032t^9

* 1494t^8 - 1687t^7 + 1494t^6 - 1032t^5
* 548t^4 - 217t^3 + 61t^2 - 11t + 1

```

### Bob Public Polynomial

```

2t^14 - 19t^13 + 84t^12 - 226t^11 + 405t^10

* 523t^9 + 540t^8 - 527t^7 + 540t^6
* 523t^5 + 405t^4 - 226t^3 + 84t^2 - 19t + 2

```

### Public Parameter Polynomial

```

t^6 - 5t^5 + 13t^4 - 17t^3 + 13t^2 - 5t + 1

```

Recover the private component:

```

PrivA = PubA / P

```

Then compute:

```

Shared = PrivA × PubB

````

---

## Decryption

The shared polynomial is converted into a key by:

1. Converting polynomial to string
2. Hashing with **SHA-256**
3. XOR with the ciphertext

---

## Solve Script

```python
import hashlib
import sympy as sp

t = sp.Symbol("t")

pubA = t**14 - 11*t**13 + 61*t**12 - 217*t**11 + 548*t**10 - 1032*t**9 + 1494*t**8 - 1687*t**7 + 1494*t**6 - 1032*t**5 + 548*t**4 - 217*t**3 + 61*t**2 - 11*t + 1
pubB = 2*t**14 - 19*t**13 + 84*t**12 - 226*t**11 + 405*t**10 - 523*t**9 + 540*t**8 - 527*t**7 + 540*t**6 - 523*t**5 + 405*t**4 - 226*t**3 + 84*t**2 - 19*t + 2
P = t**6 - 5*t**5 + 13*t**4 - 17*t**3 + 13*t**2 - 5*t + 1

ciphertext = bytes.fromhex(
"288cdf5ecf3eb860e2cb6790bff63baceaebb6ed511cd94dd0753bac59962ef0"
"cd171231dc406ac3cdc2ff299d78390ff3"
)

privA, _ = sp.div(pubA, P)

shared = sp.expand(privA * pubB)

key = hashlib.sha256(str(shared).encode()).digest()

while len(key) < len(ciphertext):
    key += hashlib.sha256(key).digest()

plaintext = bytes(c ^ k for c, k in zip(ciphertext, key))
print(plaintext.decode())
````

---

## Final Flag

```
dice{plane_or_planar_my_w0rds_4r3_411_knotted_up}
```

---

## Key Takeaway

This challenge demonstrates a common cryptographic mistake:

> Exposing a **multiplicative invariant** in a key exchange scheme allows attackers to algebraically isolate private components.

Because the protocol allowed:

```
F(connect(A,B)) = F(A) × F(B)
```

an attacker could simply **divide out the public parameter** and recover the secret.

---

> Always ensure cryptographic constructions **do not leak algebraic invariants** that allow separation of secret components.
> {: .prompt-warning }

