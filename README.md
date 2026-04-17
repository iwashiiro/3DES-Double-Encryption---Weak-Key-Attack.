# 3DES Double Encryption - Weak Key Attack.

![cryptography](https://img.shields.io/badge/topic-cryptography-red)
![vulnerability](https://img.shields.io/badge/type-vulnerability-orange)
![3DES ECB](https://img.shields.io/badge/cipher-3DES%2FECB-blue)
![Python](https://img.shields.io/badge/language-Python-grey)

---

## Overview.

This repository demonstrates a critical cryptographic weakness that arises when **3DES in ECB mode** is applied twice with the same key, a pattern sometimes seen in legacy systems under the mistaken belief that double-encrypting increases security. By exploiting **DES weak keys**, an attacker can trivially recover or forge plaintexts without any brute-force effort.

---

## What are DES weak keys?

DES has a set of known *weak keys* for which the encryption function becomes its own inverse:

```
E(P, K_weak) = P   =>   encrypt(P) = P   =>   decrypt(P) = P
```

The four DES weak keys are:

```
0x0000000000000000
0xFFFFFFFFFFFFFFFF
0x00000000FFFFFFFF
0xFFFFFFFF00000000
```

Their parity-adjusted variants (`0x0101...01`, `0xFEFE...FE`, etc.) behave identically in practice. When one of these keys is used, every plaintext block encrypts to itself, making DES a no-op.

> **Note:** 3DES accepts keys of 16 or 24 bytes. A 16-byte key is split into K1 (bytes 0-7) and K2 (bytes 8-15). The library rejects keys where K1 == K2, but K1 and K2 can each independently be a DES weak key.

---

## The vulnerability.

Consider a scheme that encrypts a plaintext `P` twice with the same 3DES/ECB key `K`:

```
ciphertext = 3DES_ECB( 3DES_ECB(P, K), K )
```

The intended security assumption is that double encryption doubles the work factor. This is already broken in theory by *Meet-in-the-Middle* attacks, but weak keys make it instantaneously exploitable:

```
if 3DES_ECB(P, K_weak) = P,  then:

  3DES_ECB( 3DES_ECB(P, K_weak), K_weak )
= 3DES_ECB( P, K_weak )
= P
```

So any ciphertext target `C` can be reached by simply sending `P = C` paired with a weak key, with no knowledge of the original key required.

> **Warning:** Double encryption with a symmetric cipher and the same key never provides additional security. It is either equivalent to single encryption, or actively weaker due to known-key attacks.

---

## How the exploit works.

1. Choose a key `K` where K1 and K2 are both DES weak keys (and K1 != K2, to satisfy the library constraint). Example: `K = 0x0101010101010101 + 0xFEFEFEFEFEFEFEFE`.

2. Compute `P = 3DES_dec( 3DES_dec(target, K), K )`. Because of the weak key property, this equals `target` itself.

3. Submit `(P, K)` to the encryption oracle. The oracle computes `3DES(3DES(P, K), K) = target`. Challenge passed.

---

## Running the exploit.

Install the dependencies:

```bash
pip install pycryptodome requests
```

Then run:

```bash
python exploit.py
```

Before running, set `BASE_URL` in the script to point at your target server. The script will automatically find a valid `(plaintext, key)` pair and submit it.

---

## How to fix it.

- Never apply the same symmetric cipher with the same key more than once, as it does not increase security.
- Replace 3DES with **AES-256**, which has no known weak keys and offers a proper 256-bit security margin.
- Avoid ECB mode entirely and use an authenticated mode such as **AES-GCM** or **AES-CBC with HMAC**.
- If legacy constraints force 3DES, validate that neither K1 nor K2 is a known weak or semi-weak DES key before use.
- Reject keys programmatically against the full list of 16 DES weak and semi-weak keys.

---

## References.

- NIST SP 800-57 - Recommendation for Key Management
- Applied Cryptography, Bruce Schneier - §12.4 Weak Keys
- NIST FIPS 46-3 - DES standard (withdrawn)
- RFC 2451 - DES/3DES weak key list
