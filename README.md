# Applied Cryptography — Hashing, Cracking, and Encryption

This repo covers three related areas of applied cryptography: cracking
weak password hashes (John the Ripper), asymmetric encryption and
digital signatures for email (GPG/Kleopatra), and symmetric file
encryption with integrity verification (AES + hashing).

## Part 1 — Password Hash Cracking (John the Ripper)

### Objective
Crack raw cryptographic hashes (MD5, SHA-256, SHA-512) using John the
Ripper with a real-world password dictionary, demonstrating how hash
algorithm choice alone — without salting — affects practical crackability.

### Environment
- Tool: John the Ripper
- Wordlist: rockyou.txt (~133-139MB, industry-standard leaked password list)

### What I did

**1. Verified the wordlist**
```
ls -lh /usr/share/wordlists/rockyou.txt
```
Confirmed the dictionary was properly decompressed before use.

**2. Cracked a raw MD5 hash**
```
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt claves-rockyou.txt
john --show --format=raw-md5 claves-rockyou.txt
```
**Result:** recovered password `123456` — near-instant, reflecting both
MD5's speed (a weakness for password hashing) and the password's presence
high in the dictionary.

**3. Cracked a raw SHA-256 hash**
```
john --format=raw-sha256 --wordlist=/usr/share/wordlists/rockyou.txt claves-rockyou.txt
john --show --format=raw-sha256 claves-rockyou.txt
```
**Result:** recovered password `password` in under 1 second
(`1g 0:00:00:00 DONE`).

**4. Cracked a raw SHA-512 hash**
```
john --format=raw-sha512 --wordlist=/usr/share/wordlists/rockyou.txt claves-rockyou.txt
john --show --format=raw-sha512 claves-rockyou.txt
```
**Result:** recovered password `iloveyou`, also in under 1 second.

### Key result
All three hashes — despite using progressively stronger/slower hash
algorithms (MD5 → SHA-256 → SHA-512) — were cracked in under a second
each. This demonstrates a critical security lesson: **hash algorithm
strength alone does not protect weak passwords.** None of these hashes
were salted, and all three plaintext passwords were common enough to
appear early in a standard leaked-password dictionary. The bottleneck
wasn't the hash algorithm — it was password predictability.

### Relevance to the SAM/NT hash cracking in `pentest-metasploitable`
This exercise complements the Windows SAM hash extraction and cracking
performed in the [`pentest-metasploitable`](../pentest-metasploitable)
repo — that exercise cracked NT-format hashes extracted from a live
system, while this one demonstrates cracking against raw, algorithm-pure
hashes. Together they show both ends of the same weakness: whether
hashes come from a live compromised system or a standalone leak, weak
passwords remain crackable regardless of the hashing algorithm used,
unless combined with salting and sufficient computational cost (e.g.
bcrypt, scrypt, Argon2).

### Remediation notes
- Never use unsalted MD5/SHA-family hashes for password storage — they're
  designed to be fast, which is the opposite of what password hashing
  needs
- Use purpose-built password hashing functions (bcrypt, scrypt, Argon2)
  that are intentionally slow and support salting
- Password strength matters independently of hash algorithm — dictionary
  attacks succeed against weak passwords regardless of what hashes them

### Files in this repo
- `screenshots/` — John the Ripper execution and `--show` output for
  each hash format (MD5, SHA-256, SHA-512)


## Part 2 — Email Encryption & Digital Signatures (Kleopatra / GPG)

### Objective
Generate an OpenPGP key pair using Kleopatra (Gpg4win), then use it from
a Thunderbird email client to send both an encrypted message and a
digitally signed message — demonstrating the two core guarantees of
asymmetric cryptography: confidentiality (encryption) and authenticity/
integrity (signing).

### Environment
- Tool: Kleopatra (Gpg4win) + Thunderbird
- Key type: OpenPGP, 255-bit EdDSA

### What I did

**1. Generated an OpenPGP key pair**
Used Kleopatra's key generation wizard, providing name and email for the
certificate, selecting key parameters (type, size, validity period), and
setting a passphrase to protect the private key.

**2. Backed up the key pair and exported the public key**
Created a secure backup of the full key pair (private + public), then
exported just the **public** key separately — the public key is what gets
shared with contacts so they can encrypt messages to you and verify your
signature; the private key never leaves your machine.

**3. Sent an encrypted email**
With the recipient's public key imported into Thunderbird, composed a
message with OpenPGP encryption enabled. The message content is
encrypted using the recipient's public key, meaning only their matching
private key can decrypt it — not even the sender can decrypt it after
the fact without a copy of the recipient's private key.

**4. Sent a digitally signed email**
Composed a second message with digital signing enabled instead of
encryption. Signing lets the recipient verify two things: that the
message genuinely came from the claimed sender (authenticity), and that
the content wasn't altered in transit (integrity) — the email client
displayed a verified signature indicator on receipt.

### Key result
Successfully generated a working OpenPGP identity and used it for both
core email security use cases: confidential (encrypted) and
authenticated (signed) communication, confirming correct behavior on
both the sending and receiving side.

### Remediation / improvement notes
- Publish the public key to a keyserver (or distribute via a
  web-of-trust model) so contacts can discover and verify it independently
  rather than relying on manual exchange
- Set a periodic expiration on the certificate to limit exposure if the
  private key is ever compromised

---

## Part 3 — Hashing & Symmetric Encryption (SHA/MD5/HMAC + AES)

### Objective
Generate cryptographic hashes of a file using multiple algorithms
(SHA-1, SHA-256, MD5, HMAC-SHA256), then encrypt and decrypt the same
file using AES at three key lengths (128/192/256-bit), verifying
integrity throughout.

### Environment
- Tool: OpenSSL (Kali Linux)
- Test file: `texto.txt`

### What I did

**1. Generated hashes with multiple algorithms**
```
sha1sum texto.txt
sha256sum texto.txt
md5sum texto.txt
```
Each algorithm produced a fixed-length digest unique to the file's exact
content — any modification to the file, however small, would produce a
completely different hash.

**2. Generated an HMAC (keyed hash)**
```
openssl dgst -sha256 -hmac "miclavesecreta" texto.txt
```
Unlike a plain hash, an HMAC incorporates a shared secret key into the
digest calculation — this adds authentication of the message's origin,
not just integrity. Only someone with the shared key could produce a
matching HMAC.

**3. Encrypted the file with AES at three key lengths**
```
openssl enc -aes-128-cbc -salt -in texto.txt -out texto_aes128.enc -k claveSegura123
openssl enc -aes-192-cbc -salt -in texto.txt -out texto_aes192.enc -k claveSegura123
openssl enc -aes-256-cbc -salt -in texto.txt -out texto_aes256.enc -k claveSegura123
```
Verified the resulting files were no longer readable as plaintext:
```
file texto_aes128.enc texto_aes192.enc texto_aes256.enc
```

**4. Verified decryption integrity**
```
openssl enc -aes-256-cbc -d -salt -in texto_aes256.enc -out texto_descifrado.txt -k claveSegura123
cat texto_descifrado.txt
```
**Result:** decrypted content matched the original file exactly,
confirming correct encryption/decryption round-trip.

### Key result
Successfully generated integrity hashes (SHA-1, SHA-256, MD5), an
authenticated HMAC, and performed full AES encryption/decryption at all
three standard key lengths — validating both the confidentiality
(encryption) and integrity (hashing) sides of applied cryptography on
the same source file.

### Remediation / improvement notes
- **MD5 and SHA-1 are cryptographically broken** for security purposes
  (known collision vulnerabilities) — acceptable for basic
  non-adversarial integrity checks, but SHA-256 or stronger should be
  used wherever tampering resistance matters
- **AES-256 is preferable to AES-128/192** where the added computational
  cost is acceptable, given its larger security margin against
  brute-force attacks

## Files in this repo
- `screenshots/` — organized by part: John the Ripper cracking results,
  GPG key generation and encrypted/signed emails, hash and AES
  encryption/decryption verification
