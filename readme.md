# Mini TLS Lab: Understanding Symmetric and Asymmetric Encryption with Docker

## Objective

This lab demonstrates:

1. Symmetric Encryption (AES)
2. Asymmetric Encryption (RSA)
3. How RSA and AES work together in TLS/HTTPS

By the end of this lab, you will understand:

```text
RSA -> Securely exchange a secret key

AES -> Encrypt actual traffic using that secret key
```

---

# Lab Architecture

We will create three containers:

```text
+---------+                      +---------+
|  Alice  | <------------------> |   Bob   |
| Client  |                      | Server  |
+---------+                      +---------+
      \                            /
       \                          /
        \                        /
         \                      /
          \                    /
           \                  /
            \                /
             +------------+
             |    Eve     |
             | Attacker   |
             +------------+
```

Roles:

| Container | Purpose  |
| --------- | -------- |
| Alice     | Client   |
| Bob       | Server   |
| Eve       | Attacker |

---

# Phase 1 - Environment Setup

## Create Docker Network

```bash
docker network create crypto-lab
```

---

## Create Alice

```bash
docker run -dit \
--name alice \
--network crypto-lab \
ubuntu:24.04
```

---

## Create Bob

```bash
docker run -dit \
--name bob \
--network crypto-lab \
ubuntu:24.04
```

---

## Create Eve

```bash
docker run -dit \
--name eve \
--network crypto-lab \
ubuntu:24.04
```

---

# Phase 2 - Install Required Packages

## Alice

```bash
docker exec -it alice bash
```

```bash
apt update
apt install -y openssl iputils-ping net-tools
```

Exit:

```bash
exit
```

---

## Bob

```bash
docker exec -it bob bash
```

```bash
apt update
apt install -y openssl iputils-ping net-tools
```

Exit:

```bash
exit
```

---

## Eve

```bash
docker exec -it eve bash
```

```bash
apt update
apt install -y openssl iputils-ping net-tools
```

Exit:

```bash
exit
```

---

# Phase 3 - Verify Connectivity

Enter Alice:

```bash
docker exec -it alice bash
```

Ping Bob:

```bash
ping bob
```

Expected:

```text
PING bob (172.x.x.x)
64 bytes from bob
```

Stop ping:

```bash
Ctrl+C
```

Exit Alice:

```bash
exit
```

---

# Phase 4 - Symmetric Encryption (AES)

In symmetric encryption:

```text
Encrypt Key = Decrypt Key
```

Both parties must already know the same secret.

---

## Create Message on Alice

```bash
docker exec -it alice bash
```

```bash
echo "Attack at dawn" > secret.txt
```

Verify:

```bash
cat secret.txt
```

Output:

```text
Attack at dawn
```

---

## Encrypt with AES

```bash
openssl enc -aes-256-cbc \
-salt \
-in secret.txt \
-out secret.enc \
-pass pass:SuperSecret123
```

View encrypted file:

```bash
cat secret.enc
```

Output will look like garbage.

---

## Send Encrypted File to Bob

Open a second terminal on host:

```bash
docker cp alice:/secret.enc .
docker cp secret.enc bob:/secret.enc
```

---

## Bob Tries Wrong Password

```bash
docker exec -it bob bash
```

```bash
openssl enc -d -aes-256-cbc \
-in secret.enc \
-out recovered.txt \
-pass pass:WrongPassword
```

Expected:

```text
bad decrypt
```

---

## Bob Uses Correct Password

```bash
openssl enc -d -aes-256-cbc \
-in secret.enc \
-out recovered.txt \
-pass pass:SuperSecret123
```

Verify:

```bash
cat recovered.txt
```

Output:

```text
Attack at dawn
```

---

# What Problem Did We Discover?

AES works.

However:

```text
How does Bob receive the AES password securely?
```

If an attacker sees:

```text
SuperSecret123
```

the encryption becomes useless.

This is called:

```text
The Key Distribution Problem
```

---

# Phase 5 - Asymmetric Encryption (RSA)

RSA solves the key distribution problem.

Bob creates:

```text
Public Key
Private Key
```

Anyone can have the public key.

Only Bob has the private key.

---

# Phase 6 - Bob Generates RSA Keys

Enter Bob:

```bash
docker exec -it bob bash
```

Generate private key:

```bash
openssl genrsa -out private.pem 2048
```

Generate public key:

```bash
openssl rsa \
-in private.pem \
-pubout \
-out public.pem
```

Verify:

```bash
ls
```

Expected:

```text
private.pem
public.pem
```

---

## View Public Key

```bash
cat public.pem
```

Output:

```text
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
```

---

# Phase 7 - Share Public Key

Copy public key to host:

```bash
docker cp bob:/public.pem .
```

Copy to Alice:

```bash
docker cp public.pem alice:/public.pem
```

Copy to Eve:

```bash
docker cp public.pem eve:/public.pem
```

This is safe because:

```text
Public keys are meant to be public.
```

---

# Phase 8 - Alice Encrypts a Message with RSA

Enter Alice:

```bash
docker exec -it alice bash
```

Create message:

```bash
echo "Launch missiles" > message.txt
```

Encrypt:

```bash
openssl pkeyutl \
-encrypt \
-pubin \
-inkey public.pem \
-in message.txt \
-out message.enc
```

---

## Transfer Encrypted Message

On host:

```bash
docker cp alice:/message.enc .
docker cp message.enc bob:/message.enc
docker cp message.enc eve:/message.enc
```

---

# Phase 9 - Eve Attempts Decryption

Enter Eve:

```bash
docker exec -it eve bash
```

Try:

```bash
openssl pkeyutl \
-decrypt \
-inkey public.pem \
-in message.enc \
-out stolen.txt
```

Expected failure.

Reason:

```text
Public Key cannot decrypt.
```

---

# Phase 10 - Bob Decrypts Successfully

Enter Bob:

```bash
docker exec -it bob bash
```

Decrypt:

```bash
openssl pkeyutl \
-decrypt \
-inkey private.pem \
-in message.enc \
-out decrypted.txt
```

Verify:

```bash
cat decrypted.txt
```

Output:

```text
Launch missiles
```

---

# Important Observation

RSA successfully solved:

```text
How do we send secrets securely?
```

But RSA is slow.

Modern HTTPS does NOT encrypt all traffic using RSA.

Instead:

```text
RSA protects an AES key

AES protects actual traffic
```

---

# Phase 11 - Simulating a TLS Handshake

This is essentially how HTTPS works.

---

## Alice Generates AES Session Key

Enter Alice:

```bash
docker exec -it alice bash
```

Generate random AES key:

```bash
openssl rand -hex 32 > aes.key
```

View:

```bash
cat aes.key
```

Example:

```text
8d2abccbc9f20c1b7f...
```

---

# Phase 12 - Encrypt AES Key Using Bob's Public Key

```bash
openssl pkeyutl \
-encrypt \
-pubin \
-inkey public.pem \
-in aes.key \
-out aes.key.enc
```

View ciphertext:

```bash
xxd aes.key.enc | head
```

Output appears random.

---

# Phase 13 - Send Encrypted AES Key

On host:

```bash
docker cp alice:/aes.key.enc .
```

Send to Bob:

```bash
docker cp aes.key.enc bob:/aes.key.enc
```

Send to Eve:

```bash
docker cp aes.key.enc eve:/aes.key.enc
```

---

# Phase 14 - Eve Tries to Read AES Key

Enter Eve:

```bash
docker exec -it eve bash
```

Attempt:

```bash
openssl pkeyutl \
-decrypt \
-inkey public.pem \
-in aes.key.enc \
-out stolen.key
```

Expected:

```text
Failure
```

Reason:

```text
Eve lacks Bob's private key.
```

---

# Phase 15 - Bob Recovers AES Key

Enter Bob:

```bash
docker exec -it bob bash
```

Decrypt:

```bash
openssl pkeyutl \
-decrypt \
-inkey private.pem \
-in aes.key.enc \
-out aes.key
```

Verify:

```bash
cat aes.key
```

Compare with Alice's copy.

They should match exactly.

Now:

```text
Alice knows AES key

Bob knows AES key

Eve does not
```

TLS handshake complete.

---

# Phase 16 - Encrypt Real Traffic Using AES

Return to Alice:

```bash
docker exec -it alice bash
```

Create request:

```bash
echo "GET /login HTTP/1.1" > request.txt
```

Load AES key:

```bash
KEY=$(cat aes.key)
```

Encrypt:

```bash
openssl enc -aes-256-cbc \
-pbkdf2 \
-pass pass:$KEY \
-in request.txt \
-out request.enc
```

---

# Phase 17 - Send Traffic

On host:

```bash
docker cp alice:/request.enc .
```

Send to Bob:

```bash
docker cp request.enc bob:/request.enc
```

Send to Eve:

```bash
docker cp request.enc eve:/request.enc
```

---

# Phase 18 - Eve Tries to Read Traffic

Enter Eve:

```bash
docker exec -it eve bash
```

```bash
openssl enc -d -aes-256-cbc \
-pbkdf2 \
-pass pass:WrongPassword \
-in request.enc
```

Expected:

```text
bad decrypt
```

---

# Phase 19 - Bob Reads Traffic

Enter Bob:

```bash
docker exec -it bob bash
```

Load AES key:

```bash
KEY=$(cat aes.key)
```

Decrypt:

```bash
openssl enc -d -aes-256-cbc \
-pbkdf2 \
-pass pass:$KEY \
-in request.enc
```

Output:

```text
GET /login HTTP/1.1
```

---

# Final Understanding

What happened?

```text
1. Bob generated RSA keys

2. Bob shared Public Key

3. Alice generated AES key

4. Alice encrypted AES key using Public Key

5. Bob decrypted AES key using Private Key

6. Alice and Bob now shared the same AES key

7. AES encrypted all traffic

8. Eve could see traffic but could not decrypt it
```

This is the core concept behind HTTPS/TLS.

```text
RSA -> Secure Key Exchange

AES -> Fast Data Encryption
```

When you visit a website:

```text
Browser
    |
    |---- gets server public key
    |
    |---- generates random AES key
    |
    |---- encrypts AES key with public key
    |
Server decrypts AES key
    |
Both now know AES key
    |
AES encrypts all traffic
```

You have just manually recreated a simplified TLS handshake using Docker and OpenSSL.
