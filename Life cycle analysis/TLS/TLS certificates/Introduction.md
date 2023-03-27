## <https://www.cs.toronto.edu/~arnold/427/19s/427_19S/tool/ssl/notes.pdf>

Basics:

> 1. Alice needs a certificate:
>    1. She chooses a public exponent _e_ and generates a private key
>    2. She generates a public key from that private key
>    3. She generate a certificate signing request ➝ sends that to the CA
>    4. The CA generates a certificate for Alice and signs it ➝ sends it to Alice
> 2. Alice gets Bob’s certificate from him:
>    1. She verifies it using the CA’s certificate (already pre-installed on her computer)
>    2. She extracts Bob’s public key from it
>    3. She attempts to encrypt her large message using Bob’s public key ➝ error!
> 3. Alice picks a symmetric key:
>    1. She picks a strong symmetric key using a pseudo-random number generator
>    2. She encrypts it with Bob’s public key ➝ symkey.enc
>    3. She hashes it and then encrypts it with her private key ➝ signature.bin
>    4. She sends both symkey.enc and signature.bin to Bob
> 4. Bob deciphers and verifies the symmetric key:
>    1. He decrypts symkey.enc using his private key
>    2. He gets and verifies Alice’s certificate and extracts her public key
>    3. He decrypts signature.bin using Alice’s public key
>    4. He compares a hashed symkey with the decrypted signature, they must match
> 5. Alice encrypts her large message with that symmetric key:
>    1. She needs to use choose a symmetric key encryption algorithm
>    2. Bob decrypts using the symmetric key and that same algorithm
>    3. She can also use something like HMAC now for authentication (this won’t be covered in this document but it’s similar to how she creates her signature, and you can read more [here](https://www.jscape.com/blog/what-is-hmac-and-how-does-it-secure-file-transfers))

## [X.509](https://en.wikipedia.org/wiki/X.509)

> An X.509 certificate binds an identity to a public key using a digital signature.

From [RFC 6698](https://datatracker.ietf.org/doc/html/rfc6698#section-1-1):

> TLS uses certificates to bind keys and names.

```none
Certificate {
  identity,
  publicKey,
  sign()
}
```

* CA Certificate
  * can issue other certificates
  * self-signed? Call it Root CA certificate
* end-entity certificate

> X.509 also defines certificate revocation lists, which are a means to distribute information about certificates that have been deemed invalid by a signing authority, as well as a certification path validation algorithm, which allows for certificates to be signed by intermediate CA certificates, which are, in turn, signed by other certificates, eventually reaching a trust anchor.

[There are different formats](https://en.wikipedia.org/wiki/X.509#Certificate_filename_extensions).

### [Certificate chains and cross-certification](https://en.wikipedia.org/wiki/X.509#Certificate_chains_and_cross-certification)

Synonym: “certificate chain” and “certification path”.
Defined in <https://datatracker.ietf.org/doc/html/rfc5280#section-3.2>

`List<Certificate>` starting with EE cert.

From <https://www.cs.toronto.edu/~arnold/427/19s/427_19S/tool/ssl/notes.pdf>:

Signing several certs with _single_ private key means _single_ point of failure.
Therefore: hierarchical structure.

Root CA has private key and root cert.
Signature of root cert is from that private key and root cert itself → **self-signed certificate**.

Intermediate CA has private key and intermediate cert.
Signature of intermediate cert is from _issuer’s (i.e. root’s)_ private key and intermediate cert itself.

End entity has private key and EE cert.
Signature of EE cert is from _issuer’s (i.e. intermediate CA’s)_ private key and EE cert itself.

Root cert and intermediate cert would have TA usage, whereas EE cert would have EE usage.

Signature of EE cert is verified by using intermediate CA’s public key.
Signature of intermediate CA’s is verified by using root CA’s public key.
If at any point a CA is already considered “trusted”, the rest of the chain is trusted.
→ **Chain of trust**.

**Trusted**: usually pre-installed in client _and_ within validity period _and_ not revoked _and_ so on…

### Signature; signing

→ `Public-key cryptography.md`
