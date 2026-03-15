---
title: DiceCTF2026 – Good Vibes [ Forensics ]
date: 2026-03-14 21:30:00 +0800
categories: [DiceCTF2026, Forensics, Good Vibes]
tags: [ctf, reverse, pcap, crypto, aes, networking]
description: Full solution and analysis of the DiceCTF "Good Vibes" challenge involving reversing a custom VPN client and decrypting AES-GCM traffic.
toc: true
---

## Challenge Overview

> operating on good vibes has no real consequences... surely...

The challenge provides two files:

```

challenge.pcap
vibepn-client

```

Our goal is to analyze the network traffic and recover the hidden flag.

---

## Step 1 — Inspecting the PCAP

Opening the capture in Wireshark reveals two different phases of communication.

### Phase 1 – Plaintext Chat

The first communication occurs over:

```

TCP/6767

```

Following the TCP stream reveals a conversation between two operators.

```

hi
hi
did anyone tell you you're an idiot
what
did you not get the memo
use the damn vpn#
what's a vpn
how do you not
ok
what
how are you so stupid
whatever
i coded it myself so its super secure
i'm sure it is
i dont want your sarcasm
you got one of us killed before, i dont want another incident.
just like [https://gofile.io/d/l6XqnD](https://gofile.io/d/l6XqnD)
you know the rest
is this malware??
what the
i give up with you
only speak to me over the vpn from now on

```

This conversation tells us two important things:

1. Their previous communications were exposed.
2. They decide to switch to a **custom VPN**.

Immediately after this chat ends, we see new traffic.

---

## Step 2 — Suspicious UDP Traffic

The new traffic appears on:

```

UDP/5959

```

Inspecting the packets reveals a consistent structure.

```

0xbe <type> <length> <sequence> ...

```

From observing the capture we can infer the protocol:

| Type | Meaning        |
| ---- | -------------- |
| 0x01 | HELLO          |
| 0x02 | CHALLENGE      |
| 0x03 | RESPONSE       |
| 0x04 | ESTABLISHED    |
| 0x10 | encrypted data |

This traffic likely originates from the provided binary:

```

vibepn-client

````

To decrypt the tunnel we must reverse this client.

---

## Step 3 — Reversing vibepn-client

Running `strings` on the binary reveals debugging messages:

```bash
strings vibepn-client
````

Important output:

```
[client] Starting handshake...
[client] Sent HELLO
[client] Sent RESPONSE
[client] Session ESTABLISHED! Tunnel is active.
```

The binary also imports OpenSSL encryption functions:

```
EVP_aes_256_gcm
EVP_EncryptInit_ex
EVP_DecryptInit_ex
RAND_bytes
```

So the VPN tunnel uses:

```
AES-256-GCM
```

At first glance this appears secure.

However, reversing the key generation function reveals a major flaw.

---

## Step 4 — Key Derivation Bug

The session key is generated using the following function:

```c
void vpn_derive_session_key(uint8_t key[32]) {
    uint32_t x = time(NULL);

    for (int i = 0; i < 32; i++) {
        x = x * 0x19660D + 0x3C6EF35F;
        key[i] = x & 0xff;
    }
}
```

The key is derived from:

```
time(NULL)
```

This means the key depends only on the **Unix timestamp of the handshake**.

Since PCAP files store packet timestamps, we can recover this value.

---

## Step 5 — Reconstructing the Session Key

From the PCAP we locate the **CHALLENGE packet** (`type = 0x02`).

The packet timestamp gives us the handshake time.

Using that timestamp as the seed we reproduce the key.

```python
def derive_key(seed):
    x = seed
    key = []

    for i in range(32):
        x = (x * 0x19660D + 0x3C6EF35F) & 0xffffffff
        key.append(x & 0xff)

    return bytes(key)
```

This recreates the **AES-256-GCM session key** used by the VPN.

---

## Step 6 — Decrypting VPN Packets

Tunnel packets contain encrypted payloads.

Packet layout:

```
nonce = packet[8:20]
ciphertext = packet[20:-16]
tag = packet[-16:]
AAD = first 20 bytes
```

Using AES-GCM we can decrypt the payload.

```python
from Crypto.Cipher import AES

cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
plaintext = cipher.decrypt_and_verify(ciphertext, tag)
```

Running this against the encrypted packets reveals the tunneled data.

---

## Step 7 — Recovering the Flag

After decrypting the VPN traffic we recover plaintext messages including the flag:

```
dice{y0u_sh0uld_alw@ys_vib3_y0ur_vpns_82bc3}
```

---

## Final Flag

```
dice{y0u_sh0uld_alw@ys_vib3_y0ur_vpns_82bc3}
```

---

## Attack Chain Summary

1. Analyze the PCAP in Wireshark.
2. Identify plaintext chat over TCP.
3. Observe switch to custom VPN traffic.
4. Reverse the `vibepn-client` binary.
5. Identify AES-256-GCM encryption.
6. Discover the key is derived from `time(NULL)`.
7. Reconstruct the key using the packet timestamp.
8. Decrypt VPN traffic and extract the flag.

---

## Key Lesson

> **Never roll your own crypto.**

Although the VPN used strong encryption:

```
AES-256-GCM
```

the security completely failed because the key was generated from a predictable value:

```
time(NULL)
```

With access to the packet capture, an attacker can reconstruct the session key and decrypt all traffic.

This challenge perfectly demonstrates why **custom cryptographic protocols are dangerous**.

```
