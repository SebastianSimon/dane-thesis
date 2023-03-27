## [OpenSSL Tutorial](https://www.cs.toronto.edu/~arnold/427/19s/427_19S/tool/ssl/notes.pdf)

> 1. Alice generates ciphertext by encrypting her message with Bob’s public key
> 2. Alice generates a signature by hashing her message and then encrypting it with her private key
> 3. Alice sends both the ciphertext and signature to Bob
> 4. Bob decrypts the ciphertext into the message with his private key
> 5. Bob decrypts the signature using Alice’s public key
> 6. Bob compares the decrypted signature to a hash of the message; if they match then he knows that only Alice could have sent the data
> 
> Note: If she didn’t hash the message, Eve could simply decrypt this signature using Alice’s freely available public key, and thus read the message

## [What is the difference between encrypting and signing in asymmetric encryption?](https://stackoverflow.com/a/454069/4642212)

Message M from Subject A to Subject B.

* Encryption: MA = A.encrypt(M, PubK of B)
* Decryption: M = B.decrypt(MA, PrivK of B)

→ Confidentiality

* Signature: SA = A.sign(M, PrivK of A)
* Verification: B.verify(SA, PubK of A)

→ Authenticity

## [Tech Talk: What is Public Key Infrastructure (PKI)?](https://www.youtube.com/watch?v=0ctat6RBrFo)

Public Key & Private Key – Key pair generated.

→ <https://cryptotools.net/rsagen>

```sh
# 1. Generate Private Key
# openssl genrsa -out private.pem # Deprecated
openssl genpkey -algorithm RSA -out private.pem

# 2. Derive Public Key
openssl rsa -in private.pem -pubout -out public.pem

# 3. Encrypt Data
# openssl rsautl -encrypt -inkey public.pem -pubin -in data.txt -out data.txt.enc # Deprecated

# 4. Decrypt Data
# openssl rsautl -decrypt -inkey private.pem -in data.txt.enc -out data2.txt # Deprecated

# 5. Verify that it works
diff -q data.txt data2.txt
```

Man page of openssl-pkeyutl:

```sh
# 3. Sign Data
openssl pkeyutl -sign -in data.txt -rawin -inkey private.pem -out data.txt.sig

# 4. Verify Data
openssl pkeyutl -verifyrecover -in data.txt.sig -inkey private.pem
openssl pkeyutl -verify -in data.txt -rawin -sigfile data.txt.sig -inkey private.pem

# 5. Verify that it works
diff -q data.txt data2.txt
```

## [Public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography)

> In a digital signature system, a sender can use a private key together with a message to create a signature.

> One important issue is confidence/proof that a particular public key is authentic, i.e. that it is correct and belongs to the person or entity claimed, and has not been tampered with or replaced by some (perhaps malicious) third party.

## Example

```sh
#!/bin/bash

# https://www.cs.toronto.edu/~arnold/427/19s/427_19S/tool/ssl/notes.pdf

mkdir CA
mkdir Alice
mkdir Bob

# Step 1: Make certificates

( cd CA
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
openssl req -x509 -noenc -key private.pem -sha256 -days 1024 -out root.crt )

( cd Alice
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
openssl req -new -key private.pem -out request.csr )

( cd Bob
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
openssl req -new -key private.pem -out request.csr )

openssl x509 -req -in Alice/request.csr -CA CA/root.crt -CAkey CA/private.pem -CAcreateserial -out Alice/certificate.crt -days 512 -sha256
openssl x509 -req -in Bob/request.csr -CA CA/root.crt -CAkey CA/private.pem -CAcreateserial -out Bob/certificate.crt -days 512 -sha256

# Step 2: Verify certificates, get SPKI

( cd Alice
openssl verify -CAfile ../CA/root.crt ../Bob/certificate.crt
openssl x509 -pubkey -in ../Bob/certificate.crt -noout > public-Bob.pem )

( cd Bob
openssl verify -CAfile ../CA/root.crt ../Alice/certificate.crt
openssl x509 -pubkey -in ../Alice/certificate.crt -noout > public-Alice.pem )

# Step 3: Alice picks a symmetric key, encrypts it using Bob’s public key, hashes it and creates a signature using her private key

( cd Alice
openssl rand -base64 -out symmetric.pem 32
openssl pkeyutl -encrypt -in symmetric.pem -pubin -inkey public-Bob.pem -out symmetric.enc
openssl dgst -sha1 -sign private.pem -out signature.bin symmetric.pem )

# Step 4: Bob receives encrypted symmetric key and signature from Alice

( cd Bob
openssl pkeyutl -decrypt -in ../Alice/symmetric.enc -inkey private.pem -out symmetric.pem
openssl dgst -sha1 -verify public-Alice.pem -signature ../Alice/signature.bin symmetric.pem )

# Step 5: Alice encrypts data using the symmetric key, Bob receives and decrypts it

( cd Alice
openssl enc -aes-256-cbc -pbkdf2 -pass file:symmetric.pem -p -md sha256 -in myfile.webp -out ciphertext.bin )

( cd Bob
openssl enc -aes-256-cbc -pbkdf2 -d -pass file:symmetric.pem -p -md sha256 -in ../Alice/ciphertext.bin -out myfile.webp )
```
