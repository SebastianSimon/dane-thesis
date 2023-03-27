## [RFC 6698](https://datatracker.ietf.org/doc/html/rfc6698): The DNS-Based Authentication of Named Entities (DANE) Transport Layer Security (TLS) Protocol: TLSA

<https://datatracker.ietf.org/doc/html/rfc6698#section-3>

> For example, to request a TLSA resource record for an HTTP server running TLS on port 443 at `www.example.com`, `_443._tcp.www.example.com` is used in the request.
> To request a TLSA resource record for an SMTP server running the STARTTLS protocol on port 25 at `mail.example.com`, `_25._tcp.mail.example.com` is used.

<https://datatracker.ietf.org/doc/html/rfc6698#section-4>

> Some specifications for applications that run over TLS, such as [RFC2818] for HTTP, **require that the server’s certificate have a domain name that matches the host name expected by the client**.
> Some specifications, such as [RFC6125], detail how to match the identity given in a PKIX certificate with those expected by the user.

> If a TLSA record has certificate usage 2, the corresponding TLS server SHOULD send the certificate that is referenced just like it currently sends intermediate certificates.

An end entity certificate is:

* <https://wiki.mozilla.org/CA/Terminology>: _“A Certificate that cannot sign other Certificates.”_

Q: “Cannot sign” or “happens to not sign”?
As far as I understand, there is nothing technically preventing a TLS certificate from signing another.

A: …

A trust anchor certificate is:

* <https://wiki.mozilla.org/CA/Terminology>: _“A Certificate that is included in [Network Security Services] with at least one of the trust bits enabled. This is usually a Root Certificate, but under certain circumstances may be an Intermediate Certificate.”_
  Trust bits essentially say that for a specific purpose, for example for TLS, the certificate is a valid EE cert Authentication Chain.

A trust anchor is:

* <https://datatracker.ietf.org/doc/html/rfc4033#section-2>: _“A configured DNSKEY RR or DS RR hash of a DNSKEY RR. A validating security-aware resolver uses this public key or hash as a starting point for building the authentication chain to a signed DNS response. In general, a validating resolver will have to obtain the initial values of its trust anchors via some secure or trusted means outside the DNS protocol. Presence of a trust anchor also implies that the resolver should expect the zone to which the trust anchor points to be signed.”_

### PKIX vs. DANE

Usages 0 and 1 are used together for “PKIX”; usages 2 and 3 for “DANE”.

PKIX is useful for limiting which CA can sign other certs in the certification path, esp. EE certs.
DANE is useful for specifying your own TA under own CA (not known to user’s client; not involving a third-party CA).

> The difference between certificate usage 1 and certificate usage 3 is that certificate usage 1 requires that the certificate pass PKIX validation, but PKIX validation is not tested for certificate usage 3.

### Structure:

https://datatracker.ietf.org/doc/html/rfc6698#section-2 defines TLSA RR.

Certificate Usage: How to match the TLS certificate 
  uint8:
    0, synonyms: “CA constraint”, “PKIX TA”.
    1, synonyms: “service certificate constraint”, “PKIX EE”.
    2, synonyms: “trust anchor assertion”, “DANE TA”.
    3, synonyms: “domain-issued certificate”, “DANE EE”; most useful, most common.

Selector: 
  uint8:
    0 for entire certificate in DER format
    1 for SubjectPublicKeyInfo in DER format

Matching Type: 
  uint8:
    0 for no hashing
    1 for SHA2-256
    2 for SHA2-512

Certificate Association Data:
  Hex byte string

#### Full TLSA RR

```none
IDENTIFIER (PORT PROTOCOL DOMAIN)  CLASS  TYPE  USAGE  SELECTOR  MATCHINGTYPE  CERTIFICATEDATA

_443._tcp.www.bortzmeyer.org.      IN     TLSA  3      1         1             A67924AFCD895B9661C4C5D67A83215F60B7D0E1DA8A30B67EEC6BD623A5B57C
```

#### An example with CNAME

<https://www.rfc-editor.org/rfc/rfc7671.html#section-5.1>

```none
; Multiple Customer Domains hosted by an example.net
; Service Provider:
;
www.example.com.              IN CNAME ex-com.example.net.
www.example.org.              IN CNAME ex-org.example.net.
;
; In the provider's DNS zone, a single certificate and TLSA
; record support multiple Customer Domains, greatly simplifying
; "virtual hosting".
;
ex-com.example.net.           IN A 192.0.2.1
ex-org.example.net.           IN A 192.0.2.1
_443._tcp.ex-com.example.net. IN CNAME tlsa._dane.example.net.
_443._tcp.ex-org.example.net. IN CNAME tlsa._dane.example.net.
tlsa._dane.example.net.       IN TLSA 3 1 1 e3b0c44298fc1c14...
```

### API

<!-- If there is going to be an API, I’d like to have a `class TLSAResourceRecord` with a blocked constructor, which means there’s a synchronous private static field which tells the constructor whether to throw an error or not.
There needs to be a `static async generate` which takes the four TLSA params, sets the flag, and returns a `Promise<TLSAResourceRecord>`. -->

[SubjectPublicKeyInfo import](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/importKey#subjectpublickeyinfo_import) might be helpful.

### Security Considerations

Relies on DNSSEC verification.
<!-- Need to have the same level of authentication used for all DNS additions and changes for a particular domain name.
SSL proxies. -->
If a certificate is revoked, a TLSA record with usage 2 (DANE-TA: Trust Anchor Assertion) regards the revoked cert as a TA; will not check revocation status.

> Because of this, domain administrators need to be responsible for being sure that the keys or certificates used in TLSA records with a certificate usage of 2 are in fact able to be used as reliable trust anchors.

> If an administrator wishes to stop using a TLSA record, the administrator can simply remove it from the DNS.  Normal clients will stop using the TLSA record after the TTL has expired. Replay attacks against the TLSA record are not possible after the expiration date on the RRsig of the TLSA record that was removed.

Key compromise:

> The risk that a given certificate that has a valid signing chain is fake is related to the number of keys that can contribute to the validation of the certificate, the quality of protection each private key receives, the value of each key to an attacker, and the value of falsifying the certificate.

> DNSSEC allows any set of domains to be configured as trust anchors and/or DLVs, but most clients are likely to use the root zone as their only trust anchor.

> A combination of technical protections, process controls, and personnel experience contribute to the quality of security that keys receive.

> DNSKEYs are inherently limited in what they can sign, so a compromise of the DNSKEY for "example.com" provides no avenue of attack against "example.org".
> Even the impact of a compromise of .com's DNSKEY, while considerable, would be limited to .com domains.
> Only the compromise of the root DNSKEY would have the equivalent impact of an unconstrained public CA.

> Public CAs are not typically constrained in what names they can sign, and therefore a compromise of even one CA allows the attacker to generate a certificate for any name in the DNS.
> A domain holder can get a certificate from any willing CA, or even multiple CAs simultaneously, making it impossible for a client to determine whether the certificate it is validating is legitimate or fraudulent.

Key compromise is less disastrous with DNSKEY than with public CAs because DNSKEYs are limited in scope as to what they can sign, whereas CAs usually have a much larger scope.

> Because a TLSA certificate association is constrained to its associated name, protocol, and port, the PKIX certificate is similarly constrained, even if its public CAs signing the certificate (if any) are not.

### Canonicalization, internationalization, etc.

<https://datatracker.ietf.org/doc/html/rfc6698#section-3>

> The base domain name is appended to the result of step 2 to complete the prepared domain name.
> The base domain name is the fully qualified DNS domain name [RFC1035] of the TLS server, with the additional restriction that every label MUST meet the rules of [RFC0952].
> The latter restriction means that, if the query is for an internationalized domain name, it MUST use the A-label form as defined in [RFC5890].

### With RFC 7671, why should matching type 0 be avoided? Summary.

* TLSA RR will almost certainly be too large, causing difficulties for UDP
* Using digest forces TLS server to transmit cert or SPKI during TLS handshake

### Future-proofing: how to make sure that extensions to TLSA can be easily adapted to in the future?

* Usage: Seems to be the most versatile since it already encodes _two_ pieces of information in a single field (2 bits (of 8)) — in basic terms: whether the certificate is used as an end-entity cert or as a trust anchor signing another cert, _and_ whether PKIX verification is required. It’s plausible that extensions to PKIX or entirely new encryption approaches force a rethinking of how authentication needs to be performed. The third bit, fourth bit, and so on, could be used for other decisions — perhaps unforeseeable today. <!-- It’s possible that the other two fields become _reinterpreted_, e.g. if a Usage’s semantics are such that something other than X.509 is used, then the selector field might not necessarily select “SPKI” anymore, but something else. However, at this point, it’s a coin toss as to whether the fields become reinterpreted or if an entirely new DNS record type will be used instead. Actually, it’s extremely unlikely that TLSA will be used for something inherently different. -->
* Selector: unlikely to ever change, but a X.509 parser and DER encoder should be available to a TLSA publisher to extract various fields from a certificate, just in case.
* Matching type: Crypto agility. Maybe common digest algos used in related cryptography protocols can be prepared, even if they’re unlikely to ever be included in the TLSA standard. But digest algos being discovered to be broken is not unprecedented.
* A new fourth field: unlikely to be backward-compatible. It’s more likely that anything _entirely new_ will be “crammed” into the Usage field, i.e. the Usage field being repurposed for new modes of TLSA authentication.

Q: Is there a precedent in which the encoding of some algorithm or data format uses a bit mask, but a specific bit causes some other bits to _completely_ change their semantics depending on whether it is set?

A: Yes: IEEE-754 denormalized floating point numbers is one such case.
