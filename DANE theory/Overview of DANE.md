Q: What are all differences between DANE and PKIX usages? Do both allow self-signed certs?

A: PKIX requires name checks. See section 4 of RFC 7671. PKIX requires preconfigured CAs, so no, it does not support self-signed certs. For the DANE usage, self-signed certs are sufficient.

Q: What is it about DANE that allows self-signed certs?

A: …

Q: If you have a DANE-EE TLSA RR in some DNS zone, do you also need a DANE-TA TLSA RR in some other DNS zone?

A: …

<https://en.wikipedia.org/wiki/DNS-based_Authentication_of_Named_Entities#Standards>

> The authentication mechanism that DNSSEC provides is distinct from the authentication mechanism that is provided by DANE, meaning: these two do different jobs. (However, DANE is built on top of DNSSEC and requires DNSSEC to work!)

## [Infoblox: What is DANE?](https://www.infoblox.com/dns-security-resource-center/dns-security-faq/what-is-dane)

One threat comes from the CAs: they can issue certificates for _any_ domain.
Therefore certificates alone are not enough.

> Over the past few years, we have seen malicious and deliberate attempts by attackers or governments to forge certificates for well-known domain names such as google.com or microsoft.com, in an attempt to wiretap or subvert TLS-protected websites.

MITM is another threat.

DANE attempts to mitigate this by trying to match certificate data served over HTTPS with certificate data directly published to the DNS.
This requires DNSSEC as a basis.

> DANE offers the option for clients to seek a second source of verification, in the case of TLSA, certificate information.
> Leveraging the authentication inherently in DNSSEC, organizations can publish the legitimate TLS certificate information in DNS, allowing clients to verify that the certificate information published over HTTPS matches the one published over DNS.

> The DNS server performs a normal DNS lookup for www.example.com TLSA record, uses DNSSEC to validate the response that came from the example.com authoritative name servers

## [RFCs](https://www.mi.fu-berlin.de/inf/groups/ag-idm/theseses/Offene-Master-Arbeiten/index.html)

### [The DNS-Based Authentication of Named Entities (DANE) Protocol: Updates and Operational Guidance](https://www.rfc-editor.org/rfc/rfc7671.html)

This is a brilliant introduction to DANE.

> Used without authentication, TLS provides protection only against eavesdropping through its use of encryption.
> With authentication, TLS also protects the transport against man-in-the-middle (MITM) attacks.

1.1.

Customer Domain: domain name of service that is hosted by a third party, prior to any redirection.

Service Provider: thing hosting service for owner of a Customer Domain; connected to via redirection.

TLSA Publisher: The entity responsible for publishing a TLSA record within a DNS zone (assumed DNSSEC-signed). Belongs to DNS service, whether outsourced or not.

In RFC7671, synonym “public key” ↔ SPKI

Server Name Indication (SNI): extends TLS protocol, identifies service names, making it easier for a TLS request to be responded with cert data specifically associated with one of many Customer Domains at the same IP-address-based TLS service endpoint. → secure virtual hosting

```none
TLSA: Usage Selector MatchingType CertificateData
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^------------------ TLSA Parameters
```

The `www.bortzmeyer.org` of `_443._tcp.www.bortzmeyer.org.` is an example of a TLSA base domain. Modification of RFC7671 over RFC6698: prefer the fully CNAME-expanded name of the TLS server (DNSSEC validation has to be secure at every stage of expansion).

2.

<https://www.rfc-editor.org/rfc/rfc7218#section-2> IANA acronyms

```none
PKIX-TA  PKIX-EE  DANE-TA  DANE-EE
Cert  SPKI
Full  SHA2-256  SHA2-512
```

* SHA2-256: **mandatory** for both client and server
* SHA2-512: **recommended** for client, **not to be exclusively provided** by server

> The digest algorithm agility protocol defined in Section 9 SHOULD be used by clients to decide how to process TLSA RRsets that employ multiple digest algorithms.
> Server operators MUST publish TLSA RRsets that are compatible (see Section 8) with digest algorithm agility (Section 9).

3.

TLS version:

* 1.0: **mandatory** for TLS clients
* 1.2 and later: **recommended** for TLS clients

Server Name Indication (SNI): **mandatory** for TLS clients, **optional** for TLS servers (might respond with default cert chain).
If server’s SNI cert chain doesn’t match client’s SNI extension exactly, server should respond with default or closest match.

4.

Usages 0 and 1: PKIX:

> […] the validation of peer certificate chains requires additional preconfigured CA TAs that are mutually trusted by the operators of the TLS server and client.

Usages 2 and 3: DANE:

> […] no preconfigured CA TAs are required and the published DANE TLSA records are sufficient to verify the peer's certificate chain.

A DANE client **should not** support all 4 usages.
Otherwise: If integrity of DNSSEC compromisable, only server’s TLSA RRSet needs to be replaced _“with one that lists suitable DANE-EE(3) or DANE-TA(2) records, effectively bypassing any added verification via public CAs.”_

> Designs in which clients support just the DANE-TA(2) and DANE-EE(3) certificate usages are RECOMMENDED.
> With DANE-TA(2) and DANE-EE(3), clients don't need to track a large changing list of X.509 TAs in order to successfully authenticate servers whose certificates are issued by a CA that is brand new or not widely trusted.

Q: So what if the inverse happens? Or is this not really a threat?

A: …

> In other words, when all four usages are supported, PKIX-TA(0) and PKIX-EE(1) offer only illusory incremental security over DANE-TA(2) and DANE-EE(3).

Q: Does this mean that a DANE client might start checking TLSA RRs with the PKIX usages, and if they cannot be validated, then the client might continue checking TLSA RRs with DANE usages, but they might pass validation, although not legitimately?

A: …

Servers _may_ offer TLSA RRs with both PKIX and DANE usages, so both types of clients are supported.

Q: Wouldn’t it be better to say that for each usage that is found, at least one TLSA RR must be validated? Wouldn’t the consequence of the recommendation from the RFC be that you have to switch DANE clients for different web services with different usages? If so, wouldn’t it be simpler to just have one DANE client capable of supporting every usage, but it validates records in a “smart” way somehow?

A: …

4.1.
Opportunistic Security (OS) [RFC7435](https://www.rfc-editor.org/rfc/rfc7435) – Some Protection Most of the Time

> This document defines the concept "Opportunistic Security" in the context of communications protocols.
> Protocol designs based on Opportunistic Security use encryption even when authentication is not available, and use authentication when possible, thereby removing barriers to the widespread use of encryption on the Internet.

> When […] the use of authentication is based on the _presence_ of server TLSA records, it is especially important to avoid the PKIX-EE(1) and PKIX-TA(0) certificate usages.

If no _secure_ TLSA records exist, default to unauthenticated TLS.
If no TLS exists, use no encryption.
If _secure_ TLSA records exist, then authentication **must** succeed.

No security advantage in using PKIX usages when DANE usages are also supported.

> Authentication via the PKIX-TA(0) and PKIX-EE(1) certificate usages is more brittle; the client and server need to happen to agree on a mutually trusted CA, but with OS the client is just trying to protect the communication channel at the request of the server and would otherwise be willing to use cleartext or unauthenticated TLS.
> The use of fragile mechanisms (like public CA authentication for some unspecified set of trusted CAs) is not sufficiently reliable for an OS client to honor the server's request for authentication.
> OS needs to be non-intrusive and to require few, if any, workarounds for valid but mismatched peers.
> Because the PKIX-TA(0) and PKIX-EE(1) usages offer no more security and are more prone to failure, they are a poor fit for OS and SHOULD NOT be used in that context.

4.2.
Interaction with Certificate Transparency (CT) [RFC6962](https://www.rfc-editor.org/rfc/rfc6962)

> This document describes an experimental protocol for publicly logging the existence of Transport Layer Security (TLS) certificates as they are issued or observed, in a manner that allows anyone to audit certificate authority (CA) activity and notice the issuance of suspect certificates as well as to audit the certificate logs themselves.
> The intent is that eventually clients would refuse to honor certificates that do not appear in a log, effectively forcing CAs to add all issued certificates to the logs.
> Logs are network services that implement the protocol operations for submissions and queries that are defined in this document.

Force CAs to make all certs available in log, or else no one will trust them.

For DANE usages **no CT checks recommended** for TLS clients.

> Publication of the server certificate or public key (digest) in a TLSA record in a DNSSEC-signed zone by the domain owner assures the TLS client that the certificate is not an unauthorized certificate issued by a rogue CA without the domain owner's consent.

Q: How?

A: …

Q: Does this specifically apply to the DANE-EE usage?

A: …

Case DANE-TA: Cert chaining fails (no known public root CA) → CT cannot apply, because a known root CA is required for CT.
But: DANE-TA is “generally intended to support non-public TAs” anyway, so, again, **no CT checks recommended**.

PKIX usages: same without DANE: the TLSA records “only constrain which CAs are acceptable in PKIX validation”.

4.3.
**Switching** PKIX usages ↔ DANE usages

Switching caused by preference of which set of usages to support.
Transition needs to be gradual, e.g. servers offer both usage types, clients accept both usage types, until most servers have switched, then most clients will disable legacy usage type support.

DANE → PKIX: CA TAs need to be fetched and managed.

This section is very vague.

Q: PKIX → DANE: What needs to happen? Nothing?

A: …

Q: Again, this builds on this idea that a TLS client should only support one set of usages. Why not make it a preference like this?

> Which DANE authentication usage do you want to choose by default?
> * [x] DANE (_recommended_)
> * [ ] PKIX (_not recommended_)

And when clicking the PKIX option:

> Warning! This is currently not recommended because of reason A, B, and C (see RFCs X, Y, and Z). Continue? <kbd>Ok</kbd> or <kbd>Cancel</kbd>.

Additionally, have a list of domains to only use DANE usage with, and a list of domains to only use PKIX usage with. Isn’t that a more user-friendly approach?

A: …

5.

Specific usage guidance

5.1. DANE-EE

New semantics (compared to RFC 6698): _“peer identity matching and validity period enforcement are based solely on the TLSA RRset properties”_

Additional use case: usage 3 and selector 1 (DANE-EE, SPKI) can also be used to _just_ authenticate a public key, not only a full certificate.

Binding (Name ↔ PK) is “based entirely on the TLSA record association”; just check TLS server’s EE cert.
Hence: If reference name doesn’t match any of cert’s names, TLS server **must** be considered authenticated regardless.
Also done for simplicity.
**Except: openssl disagrees with this and performs name checks by default (see `man openssl-s_client`, flag `-dane_ee_no_namechecks`). This willful violation is due to “unknown key share” attacks and cross-origin scripting restrictions.**

<!--
Q: By “association”, does the RFC mean that e.g. the _name_ `www.example.org` has an _associated_ name `_443._tcp.www.example.org.` which corresponds to the TLSA RRSet and only the PK of the TLSA RRs needs to match with the (hashed) SPKI of the TLS record from `www.example.org`? In other words, does it just refer to the fact that all you need to do to authenticate a TLS server is to compute the name where you expect to find TLSA RRs based on the name where you expect to find the TLS cert, and if both are found, that’s all it takes for authentication?

A: TLSA is a TLS certificate _association_, because a TLSA RR is _associated_ with a particular TLS certificate (or public key).
-->

Ignore expiration date of TLS cert!
Validity period of DNSSEC signatures determines validity period of TLSA RRs.

> Validity is reaffirmed on an ongoing basis by continuing to publish the TLSA record and signing the zone in which the record is contained, rather than via dates "set in stone" in the certificate.

Q: RRSIGs have an expiry date, right?

A: …

Q: What does “continuing to sign the zone” mean?

A: …

Q: What does “continuing to publish” mean, exactly?

A: …

> The expiration becomes a reminder to the administrator that it is likely time to rotate the key, but missing the date no longer causes an outage.
> When keys are rotated (for whatever reason), it is important to follow the procedures outlined in Section 8.

Q: If a client supports TLS but does not support DANE at all, will it still perform PKIX-related checks even if the TLSA RR has usage 3? If yes, does this act as a “fallback” for such clients?

A: …

SNI is optional.

> In organizations where it is practical to make coordinated changes in DNS TLSA records before server key rotation, it is generally best to publish end-entity DANE-EE(3) certificate associations in preference to other choices of certificate usage.
> DANE-EE(3) TLSA records support multiple server names without SNI,
> don't suddenly stop working when leaf or intermediate certificates expire,
> and don't fail when a server operator neglects to include all the required issuer certificates in the server certificate chain.

3 1 1 is the recommendation.

* SPKI because it’s _“compatible with raw public keys”_; don’t need to change when cert renewed.
* SHA2-256 because it’s required.

Q: Does the public key change every time a certificate is updated?

A: Different approaches exist; need to find recommendation in this context.

> This TLSA record type easily supports hosting arrangements with a single certificate matching all hosted domains.
> It is also the easiest to implement correctly in the client.

Client can negotiate authentication with raw PKs with 3 1 X.
If no 3 1 X TLSA RR exists, client **should not** negotiate that.

Raw PK negotiation is possible with 3 0 0, but clients aren’t expected to support this — _“requires extra logic on clients”_.

Raw PKs → See RFC 7250.

**DANE-TA**

> DANE-TA(2) records are a valid alternative for sites with many DANE services.
> […] virtual hosting is more complex

_“sufficiently complete certificate chain”_ required

> remember to replace certificates prior to their expiration dates

5.2. Certificate Usage DANE-TA(2)

New requirement: Servers publishing TLSA RRs with this usage **must** include TA cert as part of TLS response, unless all TLSA RRs are 2 0 0.

```none
requireTACertificateFromTLS := tlsaRRSet.some(({ usage }) => usage is DANE-TA(2)) and not tlsaRRSet.every(({ usage, selector, matchingType }) => usage selector matchingType is DANE-TA(2) Cert(0) Full(0))
```

Easy: _“[publish] the issuing authority as a TA for the certificate chains of all relevant services”_.

CNAME Alias example:

```none
; Two servers, each with its own certificate, that share
; a common issuer (TA).
;
www1.example.com.            IN A 192.0.2.1
www2.example.com.            IN A 192.0.2.2
_443._tcp.www1.example.com.  IN CNAME tlsa._dane.example.com.
_443._tcp.www2.example.com.  IN CNAME tlsa._dane.example.com.
tlsa._dane.example.com.      IN TLSA 2 0 1 e3b0c44298fc1c14...
```

* _“simplifies server key rotation”_
* _“server certificates can be updated without making any DNS changes”_

Once CA changes, update TLSA, but this happens much less frequently than a DANE-EE TLSA RR.

Server certs **must** include name matching one of client’s reference identifiers.
If unrelated Customer Domains are hosted, server **should** use SNI.

5.2.1.
Recommended Record Combinations

Matching type Full(0) is **not recommended**.
No need to transmit TA cert, but clients might _“not be able to augment the server certificate chain with the data obtained from DNS”_ (interop issues → section 10.1.1), especially with selector SPKI(1).
Server needs to transmit TA cert anyway.

Usage DANE-TA(2)? → Selector Cert(0) **recommended**, because constraints from TA cert are important.

5.2.2.
Trust Anchor Digests and Server Certificate Chain

<https://www.rfc-editor.org/rfc/rfc5246#section-7.4.2>:

> The sender's certificate MUST come first in the list.
> Each following certificate MUST directly certify the one preceding it.
> Because certificate validation requires that root keys be distributed independently, the self-signed certificate that specifies the root certificate authority MAY be omitted from the chain, under the assumption that the remote end must already possess it in order to validate it in any case.

But DANE-TA doesn’t presume that client has TA cert.

> In fact, client implementations are free to ignore all locally configured TAs when processing usage DANE-TA(2) TLSA records and may rely exclusively on the certificates provided in the server's certificate chain.

If digest is used and server doesn’t offer TA cert in cert chain, TA cert is unavailable to client; no authentication possible.

If TLSA publisher offers usage 2 AND (selector 1 OR matching type (NOT 0)), then server **must** include TA cert in cert chain.
2 0 0 is the _only_ TLSA RR combination _“for which the client MUST be able to verify the server's certificate chain even if the TA certificate appears only in DNS and is absent from the TLS handshake server certificate message”_.

Q: How does this not violate RFC 5246?

A: …

5.2.3. Trust Anchor Public Keys

Basically 2 1 X.

<https://www.rfc-editor.org/rfc/rfc5280#section-6.1.1> defines:

```none
TrustAnchor {
  trustedIssuerName,
  trustedPublicKeyAlgorithm,
  trustesPublicKey,
  trustedPublicKeyParameters?
}

SPKI {
  trustedPublicKeyAlgorithm,
  trustesPublicKey,
  trustedPublicKeyParameters?
}
```

So, basically:

```none
TrustAnchor {
  trustedIssuerName,
  SPKI
}
```

If cert chain from TLS server has a CA cert where SPKI matches that from a TLSA record, client can match that CA as “intended issuer”.
If not, client can check that topmost cert is “signed by” TA’s PK from TLSA.
Clients are not expected to implement this check.
→ Server **cannot rely** on 2 1 0 to be sufficient to authenticate ⟨chains issued by that public key⟩ if the TA cert is missing in the chain sent in the TLS handshake.

If no certs in chain match PK in a TLSA RR _and_ a 2 1 0 RR is available, _then_ it’s **recommended** for clients to check if topmost cert is signed by PK _and_ not expired _and_ rest of chain passes validation _“including leaf certificate name checks”_.
→ If yes, server is authenticated.

5.3.
Certificate Usage PKIX-EE(1)

Same as 3, _“but, in addition, PKIX verification is required. Therefore, name checks, certificate expiration, CT, etc. apply as they would without DANE.”_

5.4.
Certificate Usage PKIX-TA(0)

Updates RFC 6698: new client requirements.
If client trusts intermediate cert, then they _“MUST be prepared to construct longer PKIX chains than would be required for PKIX alone”_.

⟨TLSA 0 X X RR⟩ _matches_ ⟨PKIX-verified trust chains⟩ that _contain_ an ⟨issuer certificate (root OR intermediate)⟩ that _matches_ its ⟨TLSA 0 X X Cert data⟩.

More complex coordination between Customer Domain and the Service Provider in hosting arrangements → **not recommended** when Service Provider is not also TLSA publisher.

Expectation: clients will only accept chains anchored at particular root CA (same as in TLSA).
But clients’ trusted cert stores might have intermediate certs with or without that root CA.

> such a “truncated” chain might not contain the trusted root published in the server’s TLSA record

> If the omitted root is also trusted, the client may erroneously reject the server chain if it fails to determine that the shorter chain it constructed extends to a longer trusted chain that matches the TLSA record.

→ Client **must** _“continue extending chain even after any locally trusted certificate is found”_.

If not TLSA RRs match certs of chain AND trusted cert found is not self-issued, _then_ _“client **must** attempt to build a longer chain in case a certificate closer to the root matches the server’s TLSA record”_.

6.
Service Provider and TLSA Publisher Synchronization

Ideal: TLSA publisher and Service Provider are the same.
If not: difficult coordination between the two.

Publishing TLSA RRs in Service Provider’s zone:

* avoids bilateral coordination of server cert config
* avoids TLSA RRSet management

But even if published in Customer Domain’s zone:

Use CNAME RRs to delegate content of TLSA RRSet to domain operated by Service Provider.

<!--
Q: The example of why publishing TLSA RRs into the Customer Domain’s zone is _“perhaps the client application does not ‘chase’ CNAMEs to the TLSA base domain”_. How does the CNAME delegation still work? What is responsible for this “redirection” if not the client?

A: _“The hosted web service is redirected via a CNAME alias. The associated TLSA RRset is also redirected via a CNAME alias.”_ from the examples. Wait for Section 7.
-->

Only DANE usages work well with TLSA ↔ CNAME management _“across organizational boundaries”_.

With PKIX:

* Alternative 1: Service Provider needs to obtain certs _in the name of_ Customer Domain _from suitable_ public CA (“securely impersonate the customer”).
* Alternative 2: Customer Domain needs to provision PKs and certs at Service Provider’s system.

With DANE-EE:

* _“the Service Provider can publish a single TLSA RRset that matches TLS”_ cert or PK
* Name checks irrelevant → Applicable to all Customer Domains. → _“Certificate usage DANE-EE(3) makes it possible to deploy a single provider certificate for all Customer Domains.”_
* Customer Domain can employ CNAME.

With DANE-TA:

* _“Service Provider operates a private CA”_ → can issue any cert for any Customer Domain.
* Select right cert chain by SNI.

> **Without DANE, such a certificate would not pass trust verification**, but with DANE, the customer’s TLSA RRset that is aliased to the provider’s TLSA RRset can delegate authority to the provider’s CA for the corresponding service.

Redirection via MX, SRV, etc. is the norm for protocols that support it → TLSA records are found at the redirected domains, not at the base domain.
E.g. SMTP: email service hosted by SP, MX RR at CD has hostname that points at SP’s SMTP host.

> When the Customer Domain’s DNS zone is signed, the MX hostnames can be securely used as the base domains for TLSA records that are published and managed by the Service Provider.

If redirection not possible:

* **Either** CD periodically
  * updates its TLSA RRs
  * _and_ provides PKs to SP
* **or** SP periodically
  * generates keys and certs
  * _and_ **_waits_** for TLSA RRs to be published by CDs
  * _then_ deploys the generated keys and certs

→ Section 7

→ RFC 7673 Using TLSA with SRV

> The DNS-Based Authentication of Named Entities (DANE) specification (RFC 6698) describes how to use TLSA resource records secured by DNSSEC (RFC 4033) to associate a server’s connection endpoint with its Transport Layer Security (TLS) certificate (thus enabling administrators of domain names to specify the keys used in that domain’s TLS servers).
> However, application protocols that use SRV records (RFC 2782) to indirectly name the target server connection endpoints for a service domain name cannot apply the rules from RFC 6698.
> Therefore, this document provides guidelines that enable such protocols to locate and use TLSA records.

7.
TLSA Base Domain and CNAMEs

(CNAME vs SRV or MX)
No indirection support via MX, SRV, etc. → redirection via CNAME.
But: CNAME remaps to target domain for **all protocols**.
DANE-TLS clients **should** follow CNAMEs “hop by hop” **while DNSSEC-validating each domain**.
(Synonym: CNAME expansion)

If all CNAMEs secure, _then_ final CNAME target **should** be the TLSA lookup target, _or if none found_ **should** be the _initial_ name.
That base domain from found TLSA RR **must**:

* be HostName for SNI
* _and_, if Usage not 3, be _“primary reference identifier for peer certificate matching”_

How CNAME expansion works depends on the protocol.

8.
TLSA Publisher Requirements

For each { usage, selector, matching type } combo, there **must** be at least one exitsing TLSA RR matching the current TLS server cert chain.
_Past_ or _future_ certs: TLSA RRs should exist for e.g. key rollover.
These requirements are _per TLSA parameter combination_.

→ TLSA record update process

Clients may

* not support all matchingTypes
* not support the { PKIX, DANE } usages
* want to negotiate raw PKs

Raw PKs work for 3 1 X or, optionally, 3 0 0.

Therefore, must ensure that

* every TLSA parameter combination on its own is sufficient to match current cert chain

8.1.
Key Rollover with Fixed TLSA Parameters

Scenario 1:
No new param combos.
Only key changes.

> In this case, TLSA records matching the existing server certificate chain (or raw public keys) are first augmented with corresponding records matching the future keys, at least two Times to Live (TTLs) or longer before the new chain is deployed.

<!--
TLSA RRSet:
  …TLSA RRs
-->

1. _Add new_ TLSA RR to TLSA RRSet for _new_ key with TTL _ttl_.
2. _Wait_ 2× _ttl_ or more to allow for DNS caches to pick up the new record.
3. _Deploy_ new cert chain on Server.
4. _Verify_ that TLS and TLSA lookups work.
5. _Remove_ obsolete TLSA RRs.

Q: Step 2: There are tools to look up several DNS caches worldwide. Do they help here?

A: …

Scenario 2:
New param combos.
1 1 1 → 3 1 1, and some others.

1. _Republish_ the **exact same** cert data with the new combination
2. Optionally, perform a key rollover as above.

```none
; Initial TLSA RRset.
;
_443._tcp.www.example.org. IN TLSA 1 1 1 01d09d19c2139a46...
```

↓

```none
; New TLSA RRset, same key re-published as DANE-EE(3).
;
_443._tcp.www.example.org. IN TLSA 3 1 1 01d09d19c2139a46...
```

8.2.
Switching to DANE-TA(2) from DANE-EE(3)

Scenario 3:
New param combos.
3 X X → 2 X X.

migrate to a new certificate chain and TLSA RR.
New PK is presumed.

> The use of raw public keys rules out the possibility of certificate chain verification.
> Therefore, the transitional TLSA record for the planned DANE-TA(2) certificate chain is a "3 1 1" record that works even when raw public keys are used.

2 X X updated _after_ new chain deployed _and_ after 3 1 1 of original key is _dropped_.
Otherwise this violates current-per-combo requirement.

| TLSA RR                                                        | Step 0 | Step 1 | Step 2 | Step 3 |
|----------------------------------------------------------------|--------|--------|--------|--------|
| `_443._tcp.www.example.org. IN TLSA 3 1 1 01d09d19c2139a46...` | exists | exists | exists | delete |
| `_443._tcp.www.example.org. IN TLSA 3 1 1 7aa7a5359173d05b...` |        | new    | exists | delete |
| `_443._tcp.www.example.org. IN TLSA 2 0 1 c57bce38455d9e3d...` |        |        |        | new    |

Step 1 involves:

* Make new cert chain for DANE-TA cert available.
* Publish TLSA as 3 1 1 with SPKI hash of DANE-TA cert.

Step 2: _Wait_ at least 2× TTL.

Step 3:

* Delete both 3 1 1 TLSA RRs.
* Publish 2 0 1 TLSA RR.

8.3.
Switching to New TLSA Parameters

Scenario 4:
New matching type.

E.g. X Y 1 to X Y 2 → SHOULD publish X Y 2 for _every_ X and Y of X Y 1.

Scenario 5:
Removing matching type.

Remove entirely.

8.4.
TLSA Publisher Requirements: Summary

> server operators updating TLSA records should make one change at a time

* If same params, publish TLSA of new TLS cert before the cert itself
* Wait for TLSA RRSets _“to expire from DNS caches before configuring servers to use the new certificate chain”_
* Remove obsolete TLSA RRs
* Publish TLSA RRSets for all param combos for current and planned cert chains.

Ensures that some TLSA of _current_ TLS cert chain exists at any time.

> As a result, no matter what combinations of usage, selector, and matching type may be supported by a given client, they will be sufficient to authenticate the server.

9.
Digest Algorithm Agility

How to negotiate “strongest” algo?
Note: _Full_ is irrelevant here.

Clients’ job: configure order of Map { MatchingType → Algo }.
But also: **extensible** for completely new algos.
**Recommended**.

_“To make digest algorithm agility possible”_ means _To make the Z in TLSA X Y Z interchangeable_.
Clients **should** use digest algorithm agility.

> Algorithm agility is to be applied after first discarding any unusable or malformed records (unsupported digest algorithm, or incorrect digest length).

This _“first discard incorrect stuff”_ step is a common first step in any procedure.
This validation needs to be built into DANE validators.

After discarding, the only TLSA records to consider for the client are the records with matching type Full, and the records with the _strongest_ digest algo, _per usage and selector_.

10.
General DANE Guidelines

10.1.
DANE DNS Record Size Guidelines

TLSA RR size matters!

10.1.1.
UDP and TCP Considerations

Avoid params that cause TLSA RR to be so big it causes UDP fragmentation.
Not everything can support TCP.
<!-- (Probably out of scope. Let’s stop focusing on UDP and DTLS.) -->

10.1.2.
Packet Size Considerations for TLSA Parameters

X 0 0 should not be published: too big.
Cert rollover: need to publish _two_ of these big boys (one republished, one new).
X 1 0: some keys are very big, so avoid if their digest would be smaller.
X Y ¬0 is **recommended** and forces transmitting full TLS cert during handshake.

10.2.
Certificate Name Check Conventions

> DANE clients MUST send the SNI extension with a HostName value of the base domain of the TLSA RRset.

Usages 0, 1, 2: DANE clients **must** verify if server name is listed in cert’s SAN or CN (Note: CN is deprecated). **Must** use TLSA base domain. Though, with SMTP (e.g.): can use either MX name or destination name. → RFC 7672

Example:

```none
example.com.               IN MX 0 mail.example.com.
mail.example.com.          IN A 192.0.2.1
_25._tcp.mail.example.com. IN TLSA 2 0 1 (
                              E8B54E0B4BAA815B06D3462D65FBC7C0
                              CF556ECCF9F5303EBFBB77D022F834C0 )
```

HostName in the client’s SNI extension **must** be `mail.example.com`.
One of the names in the cert **must** be `mail.example.com` _or_ `example.com`.

10.3.
Design Considerations for Protocols Using DANE

What if authentication fails?

* Usually: don’t use the server
* Report failure to user (“audit mode”) → _potential problem_, but still connect
* In certain contexts, authentication is _mandatory_, in others it’s _opportunistic_ (not using encryption is not recommended, but this already exists in browsers).

There are similar ideas like how a browser communicates DNS over HTTPS or HTTPS-only (or HSTS) to the user.
For example: show an error page with an <kbd>Accept the risk and continue anyway</kbd> button.
In a JS API, maybe throw `class DNSSECValidationError extends Error`, `class DANEAuthenticationError extends Error`, etc.

> Standards for opportunistic DANE TLS specific to a particular application protocol may modify the above requirements.
> The key consideration is whether or not mandating the use of (unauthenticated) TLS even with unusable TLSA records is asking for more security than one can realistically expect.
> If expecting TLS support when unusable TLSA records are published is realistic for the application in question, then the application MUST avoid cleartext.
> If not realistic, then mandating TLS would cause clients (even in the absence of active attacks) to run into problems with various peers that do not interoperate "securely enough".
> That would create strong incentives to just disable Opportunistic Security and stick with cleartext.

If we accept that DANE offers an important advantage, then this is why DANE needs to be **simple** to implement for server admins.
Let’s say, you’re creating a certificate via Let’s Encrypt.
A DANE-publishing service would then (perhaps with opt-in) automatically create the recommended TLSA records, wait for cache invalidation if necessary, then publish the TLS certs, and then act as a DANE client to verify that it works.
This tool can start with the recommended 3 1 1 support.

→ `Conceptual design of a DANE identity manager.md`

11.
Note on DNSSEC Security

<!-- Read this later. -->

DNS zone needs to be re-signed regularly and the validity period shouldn’t be too long.

<!-- 12.
Summary of Updates to RFC 6698

Read this later. -->

13.
Operational Considerations

Key lost or compromised → Need to update ASAP, so keep TTL short enough, or else your availability is in danger.
Same for signature validity period ← MITM.

14.
Security Considerations

Public CA Web PKI not supported by Application protocol → treat as unusable → connect if opportunistic

(Not clear how this is different from the rest of the RFC, but this is not likely to be relevant anyway, since I’m focusing on HTTPS/TLS, which supports everything we know anyway. I think the presence of such records is supposed to _enforce_ the use of encryption since using authentication _implies_ using encryption.)

## `man openssl-verification-options`

(Everything here is quoted from man pages)

> DANE support is documented in `openssl-s_client(1)`, `SSL_CTX_dane_enable(3)`, `SSL_set1_host(3)`, `X509_VERIFY_PARAM_set_flags(3)`, and `X509_check_host(3)`.
> 
> ### `openssl-s_client`
> 
> SSL/TLS client program.
> 
> This command implements a generic SSL/TLS client which connects to a remote host using SSL/TLS. It is a very useful diagnostic tool for SSL servers.
> 
> #### `-dane_tlsa_domain` `domain`
> 
> Enable RFC6698/RFC7671 DANE TLSA authentication and **specify the TLSA base domain which becomes the default SNI hint** and the primary reference identifier for hostname checks. This must be used in combination with at least one instance of the `-dane_tlsa_rrdata` option below.
> 
> When DANE authentication succeeds, the diagnostic output will include the lowest (closest to 0) depth at which a TLSA record authenticated a chain certificate.
> When that TLSA record is a "2 1 0" trust anchor public key that signed (rather than matched) the top-most certificate of the chain, the result is reported as "TA public key verified".  Otherwise, either the TLSA record "matched TA certificate" at a positive depth or else "matched EE certificate" at depth 0.
> 
> #### `-dane_tlsa_rrdata` `rrdata`
> 
> Use one or more times to specify the RRDATA fields of the DANE TLSA RRset associated with the target service.
> The `rrdata` value is specified in "presentation form", that is four whitespace separated fields that specify the usage, selector, matching type and associated data, with the last of these encoded in hexadecimal.
> Optional whitespace is ignored in the associated data field.
> For example:
> 
> ```sh
> openssl s_client -brief -starttls smtp -connect smtp.example.com:25 -dane_tlsa_domain smtp.example.com -dane_tlsa_rrdata "2 1 1 B111DD8A1C2091A89BD4FD60C57F0716CCE50FEEFF8137CDBEE0326E 02CF362B" -dane_tlsa_rrdata "2 1 1 60B87575447DCBA2A36B7D11AC09FB24A9DB406FEE12D2CC90180517 616E8A18"
> ```
> 
> ```none
> Verification: OK
> Verified peername: smtp.example.com
> DANE TLSA 2 1 1 ...ee12d2cc90180517616e8a18 matched TA certificate at depth 1
> ...
> ```
> 
> #### `-dane_ee_no_namechecks`
> 
> This disables server name checks when authenticating via DANE-EE(3) TLSA records.
> For some applications, primarily web browsers, it is not safe to disable name checks due to "unknown key share" attacks, in which a malicious server can convince a client that a connection to a victim server is instead a secure connection to the malicious server.
> The malicious server may then be able to violate cross-origin scripting restrictions.
> Thus, despite the text of RFC7671, name checks are by default enabled for DANE-EE(3) TLSA records, and can be disabled in applications where it is safe to do so.
> In particular, SMTP and XMPP clients should set this option as SRV and MX records already make it possible for a remote domain to redirect client connections to any server of its choice, and in any case SMTP and XMPP clients do not execute scripts downloaded from remote servers.
