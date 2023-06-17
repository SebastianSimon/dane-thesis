---
documentclass: scrreprt
header-includes:
- \usepackage{tcolorbox}
- \newtcolorbox{myquote}{colback=black!5, colframe=black!30, arc=0mm, boxrule=0mm, leftrule=1mm, fontupper=\itshape}
- \renewenvironment{quote}{\begin{myquote}}{\end{myquote}}
papersize: a4
urlstyle: tt
boxlinks: true
monofont: Fira Code
mainfont: FreeSerif
sansfont: FreeSans
secnumdepth: 1
toc-depth: 1
title: Management of Cryptographic Identities using DNS-based Authentication of Named Entities (DANE)
subtitle: Bachelor thesis
date: Berlin, 2023-04-25
author: |
  Sebastian Simon  
  _Matriculation number:_ 375871  
  Technische Universität Berlin  
  ![](TU%20Berlin%20Logo.pdf){ width=128px }  
  _Department for_ Security in Telecommunications (SecT)  
  _First examiner:_ Prof. Dr. Jean-Pierre Seifert  
  _Second examiner:_ Prof. Dr. Stefan Schmid  
  _Supervisor:_ Nils Wisiol
abstract: |
  **Abstract**  
  The Domain Name System (DNS) provides a service to look up server addresses by name, but this service, by default, lacks authentication and is prone to forgery.
  Various security protocols and extensions exist, but their complexity is high and, thereby, adoption is increasing only slowly.
  
  This thesis focuses on automating and simplifying deployment of DNS-based Authentication of Named Entities (DANE), a set of security protocols which can be used by clients to authenticate a server.
  In particular, we establish a prototype of an identity management application, driven by user-centered design, which helps in DNS zone administration with automatic synchronization between a TLS certificate on the server and a TLSA Resource Record in the DNS.
  This synchronization provides the TLS certificate association which enables clients to cross-check cryptographic data, such as public keys or certificates, from the server itself and the DNS.
  TLS certificates are generally issued by external Certificate Authorities; the purpose of this DANE binding is to be able to avoid trust in them.
  
  The goal of the application is to minimize human error and increase security.
  However, implementing DANE requires careful management; an implementation needs to obey different requirements and support different scenarios.
  We examine the life cycle events of a server identity and review the relevant specifications and standards, all of which help ensure correct management.
  
  **Zusammenfassung**  
  Das _Domain Name System_ (DNS) ist ein Dienst zum Abrufen von Serveradressen anhand eines Namens, aber standardmäßig fehlt in diesem Dienst eine Authentifizierung von Daten und diese sind vor einer Verfälschung nicht geschützt.
  Unterschiedliche Sicherheitsprotokolle und -erweiterungen existieren, die jedoch eine hohe Komplexität haben.
  Daher schreitet deren Einsatz nur langsam voran.
  
  Diese Arbeit befasst sich mit einer automatisierten und vereinfachten Bereitstellung der DNS-basierten Authentifizierung von Benannten Entitäten (_DNS-based Authentication of Named Entities_ – DANE), einer Reihe von Sicherheitsprotokollen die ein Client benutzen kann, um einen Server zu authentifizieren.
  Wir errichten konkret einen Prototypen einer Anwendung für Identitätenmanagement mit einem benutzerzentrierten Design; diese soll bei der DNS-Zonen-Administration in Form von einer automatischen Syn­chro­ni­sie­rung zwischen einem TLS-Zertifikat eines Servers und einem TLSA-Eintrag im DNS helfen.
  Diese Syn­chro­ni­sie­rung stellt die TLS-Zertifikats-Assoziation bereit, mit der ein Client kryptographische Daten  wie zum Beispiel öffentliche Schlüssel oder Zertifikate vom Server selbst und vom DNS abgleichen kann.
  Generell werden TLS-Zertifikate von externen _Certificate Authorities_ ausgestellt; Zweck dieser Ausprägung von DANE ist es, auf das Vertrauen in diese verzichten zu können.
  
  Ziel der Anwendung ist es, menschliche Fehler auf ein Minimum zu beschränken und die Sicherheit zu erhöhen.
  Allerdings erfordert eine Implementierung von DANE sorgfältiges Management; sie muss unterschiedlichen Anforderungen gerecht werden und unterschiedliche Szenarien unterstützen.
  Wir betrachten die Ereignisse im Lebenszyklus einer Serveridentität und begutachten relevante Spezifikationen und Normen, die beim korrekten Management behilflich sind.
bibliography: biblio.bib
---

# Introduction {#sec:introduction}

<!-- Brief history -->
The Internet has come a long way since its initial conception in the late 1960s and its rapidly increasing adoption by the wider public in the following decades.
However, many fundamental protocols—especially in its early days—have been engineered with only few security considerations in mind.
Over time, many different protocols, standards, and practices have been proposed and standardized in order to improve security of different systems, particularly for the Internet.

<!-- Internet security now -->
Nowadays, when a web browser loads to a website, the underlying connection is almost always secure: our data is almost completely safe from eavesdropping and tampering, and will arrive at its intended destination in a timely manner most of the time.
In other words, the security-related properties of _confidentiality_, _integrity_, _availability_, and _authenticity_ hold in most cases today.

<!-- DNS workflow now -->
Typically, when a web browser connects to the Internet _today_, e.g. requesting a resource, it first queries the Domain Name System (DNS), the _“phone book” of the Internet_.
The original purpose of the DNS is to provide a mapping between human-friendly names, such as `www.example.com`, and machine-friendly IP addresses, such as `93.184.216.34`.
IP addresses are stored in a Resource Record (RR) of type `A` (for IPv4) or `AAAA` (for IPv6).
Other RR types are also needed, e.g. the name server (type `NS`) which provides authoritative data on the corresponding administrative portion of the DNS (the _DNS zone_), but the DNS can also store arbitrary information [@rfc882].
RRs are identified by a DNS name; RRs of the same name and type form a Resource Record Set (RRSet).
If supported, the response from the DNS is then validated using the DNS Security Extension protocol (DNSSEC) [@rfc4033].
Once the response is received, the router routes to the obtained IP address, eventually reaching a server.

<!-- TLS workflow now -->
The browser requests the Transport Layer Security (TLS) certificate from a _TLS server_ during the so-called _TLS handshake_.
During this client-initiated process, the TLS version is negotiated, and a stateful connection (e.g. via TCP sockets) between client (i.e. the browser) and server is established, over which encrypted data is sent.
Web pages are then served over the Hypertext Transport Protocol (HTTP), the _secure_ version of which is called HTTPS.

<!-- Internet, common user -->
All of this happens in the background.
The only prominent indicator for the average Web user is a little lock icon next to the “address bar”, which contains a URL starting with `https://`.

<!-- Trust store -->
A TLS certificate needs to be cryptographically _signed_ by some issuer in order to be trusted; this also implies trust in that issuer.
Certificates signed by other certificates form a _certificate chain_ [@x509].
Web browsers come with a _trust store_ pre-installed, which contains a plethora of existing certificates owned by Certificate Authorities (CAs).
These are _root certificates_ and _intermediate certificates_, and are collectively called _trust anchor certificates_.
Trust anchor (TA) certificates have been used to sign nearly every website certificate that the web browser will come across.
These website certificates are also called _end-entity_ (EE) certificates.

<!-- CA issues -->
In this particular Public Key Infrastructure, referred to as “PKIX”,[^pkix] the trust store serves as a “secondary source” that the browser can consult in order to “cross-check” a given certificate chain.
However, these trust anchor certificates are not safe from security vulnerabilities; a compromised trust anchor (or CA) might compromise the security of a TLS connection.
One could consider using one’s own trust anchor certificate or a self-signed end-entity certificate, but currently, such custom certificates are not configured to be trusted by every browser, resulting in the TLS connection being rejected by the browser.

  [^pkix]: The “X” in “PKIX” stands for the corresponding ITU-T standard, “X.509”.

<!-- DANE intro -->
An alternative approach is to place certificates into the DNS, which in turn serves as a secondary source.
This is the idea of DNS-based Authentication of Named Entities (DANE), operating in the realm between DNS and target server.
DANE is used to authenticate a target server by verifying if its cryptographic data matches with the cryptographic data found in the corresponding DNS zone, based on its DNS name.
In the case of TLS, a RR of type `TLSA` is used.
A validating client will query this RR and compare it to the TLS certificate, so a DNS administrator will need to make sure to put a `TLSA` RR with the relevant certificate data in their DNS zone [@rfc6698].

<!-- DANE mgmt issues -->
But this is easier said than done.
Manually publishing a `TLSA` RR requires regularly updating the DNS zone, using tools to convert the certificate into different formats, extract specific information from it, compute digests using hashing algorithms, and keeping track of which RRs to keep and which ones to delete—every step being prone to human error, potentially compromising security.

<!-- DANE mgmt considerations -->
This process can be simplified using automation, but even this requires careful management.
On top of that, there are several requirements for both clients and servers that need to be covered, and different validation scenarios to be supported.
This is what this thesis focuses on.
We establish a prototype of a DANE identity management application that synchronizes TLSA RRs with TLS certificates automatically.
The approach used to ensure correct management and validity is to consider _server identities_ and examine their entire _identity life cycle_, i.e. all states and transitions relevant to DANE RR management.

<!-- Model -->
To illustrate the relationships between the objects involved in DANE management, [figure @fig:model] provides a complete entity-relationship model.
It helps visualize which parts exactly are affected by the proposed identity manager, and helps decide how objects need to be manipulated.
The left half (including “TLSA RR”) is the DNS side, the right half is the Web service side.
The subsequent sections will refer back to this diagram, pointing out specific relationships.
There are five links that can have a cardinality of zero; if and only if all of them are exactly zero in a specific system, it means that this system _does not support_ the TLSA binding of DANE.

<!-- Outline -->
[Section @sec:background] briefly examines the _history and prior art_ on this topic.
[Section @sec:protocols] explores the _protocols and specifications_ and [section @sec:standards] explores the more general _standards_ involved in DANE identity management.
[Section @sec:implementation] explores the _implementation_ aspect of the proposed application.
[Section @sec:conclusion] reviews the efforts of establishing DANE identity management done here and provides some pointers for future work.

::: {.figure #fig:model}
```{.mermaid format=pdf theme=neutral background=transparent}
erDiagram
  "DNS zone" ||--o| "TLSA RRSet": contains
  "DNS zone" ||--|{ "DNS name": "has authority over"
  "DNS name" }|--o| "TLSA RRSet": identifies
  "DNS name" ||--o{ "TLSA RR": identifies
  Identitiy ||--|{ "TLS certificate": has
  "TLS certificate" }|--|| "Public key": has
  Identitiy ||--|{ "Public key": has
  "TLS certificate" ||--o{ "TLSA RR": "corresponds to"
  "Public key" ||--o{ "TLSA RR": "corresponds to"
  "TLSA RR" }|--|| "TLSA RRSet": "belongs to"
  "TLS certificate" }|--|{ "DNS name": references
```

Relationship between entities concerning DNS, TLS certificates, and DANE
:::

# Background {#sec:background}

<!-- Subsection intro -->
DANE is a relatively new set of protocols, but in order to understand its origin and purpose we need to understand its place in history.
This section covers the history and security-related developments in the DNS and in the TLS protocol, including issues in DNSSEC deployment, and vulnerabilities and the proposed mitigations and solutions.
Finally, the state of the art in the employment of the DANE binding “TLSA” will be covered, which rests upon DNS (and DNSSEC) and TLS.

## History of the Internet, DNS, and the emergence of DNSSEC {#sec:background-dnssec}

<!-- Brief history -->
World-wide communication between machines—what eventually became _the Internet_—has a rich history.
Briefly, remote hosts are accessible by an _address_, but also by a _host name_; the mapping between the two used to be managed in centralized databases within different internets until the early 1980s.
Due to the rapidly growing size of internets, the diversity in their operation, and administrative and personnel issues, this quickly became unmanageable, thus a decentralized database needed to be established which delegates administration to separate, hierarchically structured _domains_.
This system is known as the _Domain Name System_, or DNS, today [@rfc882].
See [section @sec:protocols-dns] for a more in-depth overview of DNS.

<!-- DNS cache poisoning -->
Many threats and vulnerabilities have been identified in the DNS [@rfc3833].
One of the major vulnerabilities is known as _DNS cache poisoning_[^dcp].
DNS name servers utilize caches to store query results in order to improve performance.
The exploit happens when an attacker taints the DNS cache by bombarding DNS servers with DNS query _responses_ until one server decides to make a DNS query and inadvertently accepts one of the attacker’s responses instead of the real one.
This problem has historically been mitigated by using randomized transaction IDs as well as randomized destination ports to avoid guessing them easily [@stewart2003dns].

  [^dcp]: _DNS cache poisoning_ is also known as _DNS spoofing_ or _name chaining attack_.

<!-- History DNSSEC -->
But the long-term solution[^lts] to this problem are the DNS Security Extensions (DNSSEC), a protocol proposed in 1997 [@rfc3833].
DNSSEC introduces a public-key infrastructure into the DNS which publishes a public key of the DNS zone, whose private counterpart is used to sign the RRSets within the zone.
Additionally, a digest of the public key is also provided and maintained by the _parent_ zone, which in turn has its own public key and signatures.
This additional information is added to every DNS response.
[Section @sec:protocols-dnssec] explains how DNSSEC works in detail.

  [^lts]: However, the algorithms utilized by DNSSEC would need to be fundamentally changed and readied for _post-quantum cryptography_, i.e. still provide security in the presence of the rapidly evolving field of quantum computing which threatens to “break” even the most advanced cryptographic algorithms commonly used today [@quantum2020].

<!-- Security of DNSSEC, other protocols -->
After nearly a decade of design work, the DNSSEC standard was published in March 2005 [@rfc4033].
In August 2004, while DNSSEC was still in development, RFC 3833 analyzed the identified threats and vulnerabilities and evaluated how the DNSSEC standard would solve these problems [@rfc3833].
The security properties achieved by DNSSEC are data integrity and data origin authentication.
DNSSEC is not concerned with achieving confidentiality.
DNSSEC validation is also not always applicable in securing every part of DNS communication.
[Section @sec:protocols-other] provides an overview of other DNS-related security protocols, which solve these issues.

<!-- Availability in DNSSEC -->
Another security concern that remains with DNSSEC is _(Distributed) Denial of Service_ (DDoS, DoS)[^daa], impacting availability of hosts that handle DNS queries.
In fact, the problem is even exacerbated with DNSSEC due to the larger packet payload.
The availability concern will not be discussed in detail in this thesis, but some recent efforts to mitigate DDoS attacks have been done with Software Defined Networks (SDNs) [@sdn2017; @SaharanGupta2019].

  [^daa]: In the context of DNS, specifically, a variant of this exists, known as a _DNS amplification attack_.

<!-- 2010 DNSSEC launch -->
Starting in December 2009, DNSSEC was on its way to be launched for the root DNS zone.
This was a lengthy process that involved a lot of testing, e.g. the effects of the larger payloads in DNS responses, and was finally completed in July 2010: for the first time, root name servers would serve a signed root zone.
The process involved two lengthy ceremonies of generating public and private key pairs and using them to sign the root zone [@IANADNSSECArchive].

<!-- DNSSEC 2010..2017 -->
But this is not the end of the story.
In the years after the launch, DNSSEC struggled to find adoption; only very few domain registrars offered support for DNSSEC, and not all of them deployed it correctly or fully, even in 2017 [@chung2017; @chung2017blog].
Since correct deployment of DNSSEC is complex and a misconfiguration threatens the availability of the domain, it might explain the reluctant pace of adoption.
Common misconfigurations include the mismatch of keys, digests of keys, and signatures, the lack of one or more required RRs, or the missing renewal of expired keys or signatures [@vanAdrichem2015].

<!-- DNSSEC 2017..2018 -->
The key roll-over of the root zone was another issue that took eight years to plan and complete.
The fact that so many complex steps were involved in the design and rollout of DNSSEC, taking two decades of work, likely was another reason why DNSSEC deployment remained low before 2018.
As Dan York, leader of the _Internet Society’s Open Standards Everywhere_ project, puts it [@constantin2020]:

> Deployment is moving on.
> I think there was a pause between 2015 and 2018, while we waited around for the changing of the root key, where people running the DNS infrastructure kind of wanted to wait and see how the root key rollover would go.
> It completed in 2018 and all things are good, the lights are green, and now we’re seeing in the charts how DNSSEC deployment is going up.

<!-- DNSSEC 2018.. -->
After the first successful key roll-over, DNSSEC adoption has been increasing.
Key roll-overs became a quarterly public event called the “Root KSK Ceremonies” [@IANACeremonies].
In 2019, registrars have already noticeably increased their support for DNSSEC [@roth2019].

<!-- Deployment of DNSSEC today -->
By the end of 2020, the _Internet Corporation for Assigned Names and Numbers_ (ICANN) has announced that _all_ generic top-level domains (_gTLDs_, such as `com`, `org`, `info`, etc.) support DNSSEC [@icannGTLDs].
Today, 142 of the 248 country-code top-level domains (_ccTLDs_, such as `de`, `us`, `ua`, etc.), i.e. 57.3 %, support DNSSEC [@statDNSCCTLDs].
However, not many domains _themselves_ deploy DNSSEC: only an estimated 5.9 % of `.com` domains, 8.4 % of `.edu` domains, and—at least—83.7 % of `.gov` domains fully deploy DNSSEC [@DNSSECDeployment].
Almost a third of the world performs DNSSEC _validation_: 31.7 % worldwide on average, the most in Europe (46.2 %), the least in Asia (27.6 %), with Oceania, North and South America, and Africa falling in between [@DNSSECWorldMap].

## Trust in TLS Certificate Authorities {#sec:background-tls}

<!-- CAs -->
TLS certificates are signed by Certificate Authorities (CAs).
See [section @sec:protocols-tls] for more details on how TLS works.

<!-- CAs bad -->
There are hundreds of root certificates owned by over 100 different CAs [@enwiki1150391606].
Since every root certificate (and intermediate certificate) can issue certificates for any domain, there is a risk of forged certificates being issued for popular domains, either by a malicious CA or by an attacker targeting a CA.
This can result in the encryption provided by TLS to be circumvented [@InfobloxDANE].

<!-- Bad CA examples -->
Quite a few of these attempts have been documented; these are just two examples:

* In 2011, fraudulent certificates have been issued for _Google_ by _DigiNotar_ after being hacked by Irani attackers; shortly afterwards, the CA declared bankruptcy [@enwiki1121079203; @google2011; @facebook2016].
* In 2015, this happened to _Google_ again, this time issued by the Chinese registrar _CNNIC_ via an intermediate certificate owned by _MCS Holdings_; the certificate’s private key might have been deliberately used to perform man-in-the-middle attacks [@google2015].

<!-- Proposals other than DANE -->
A few solutions have been used to recognize and deal with these threats, including _certificate pinning_ (or HTTP Public Key Pinning – HPKP), which is a protocol for accepting only a specific set of certificates or public keys, and _Certificate Transparency_ (CT), which proposes a public append-only ledger for auditing issued certificates.
In 2016, _Facebook_ discovered certificates unexpectedly issued by _Let’s Encrypt_, which were requested by the hosting vendor managing one of _Facebook_’s domains; this was not malicious activity, but CT helped with quickly discovering these certificates as a possible security anomaly [@facebook2016].
HPKP has been deprecated in 2017 due to its high complexity and potential for incorrect usage [@Leyden2017; @Tung2017].

<!-- Revocation is hard -->
Another challenge with certificates is their revocation.
Two protocols have been in common use: the _Certificate Revocation List_ (CRL) and the newer _Online Certificate Status Protocol_ (OCSP), both explained in a bit more detail in [section @sec:protocols-tls-revocation].
Both of these require communication with a third party when issuing a revocation, as well as trust in that third party, and consultation of it when verifying the validity of a certificate.
This means, there are privacy concerns and operational difficulties.

## Deployment of DANE {#sec:background-dane}

<!-- DANE, resting upon DNSSEC and TLS -->
Given the background in DNS and TLS and their vulnerabilities, the need for DANE as an alternative to Certificate Authorities is evident.
See [section @sec:protocols-tlsa] for a detailed look into DANE (TLSA).
DANE depends on correct DNSSEC deployment, but DNSSEC still has low usage and high complexity.
DANE introduces _even higher_ complexity and its usage is expected to be _even lower_.

<!-- DANE deployment status -->
The current deployment status of DANE remains low.
In 2017, a handful of websites and mail servers with correct deployment of a `TLSA` RR have been listed as known examples [@isocDANE].
DANE appears to be more popular in email transmission using SMTP with TLS, yet only a handful of email service providers support DANE correctly; the quantity of correct `TLSA` RR deployment also remains low, and is increasing only slowly [@lee2020longitudinal; @constantin2020].
In 2020, just over 5,000 mail exchange servers have had DANE RRs deployed.
By 2023, the number grew to over 9,000.
A total of approximately 3,76 mil. DANE-protected SMTP domains have been recorded [@DNSSECTools].

<!-- Common mistakes -->
A set of common mistakes has been identified when deploying a `TLSA` RR, most of which come down to nonconformance with the relevant specifications [@sys4DANE].
Since DANE depends on DNSSEC, some of these are similar to the common misconfigurations of DNSSEC—e.g. only partial implementation—and some of them are _caused_ by them.
Lack of (reliable) automation and correct synchronization between TLS certificate and TLSA entry is a major issue.
Another interesting point is the misguided deployment of DANE simply for the sake of using a _cool, new protocol_.
DANE is not a one-and-done setup, but requires continual maintenance in order to ensure availability and integrity.
Additionally, proper risk assessment and prioritization are important steps in securing any system [@PwnDefendDNSSEC].

<!-- DANE management implementation -->
As for the implementation efforts of DANE in DNS management systems, whereas some do support the `TLSA` RR in a DNS zone, _automatically managing_ them still comes down to the individual DNS administrator.
deSEC refers to third-party tools to create `TLSA` RRs, but is _“planning to provide a tool that is connected directly to [their] API in the future”_ [@deSEC]; the corresponding GitHub repository includes designs for a DANE identity manager in an early stage of development [@deSECIdM].
Other popular domain registrars and Web hosting services, such as GoDaddy [@GoDaddy], netcup [@netcup], cPanel [@cPanel], or CloudFlare [@CloudFlare], either do not have any plans on automatically supporting DANE whatsoever, or it is not a near-future priority on their road map, currently.

<!-- DANE tools -->
A few online tools exist to generate DANE RRs [@huque], as well as validators that test for correct configuration of `TLSA` RRs and display some debugging output [@sys4Validator; @sidnValidator].
Software tools for these purposes exist, too [@huqueGit; @hashSlinger].
There is, however, a lack of _integration_ of the services that these tools provide into applications that people use daily.

<!-- DANE validation implementations -->
The DANE binding for TLS certificates for HTTP ultimately requires DANE validation in the browser.
Some browser vendors have made some initial efforts in testing DANE validation but have hit certain roadblocks, e.g. high complexity, low performance, or `TLSA` lookups being blocked by the system [@lee2020longitudinal; @langleyNotDANE; @constantin2020].
Unfortunately, major browser vendors do not consider the DANE TLSA protocol to be of high priority, currently; it is unlikely that we will see built-in validation any time soon.

<!-- DANE in Mozilla Firefox -->
_Mozilla Firefox_ has long-standing _Bugzilla_ entries with requests to support DANE natively in the browser [@bugzilla672600; @bugzilla1077323].
These _bugs_ acknowledge the chicken-and-egg problem of DANE: not enough Internet services deploy DANE—and DNSSEC for that matter—for implementers of client validators to care, and not enough client validation implementations exist for Internet services maintainers to care.
They also chronicle the political environment surrounding the _CAs vs. DANE_ debate.
For example, starting 2014, a lot of effort has been put into _Let’s Encrypt_, a free CA.
_Mozilla_ is involved in the _Internet Security Research Group_ (ISRG) responsible for facilitating the infrastructure of _Let’s Encrypt_ [@le2014].
As of now, no work is actively being done to support DANE validation natively in _Mozilla Firefox_.
However, a browser extension exists that promises to provide a built-in DANE validator using _DNS over HTTPS_ [@defkevFxExtension].

<!-- DANE in Google Chrome -->
The _Chromium_ browser (and, by extension, _Google Chrome_) tracks its own _issues_ [@chromium1339991; @chromium50874].
Other security protocols and approaches were more relevant at the time these issues were filed [@gruener2019].
As of today, there are still no plans to support DANE on their road map.

<!-- DANE bad? -->
Several arguments against DANE and DNSSEC have been presented, which focus on the flaws of DNSSEC itself, but also argue that DANE would not solve the problems of existing CAs, and instead would simply introduce _yet another_ “CA”, only that it would be _government-controlled_.
It is pointed out that DANE would make things worse because suspicious activity at a trust anchor would no longer affect the trust in a CA company, but in an entire DNS zone, e.g. for the top-level domain `com` [@Ptacek2015; @Ptacek2015FAQ].
Complete DANE validation is computationally expensive and introduces a larger network load, thus increasing latency, which is cited as another reason why browsers have been avoiding its implementation [@lee2020longitudinal; @langleyNotDANE].

<!-- Summary -->
In summary, the current state of DANE is enveloped in a very long history of the life and death of cryptographic algorithms and protocols.
The state of cryptography as a whole is ever-changing, and DANE is only one of many ideas being proposed, none of which are perfect.
Only very few people see a priority in adopting DANE, so it is worth a try to improve the situation by taking steps towards simplifying the process—be it only for the motivation of more open-ended discussion about generally improving the trust model within communication systems.

# Protocols involved in DANE identity management {#sec:protocols}

This section summarizes the protocol specifications for TLS, DNS, DNSSEC, and DANE TLSA.
It also mentions the other DANE bindings: IPSECKEY, SSHFP, OPENPGPKEY, SMIMEA.
This provides the theoretical background for a technical implementation of DANE.
Finally, some related protocols are briefly summarized and their relationships visualized.

## Transport Layer Security (TLS) certificates {#sec:protocols-tls}

<!-- TLS definition and contents -->
TLS _public-key certificates_ are standardized by ITU-T X.509 since 1988 [@x509].
This subsection introduces the TLS protocol where it is relevant in its involvement as a DANE binding.
The _Transport Layer Security_ protocol—also known by its former name _Secure Sockets Layer_ (SSL)—is the public-key infrastructure prevalent on the Web.
By definition, a _public-key certificate_ binds a _public key_ to an _identity_ [@enwiki1149564995; @rfc6698] (or simply a _name_ [@rfc6698] or _identifier_ [@rfc4949]).
[Section @sec:standards-terminology] covers the terminology around the word “identity”.
According to the standard, a TLS certificate includes a set of names (server hostnames), some identifying information, such as a geographic address, a public key, a validity period (valid from _X_ until _Y_), a signature, and a few other pieces of data.

<!-- TLS Model -->
This is modeled (see [figure @fig:model]) by the following relationships:

* One _TLS certificate_ belongs to one _Identity_
* One _TLS certificate_ has one _Public key_
* One _Identity_ can have multiple _Public keys_ and multiple _TLS certificates_
* One _Public key_ can belong to different _TLS certificates_ (owned by the same _Identity_)
* One _TLS certificate_ references one or more _DNS names_ (or “host names”, “subdomains”, or just “names”)
* One _DNS name_ can be referenced by multiple _TLS certificates_

### Overview

<!-- TLS Purpose -->
A _public key_ corresponds to a _private key_; both are mathematically “linked” with the property that encryption of data with _one_ of the keys can only be reversed (i.e. the _ciphertext_ being decrypted) with the _other_ key.
This idea is known as asymmetric cryptography, with _RSA_ (named after its inventors, Rivest, Shamir, and Adleman) being the commonly used algorithm [@enwiki1148626325].
The public key within the certificate is used for encryption of data, whereas the private key (kept secret) can be used for signing data.
This means a certificate allows communication with _confidentiality_, _integrity_, and _authenticity_, which is the purpose of Web browsers requesting TLS certificates from TLS servers.

<!-- Formats -->
Certificates come in a few different formats and can be stored in files with different extensions depending on the format [@enwiki1149564995].
These are the two most common ones:

| Format | File extensions | Encoding and structure | Standard |
|--------|------|--------|-------|
| DER (_Distinguished Encoding Rules_) | `.crt`, `.cer`, `.der` | Compact binary format of certificate data | ITU-T X.690 / ISO/IEC 8825-1 [@x690] |
| PEM (_Privacy-enhanced Electronic Mail_) | `.pem` | Base64 encoding of DER certificate | RFC 7468 [@rfc7468] |

<!-- TLS chain intro -->
In order to prove the identity of the certificate holder, the certificate includes a signature.
Simply put, a signature is _ciphertext_ (i.e. the result of encryption), generated via a private key, of the hashed certificate data.
If the identity behind the certificate uses its own private key corresponding to the public key found in the certificate, then this certificate is _self-signed_.
On its own, a self-signed certificate doesn’t confidently prove its corresponding identity.
A website, therefore, usually has its certificate signed via the private key of another certificate; that certificate belongs to a _Certificate Authority_ (CA) which is an entity that serves as the _issuer_ of the website’s certificate [@rfc5280; @SequeiraGhaziOpenSSL].
Usually, there is a so-called _certificate chain_ (or _certification path_) of three certificates served by any TLS server found on the Internet:

* The website’s certificate is the _end-entity certificate_ and is signed by an _intermediate certificate_;
* the _intermediate certificate_ is a _trust anchor certificate_ and is signed by a _root certificate_;
* the _root certificate_ (also a _trust anchor certificate_) is self-signed.

The hierarchy involves an intermediate certificate to mitigate the risk of key compromise of that trust anchor.
The private key of a root certificate is kept physically secure and is never accessed for any signing activity, except the _very few_ times it needs to sign an intermediate certificate, whereas the intermediate certificates constantly need access to their private keys in order to sign end-entity certificates [@SequeiraGhaziOpenSSL].

<!-- Trust store -->
The trust model that involves this chain of certificates vouching for one another is referred to as a _public-key infrastructure_ (PKI); this specific one, using the X.509 standard, is called PKIX.
The root certificate itself is trusted by virtue of being pre-installed on the operating system within the _trust store_, a regularly updated set of established root CA certificates.
The intermediate certificates are usually also pre-installed.

### Workflow from issuance to secure connection

<!-- CA -->
The workflow of _creating_ a certificate and _verifying_ its signature can be explained by the following small example [@SequeiraGhaziOpenSSL].
In the [appendix](#lst:openssl), a code sample using _OpenSSL_, an open-source cryptography library, is presented, which generates certificates, verifies them, and uses them to encrypt data.
There are three subjects involved: Alice _(she / her)_, Bob _(he / him)_, and a certificate authority _(the CA)_.
We start with the CA creating its certificate:

1. The CA generates a private key and a public key.
2. The CA generates a certificate file which includes its public key, its identifying information, a validity period, a serial number, and a signature generated by the private key.

The CA’s certificate is (eventually or already) installed onto Alice’s and Bob’s systems.

<!-- CSRs -->
Next, the subjects seeking to communicate with confidentiality and authenticity, Alice and Bob, create their own certificates, signed by the CA:

3. Alice and Bob each generate their own private key and public key.
4. Alice and Bob each create a _certificate signing request_ file (CSR) which include their individual public keys as well as their individual identifying information.
5. Alice and Bob each send their individual CSR to the CA.
6. The CA creates one certificate for each CSR by attaching a signature, a validity period, and a serial number to it; the signature is generated by hashing and encrypting the CSR data using the CA’s own private key.
7. The CA sends Alice’s certificate to Alice, and Bob’s certificate to Bob.

At no point did any private key get shared.
Also, whether a certificate is an end-entity certificate or a trust anchor certificate is part of the certificate data itself (within the `basicConstraints.cA` extension).
An end-entity certificate cannot issue another certificate.

<!-- Verification -->
If Alice now wishes to communicate with Bob, they’d have to share their certificates, and verify their signatures first:

8. Alice verifies Bob’s certificate by hashing his certificate data and decrypting his signature using the CA’s public key; if both resulting digests match, the verification is successful.
9. Alice extracts Bob’s public key from his certificate.

If Alice publishes a signature, everyone can verify it using her public key, but only she could have created it using her own private key.
Everyone can send Bob an encrypted message using his public key, but only he can decipher it using his own private key.
The signature provides _authenticity_ (and _integrity_), the encryption provides _confidentiality_—both are combined in a TLS connection [@enwiki1148372231].

<!-- Encryption -->
In practice, arbitrary data is not directly encrypted using _public-key cryptography_ due to performance reasons, but more importantly due to the nature of the RSA algorithm: only relatively short messages can be encrypted.
Instead, a different algorithm (usually the _Advanced Encryption Standard_—AES) is used to generate a _symmetric_ key.
The symmetric key _itself_ is encrypted using RSA and can then be shared.

10. Alice generates a symmetric key.
11. Alice encrypts the symmetric key using Bob’s public key.
12. Alice hashes the symmetric key and encrypts the digest using her own private key, resulting in a signature.
13. Alice sends Bob both the encrypted symmetric key as well as the signature.
14. Bob decrypts the encrypted symmetric key using his own private key.
15. Bob verifies the signature using Alice’s public key (i.e. the digest of the symmetric key must match with the decrypted signature).

At this point, the symmetric key has been shared, which can now be used by _both_ subjects to both encrypt and decrypt messages which can be sent over the same secure connection.
At no point did the symmetric key get communicated in _plain text_.

### Revocation {#sec:protocols-tls-revocation}

<!-- What and why? -->
Misissued or forged certificates, or certificates whose private key might have been compromised, can be revoked or held back during investigation of suspicious activity.
A certificate may also need to be revoked for more innocuous reasons, such as the change of some identifier associated with the certificate.
An expired certificate does not need to be revoked.
Two standards for revocations are in common use:

| Name | Standard |
|---|-|
| CRL (_Certificate Revocation List_) | RFC 5280 [@rfc5280] |
| OCSP (_Online Certificate Status Protocol_) | RFC 6960 [@rfc6960] |

<!-- CRLs -->
A **CRL** provides a publicly accessible repository of revoked certificates, identified by their serial number, with the timestamp of revocation attached.
Each CA accepts CRL requests by entities they issued certificates for.
Each CA can then attach the certificate in question to their CRL, and then regularly publish updates of their CRL.
In order to determine whether a certificate is valid, a client will verify the signature using the referenced CA’s certificate, check that the root certificate is found in its trust store, check the validity periods, _and_ make sure the certificates are not on the CRL corresponding to the CA.

<!-- CRL problems -->
Updates to CRLs may be published with a delay, but knowing the _current_ status of a certificate is sometimes critical.
Another problem is the potentially large size of each CRL, and the burden put on the client to store these lists and read from them [@Helme2017; @Ryan2020].
In 2022, browser vendors introduced some improvements into CRL lookups, utilizing browser-specific summaries of revoked certificates issued by several CAs, as well as probabilistic algorithms, minimizing the size of CRLs [@Gable2022].

<!-- OCSP -->
The **OCSP** provides an alternative approach which allows the client to make a request to the issuing CA, which in turn requests a dedicated _OCSP responder_ about a specific set of certificates.
The CA then responds with each certificate’s status.

<!-- Revocation is broken -->
The advantage is that a CA is now leveraged to check a certificate’s status, thus improving performance.
The _disadvantage_ is that _a CA is now leveraged to check a certificate’s status_, which is a privacy concern.
A general problem is that a certificate’s status cannot be retrieved if the OCSP service is not available and a client’s CRL is outdated, but the CRL server is _also_ down.
Web browsers have been opting to simply ignore the response if it does not come back quickly enough, which is a risk, but the alternative would be unavailability or severe latency of many websites [@Helme2017].

<!-- OCSP stapling -->
_OCSP Stapling_ is another approach in which a current, time-limited OCSP response, signed by the CA, is sent together with the TLS certificate by the TLS server itself [@rfc6961].
Now, the browser need not consult the CA itself, increasing performance even further _and_ solving the privacy issue.
In order to force such an OCSP response, the browser needs to include a special flag, `Must-Staple`, in its request.
However, if OCSP stapling goes wrong, the TLS server is not in control, so it cannot resolve any issues.

<!-- CT -->
A remaining problem is the threat of a compromised CA, CRL server, or OCSP responder.
_Certificate Transparency_ (CT), mentioned in [section @sec:background-tls], is one methodology that can be used additionally to make sure that revocations—and any other part of a certificate’s or CA’s life cycle, for that matter—are being monitored for suspicious activity [@Helme2017].
CT _helps_ in identifying certificates that possibly need to be revoked, but certificate revocation itself remains a hard problem.

## The Domain Name System (DNS) {#sec:protocols-dns}

<!-- DNS Intro and references -->
The purpose and architecture of the DNS is documented and specified in RFC 1034 [@rfc1034] and RFC 1035 [@rfc1035]; some important terminology is defined in RFC 2181 [@rfc2181] and RFC 7719 [@rfc7719].
Some important aspects of the history and function of the DNS have already been explained in the introduction, [section @sec:introduction], and the background, [section @sec:background-dnssec].
This section focuses on a few concepts necessary to explain how DANE works: Resource Record structure, DNS names, DNS zones, and name servers.

<!-- RRs -->
A Resource Record (RR) contains information associated with a specific name.
The general structure of an RR includes:

| Part | Explanation |
|--|------|
| Name | Refers to a DNS name, i.e. a hostname or “subdomain name” |
| Time To Live (TTL) | Tells a DNS cache how long the record should be cached for, in seconds, before expiring |
| Class | Used for rare or historical purposes; here, this will always be `IN`, short for _Internet_ |
| Type | The type of record, explaining the usage of the data |
| Data | The actual data of the record |

All RRs that have different _data_, but have the same _name_, _TTL_, _class_, and _type_ form a Resource Record Set (RRSet).

<!-- Zones -->
Names are relative to a _DNS zone_, which is an administrative portion of the DNS, identified by a _fully qualified name_.
Zones are structured hierarchically and are all descendants of the _root zone_ [@LearnMSDNS].
For example: `example.com.` is the name of a DNS zone, which is a child zone of the `com.` zone, which is a child zone of the `.` zone (i.e., the root zone).
A DNS entry of this zone might look like this:

```none
www                       18952  IN  A           93.184.216.34
```

`www` is the name, `18952` is the TTL (5 hours, 15 minutes, 52 seconds), `IN` is the class, `A` is the type (meaning that this record identifies the IPv4 address), `93.184.216.34` is the data (identifying the IPv4 address).
If this RR sits in the `example.com.` DNS zone, then a DNS lookup for the name `www.example.com.` and type `A` will tell a client that the IPv4 address of that name is `93.184.216.34`.
A special name is `@`, which refers to the name of the DNS zone itself, called the “zone apex” or “naked domain”.

<!-- FQNs -->
However, the relative DNS name `www` requires the context of the DNS zone, identified by a RR of type `SOA` (_Start Of Authority_).
To represent an individual record unambiguously, its name needs to be expanded with the name of the DNS zone, creating another _fully qualified name_, in this case `www.example.com.`.
Therefore this example is a more common sight:

```none
www.example.com.          18952  IN  A           93.184.216.34
```

<!-- Presentation vs. wire format -->
These rows are in the so-called _presentation format_, a human-readable representation of RRs.
Over the wire, the _wire format_ is used instead, a compact binary format.
The TTL is represented as a 4-byte number.
The class and type are converted to 2-byte numbers, assigned by the relevant protocol standards—e.g. `1` for the class `IN`; `1` for the type `A`, `15` for the type `MX` (identifying the name of a mail server), etc.
The data is split into a 2-byte length descriptor, the “RDLENGTH”, and an “RDATA” field, whose length is specified by the “RDLENGTH”, in bytes.
The expression of the name over the wire is slightly more complicated: a sequence of lengths and individual labels (e.g. `3` `www` `7` `example` `3` `com` `0`).

<!-- DNS Model -->
In the model presented in [figure @fig:model], the RR and RRSet type we care most about is the `TLSA` type for DANE.
The following relationships so far are visualized:

* One _DNS zone_ has authority over one or more _DNS names_
* One _DNS name_ identifies zero or more individual _TLSA RRs_
* One _TLSA RRs_ belongs to one specific _TLSA RRSet_
* One _TLSA RRSet_ contains one or more _TLSA RRs_
* Any one _TLSA RR_ is identified by one specific _DNS name_
* Any one _DNS name_ authoritatively belongs to one specific _DNS zone_
* One _DNS zone_ can contain zero or more _TLSA RRSet_

For the sake of simplicity, the _class_ and the _TTL_ of all `TLSA` RRs will be the same, so they do not play a major role in identifying different RRSets.
Therefore:

* One _DNS name_ contains at most one _TLSA RRSet_

<!-- Name servers -->
In order to understand the security aspect of the DNS a bit better, it is worth briefly taking a look at the purpose of _name servers_.
Name servers are servers that have two main functions:

* For requests about the zone they are _responsible_ for—i.e. if the name server is an authority of the zone by virtue of being listed as one of the `NS` RRs within that zone—, they directly serve the DNS RRs of the zone as an _authoritative answer_;
* For other requests, they respond with a _non-authoritative answer_ by either
  * performing a DNS lookup themselves (or looking into their cache), then caching the response with the RRs they receive, and then sending it back to the original requesting entity, or
  * sending a response back to the original requesting entity, telling them which other name server to request, which is likely to have an answer.

Whenever a client performs a DNS lookup, it will eventually receive an answer, either directly or indirectly.

<!-- Zone transfer -->
An interesting question is how these name servers receive updates, especially regarding security considerations, and how name servers accept the authority of each other.
If a DNS administrator chooses to update the RRs in their zone, they will perform a _zone transfer_.
Best practice suggests, that a DNS zone has at least _two_ name servers [@IANAReqs], which are only _secondary_ servers, i.e. serving only a read-only copy of the zone data.
Another name server, the _primary_ server, should exist, which contains the _real_ zone data, pushes updates to the secondaries, and is not publicly reachable in the Internet—only the secondaries have access to it [@Jevtic2019].
This concept is known as _hidden primary_.
Making the secondary servers only accept zone transfers from the hidden primary, keeping credentials for server access safe, keeping the secondaries physically separate, and ensuring the physical security of all servers are all important factors when considering name server security [@InfobloxDNS; @HeimdalDNS].
Additionally, other security protocols exist, covered in [section @sec:protocols-other].

## DNS Security Extensions (DNSSEC) {#sec:protocols-dnssec}

<!-- DNSSEC refs -->
DNSSEC is specified in RFC 4033 [@rfc4033], RFC 4034 [@rfc4034], and RFC 4035 [@rfc4035].
The reason why DNSSEC has been established as an extension to the DNS, is explained in the background, [section @sec:background-dnssec].
The _Internet Assigned Numbers Authority_ (IANA) has published a set of general DNS and DNSSEC guidelines [@IANAReqs].
RFC 1912 also explains general tips for correctly configuring DNS [@rfc1912].

<!-- DNSSEC intro -->
This section explains how DNSSEC works.
As previously stated, the DNS Security Extensions provide a public-key infrastructure (PKI) for the DNS in order to achieve data integrity and data origin authentication.
This means it involves public and private keys, as well as signatures created from the private key.
A DNSSEC-validated DNS response ensures the authenticity of the response data—this means a client can be confident that it will arrive at the desired destination.

<!-- RRs -->
DNSSEC introduces a few new record types that every zone can use, the important ones being listed in the following table:

| Type | Meaning |
|-|--|
| `RRSIG` | RRSet Signature |
| `DNSKEY` | Public key |
| `DS` | Delegation Signer |
| `NSEC` | Next Secure |

In short, the public key is stored in a `DNSKEY` RR; the corresponding private key signs an entire RRSet, which is stored in an `RRSIG` RR.
The `RRSIG` has a validity period, so the zone needs to be periodically re-signed.
Since an `RRSIG` RR cannot be easily and securely revoked, it is recommended to not make the validity period too long, especially for critical Resource Records [@rfc7671].

<!-- ZSK vs. KSK -->
Every zone has _two_ underlying types of public and private key pairs:

| Key type | Public key | Private key |
|----|---|-------|
| ZSK (_Zone-signing key_) | Stored in a RR of type `DNSKEY` | Used to produce the signature of any RRSet, which is stored in an RR of type `RRSIG` |
| KSK (_Key-signing key_) | Stored in another RR of type `DNSKEY` | Used to produce the signature of the `DNSKEY` RRSet, specifically, which is stored in another RR of type `RRSIG` |

<!-- Chain of trust -->
A child zone (such as `example.com.`) _owns_ its `DNSKEY` RR with the name `example.com.`.
The _parent_ zone (`com.`) _owns_ a RR of type `DS` and the name `example.com.`, which includes a digest of the public KSK from the child’s `DNSKEY`.
The name of both records is the same, but the ownership is not [@sf734034].
It is the responsibility of the DNS administrator of the `example.com.` zone to transmit the digest of their public key to the administrator of the parent zone, so that it can be inserted there.
The parent zone then _vouches_ for the child zone by virtue of a valid `DS` RR.
The parent zone has its own `DNSKEY` RR, vouched for by _its_ parent zone’s `DS` RR, and so on, until eventually reaching the root zone, whose public key digest is publicly known.
These links form the _chain of trust_ throughout the hierarchy of a DNS zone, from a leaf zone up to the root zone.
This is visualized in [figure @fig:chain-of-trust].
Together with the signing of RRSets, this defines the public-key infrastructure of DNSSEC.

::: {.figure #fig:chain-of-trust}
```{.mermaid format=pdf theme=neutral background=transparent width=240}
flowchart TB
  subgraph "⠀. (root) zone⠀⠀⠀⠀⠀⠀" %% <https://github.com/mermaid-js/mermaid/issues/1420>
    A[. IN DNSKEY] -->|signs| B
  end
  subgraph "⠀com. zone⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀" %% <https://github.com/mermaid-js/mermaid/issues/1420>
    B[com. IN DS] -->|contains digest of| C
    C[com. IN DNSKEY] -->|signs| D
  end
  subgraph "⠀example.com. zone⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀" %% <https://github.com/mermaid-js/mermaid/issues/1420>
    D[example.com. IN DS] -->|contains digest of| E[example.com. IN DNSKEY]
  end
```

Chain of trust in DNSSEC
:::

<!-- Denial of existence -->
Finally, `NSEC` can tell a client what the _next_ child zone in alphabetical order is.
This record type exists for the purpose of authenticating the _denial of existence_ of a specific DNS name.
It prevents an attacker from forging a DNS response that contains RRs while the real response contains none; it also prevents an attacker from doing the opposite: forging a DNS response that contains _no_ RRs while the real response contains _some_.

<!-- Validation -->
A client or a DNS server can always validate DNS responses using DNSSEC, though different types of DNS resolvers exist, not all of which verify DNSSEC signatures.
The validation result ranges from _secure_ (all keys, digests, and signatures correctly match), over _bogus_ (some keys, digests, or signatures don’t match, or some `RRSIG` RRs are expired), to _insecure_ or _indeterminate_ (no DNSSEC deployed).
If the response is not deemed to be fully _secure_, e.g. due to being tampered with, the data simply will not reach its destination; the response will effectively tell the client that no data was retrieved [@ICANNDNSSEC].

<!-- DNSSEC for DANE -->
In the context of _managing_ DANE identities, particularly using `TLSA` RRs, we will assumed that DNSSEC is fully deployed, and all relevant DNS zones are correctly signed and secure.
This is because DANE is based on DNSSEC—or more accurately: DNSSEC is a _security requirement_ for DANE.
It is not a _technical_ dependency, but DANE without DNSSEC is not useful—and the question is not really _“Why is DANE based on DNSSEC?”_.
Instead, there are three facts to consider:

* DANE enables the authentication of servers using public key or certificate associations
* DANE is a set of _DNS-based_ protocols
* The DNS protocol is inherently not very secure because it is susceptible to a variety of data manipulation attacks

If data is possible to be manipulated midway, no authenticity, no integrity can be guaranteed; any benefit, that employing DANE would have provided, is lost.
It is akin to locking a door with a high-quality key and then hiding the key under the doormat, for any sufficiently diligent intruder to find.
DANE is “based” on DNSSEC in the sense that DANE is a set of security protocols that are based on the DNS—and we need security throughout _all_ protocol layers in order to benefit from security at any _one_ layer.

## The DANE TLS association (TLSA) {#sec:protocols-tlsa}

<!-- Intro -->
TLSA is the most interesting binding for DANE, because it relates to the TLS protocol which pervades everyday usage of the Internet.
The first specification of TLSA has been published in August 2012 as RFC 6698, which proposes a protocol to allow DNS zone administrators to deposit the certificate data or public key of the TLS certificate of the corresponding TLS server into their DNS zone—either verbatim or in the form of a digest.
The intended benefit of using the `TLSA` Resource Record is to avoid the dependency on Certificate Authorities to sign and issue certificates for websites [@rfc6698].
Originally, an implementation of this protocol did not involve any particular change in the workings of a TLS server—only a TLS client would need to be extended.
However, in October 2015, the protocol has been updated by RFC 7671, simplified in certain areas and extended in others.
New guidelines and recommendations have been detailed, and new requirements for both TLS clients and TLS servers have been introduced [@rfc7671].
The section will summarize the specifications in order to extract the guidelines and recommendations needed for an identity manager.

<!-- Purpose -->
The purpose of the `TLSA` RRSet is to _authenticate_ a TLS server.
The benefit of _authentication_ is explained by RFC 7671 as follows [@rfc7671]:

> Used without authentication, TLS provides protection only against eavesdropping through its use of encryption.
> With authentication, TLS also protects the transport against man-in-the-middle (MITM) attacks.

<!-- Mechanism -->
The general mechanism of authentication involves the client making a request to the TLS server as well as to the DNS.
The TLS server responds with its certificate chain (part of the TLS handshake) while the DNS responds with the `TLSA` RRSet.
The client then either compares the certificates or the public keys of both pieces of information directly—after hashing and extraction of the public key, if necessary—, or it validates the signature of the TLS certificate using the trust anchor certificate referenced in the `TLSA` RR.
The client only needs _one_ `TLSA` RR for the authentication, but if the client does not support any of the parameter combinations found, it will not use DANE, and, instead, proceed with the TLS handshake normally.
If authentication fails, the client will abort the TLS handshake.
This process is visualized in [figure @fig:dane-validation].[^msd]

  [^msd]: In the sequence diagram, the box with the “par” label and the two regions indicates two processes that execute _in parallel_.
  The box with the “break” label is a conditional execution depending on the condition written inside.

::: {.figure #fig:dane-validation}
```{.mermaid format=pdf theme=neutral background=transparent}
sequenceDiagram
  participant DNS
  participant Client
  participant TLS Server
  par DNS request
    Client->>+DNS: IN TLSA
  and TLS handshake
    Client->>+TLS Server: SYN
  end
  DNS->>Client: DNSSEC validation state
  break if validation state is bogus
    Client->>TLS Server: FIN
  end
  DNS->>-Client: TLSA RRSet
  TLS Server->>-Client: SYN+ACK
  Client->>TLS Server: ACK
  Client->>+TLS Server: ClientHello
  TLS Server->>Client: ServerHello
  TLS Server->>Client: Certificate
  TLS Server->>-Client: ServerHelloDone
  Note over Client: Compare certificate to each TLSA RR
  Note over Client, TLS Server: Key and data exchange, etc.
```

Example flow of TLS server authentication with a DANE-validating client
:::

<!-- Opportunistic Security -->
_Opportunistic Security_ is a validation approach where a client will check if DNSSEC-validated, supported `TLSA` RRs exist, and if they do, then DANE authentication must succeed in order to establish a TLS connection.
If such records do not exist, default to unauthenticated TLS.
If no TLS is available, default to an unencrypted connection.

<!-- DNSSEC -->
Correct DANE deployment hinges on correct DNSSEC deployment.
If a `TLSA` RRSet is found for a particular domain, then a DANE client must use it if and only if the DNSSEC validation state is _secure_.
If it is _bogus_, the TLS handshake must be aborted or not started to begin with.
If it is _insecure_ or _indeterminate_ the `TLSA` RRSet shall not be used for a TLS connection.
Successful DNSSEC validation serves the same purpose as the traditional PKI with CAs: trusted keys sign untrusted keys.
RFC 6698 points out certain advantages of DNSSEC over Certificate Authorities: the public keys published into the DNS are directly tied to DNS names, distribution of keys over DNSSEC is more straightforward, and there is only a single parent zone that vouches for the child zone using signatures, instead of hundreds of possible CAs.
RFC 6698 also points out [@rfc6698]:

> While the resulting system still has residual security vulnerabilities, it restricts the scope of assertions that can be made by any entity, consistent with the naming scope imposed by the DNS hierarchy.
> As a result, DANE embodies the security “principle of least privilege” that is lacking in the current public CA model.

### The `TLSA` Resource Record {#sec:protocols-tlsa-rr}

<!-- Structure -->
The data of a RR of type `TLSA` consists of four fields:

* The TLSA parameters, each an unsigned one-byte number
  1. Usage
  2. Selector
  3. Matching type
* A field of arbitrary length
  4. Certificate data

Each of the TLSA parameters has a specific set of numbers as possible values.
The numbers also have IANA names associated with them, defined in RFC 7218 [@rfc7218], and, separately, RFC 6698 [@rfc6698] defines some labels for each usage.

<!-- Usages -->
The possible values for the **Usage** field are:

| Value | IANA name | RFC 6698 label |
|-|---|-----|
| `0` | `PKIX-TA` | CA constraint |
| `1` | `PKIX-EE` | Service certificate constraint |
| `2` | `DANE-TA` | Trust anchor assertion |
| `3` | `DANE-EE` | Domain-issued certificate |

**Usages** are explained in more depth later in this section.

<!-- Selectors and SPKI -->
The possible values for the **Selector** field are:

| Value | IANA name |
|-|----|
| `0` | `Cert` |
| `1` | `SPKI` |

“SPKI” refers to the _Subject Public Key Info_ of a certificate, which includes the _modulus_ and _exponent_ of the public key, as well as the _algorithm_.
These are technical details on how such a public key is stored, but the details are not important for defining a `TLSA` RR.

<!-- Matching types -->
The possible values for the **Matching type** field are:

| Value | IANA name |
|-|----|
| `0` | `Full` |
| `1` | `SHA2-256` |
| `2` | `SHA2-512` |

<!-- Parameter combinations -->
Given the possible values, there are $4 \times 2 \times 3 = 24$ possible _parameter combinations_ for a `TLSA` RR.
Different kinds of `TLSA` RRs are usually referred to by their parameter combinations in the order _usage_, _selector_, _matching type_.
For example, one might say that a certain domain stores a certificate in a _“`2` `0` `0` `TLSA` record”_ or utilizes a _“`3` `1` `1` `TLSA` record”_.

<!-- Certificate data -->
Finally, the **Certificate data** consists of raw bytes in the wire format, but is represented as a hex string in presentation format.
The **Selector** and **Matching type** fields determine the content of the **Certificate data**.
Basically, the **Selector** tells you if the `TLSA` RR involves the full TLS certificate or just its public key, and the **Matching type** tells you which algorithm, if any, has been used to hash this particular data, whose digest is used as the **Certificate data**.

<!-- Location, underscore -->
The `TLSA` RR is located in the DNS under a name constructed from the port, the transport protocol, and the DNS name providing the target service (e.g. `www.example.com`, providing the Web service by having an IP address of a server that serves content via HTTP).
For HTTPS, this DNS name would be `_443._tcp.www.example.com`, because HTTPS is served over TCP and port 443 by default.
For SMTP, given a DNS name for a mail server such as `mail.example.com`, the resulting DNS name would be something like `_25._tcp.mail.example.com`.
The “underscored” labels like `_443` and `_tcp` are subject to the scoped interpretation rules of RFC 8552 [@rfc8552]; DNS names must not have _arbitrary_ labels that start with an underscore (`_`).

<!-- Model -->
In the model at [figure @fig:model], the final links can now be explained:

* One _TLS certificate_ can be involved in zero or more _TLSA RRs_.
* One _Public key_ can be involved in zero or more _TLSA RRs_.
* One _TLSA RR_ corresponds to either exactly one _Public key_ or exactly one _TLS certificate_, which includes exactly one _Public key_.

<!-- CNAME -->
The final relationship appears to contradict the definition of an RRSet.
However, it is just an _abstraction_:

* One _TLSA RRSet_ is identified by one _**or more**_ _DNS names_

This abstraction can be implemented using the _indirection_ provided by the `CNAME` RR (_canonical name_).
Consider these example RRs:

```none
web.example.com.          3600   IN  CNAME       www.example.com.
m.example.com.            3600   IN  CNAME       www.example.com.
www.example.com.          3600   IN  A           93.184.216.34
```

The two `CNAME` RRs tell a DNS resolver to treat both the name `web.example.com.` and the name `m.example.com.` as if it is the name `www.example.com.`.
As a result, `web.example.com.`, `m.example.com.`, and `www.example.com.` will all have the same IPv4 address, `93.184.216.34`.
This is quite useful in simplifying DNS management.

<!-- CNAME and TLSA -->
Something similar can be achieved with `TLSA` RRs, therefore, backtracking every indirection, it can be said that one `TLSA` RRSet can be “reused” by multiple DNS names.
This is what `TLSA` used with `CNAME` can look like:

```none
a.example.com.            3600   IN  A           192.0.2.1
b.example.com.            3600   IN  A           192.0.2.2
_443._tcp.a.example.com.  3600   IN  CNAME       tlsa._dane.example.com.
_443._tcp.b.example.com.  3600   IN  CNAME       tlsa._dane.example.com.
tlsa._dane.example.com.   3600   IN  TLSA        2 0 1 ( E3B0C44298FC1C149AFBF4
                                                         C8996FB92427AE41E4649B
                                                         934CA495991B7852B855 )
```

Now, `a.example.com` and `b.example.com` can both use the same DANE record.
The aliasing to `tlsa._dane.`… is directly recommended by RFC 7671 [@rfc7671], and `_dane` is guaranteed to be an available underscored name by RFC 8552 [@rfc8552].

### Generating a `TLSA` RR {#sec:protocols-tlsa-generating}

<!-- Algorithm -->
Given a TLS certificate _cert_, a desired **Usage** _u_, a desired **Selector** _s_, and a desired **Matching type** _m_, a `TLSA` RR, in presentation format, can be generated by following these steps:

1. Let _result_ be the list « _u_, _s_, _m_ ».
2. If _s_ is `0` (_`Cert`_),
   1. let _selection_ be the DER-encoded binary data of _cert_.
3. Else, if _s_ is `1` (`SPKI`),
   1. let _selection_ be the DER-encoded binary data of the _Subject Public Key Info_ of _cert_.
4. If _m_ is not `0`,
   1. let _algorithm_ be the hashing algorithm corresponding to the **Matching type** as chosen by _m_, and
   2. let _data_ be HexBytes(_algorithm_(_selection_)).
5. Else, if _m_ is `0`,
   1. let _data_ be HexBytes(_selection_).
6. Append _data_ to the end of _result_.
7. Return _result_, joined by a space (U+0020).

HexBytes is a helper algorithm that takes arbitrary binary data as input and returns a string by replacing every byte by its hexadecimal representation, padded with `0` from the left, with a length of 2 per byte.

<!-- Example -->
Given the following example certificate, we apply this algorithm with the desired parameter combination of _u_ = `3`, _s_ = `1`, _m_ = `1`; _cert_ is defined by the following certificate in PEM format:

```none
-----BEGIN CERTIFICATE-----
MIIFHzCCBAegAwIBAgISA2Ve5XLN8+Ifr+UQjVKP91PiMA0GCSqGSIb3DQEBCwUA
MDIxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MQswCQYDVQQD
EwJSMzAeFw0yMzAxMDExMTE4NDVaFw0yMzA0MDExMTE4NDRaMBgxFjAUBgNVBAMT
DXd3dy5odXF1ZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCh
Rbopbzp5S3UaVKTKbxze3J7MGzxh/i3pYupG2eE0M+h4op+esreqTEiQPDuzM22D
NzvMe8XLd+uyEPloTuIknNvGu/oqiQO860PZBpiiZ7UUpCnPyj/8eLQzc3kyL6Do
TRFTYe+2sQ1X97KIxyQRuVPRtT1wRbbeqMSLc5KK8YP8XBXzAwfVEXw3anHqGzg4
6f2HMp1C3s+5VVycy/22+1lRsse2ywWunJ0TVP6X82JR4n6VxPnPnsNyRMCSI1ob
KsGAVPjCvut6mU1iW75Azibdl1G8KhZOnCJ0zsUzmz3UmO9JQBNMOvw+D5srtgyc
VaiBygz9FoA9qe1TSvF/AgMBAAGjggJHMIICQzAOBgNVHQ8BAf8EBAMCBaAwHQYD
VR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMAwGA1UdEwEB/wQCMAAwHQYDVR0O
BBYEFFaET0SUTRwq0+eFPPZYrrNL5QnHMB8GA1UdIwQYMBaAFBQusxe3WFbLrlAJ
QOYfr52LFMLGMFUGCCsGAQUFBwEBBEkwRzAhBggrBgEFBQcwAYYVaHR0cDovL3Iz
Lm8ubGVuY3Iub3JnMCIGCCsGAQUFBzAChhZodHRwOi8vcjMuaS5sZW5jci5vcmcv
MBgGA1UdEQQRMA+CDXd3dy5odXF1ZS5jb20wTAYDVR0gBEUwQzAIBgZngQwBAgEw
NwYLKwYBBAGC3xMBAQEwKDAmBggrBgEFBQcCARYaaHR0cDovL2Nwcy5sZXRzZW5j
cnlwdC5vcmcwggEDBgorBgEEAdZ5AgQCBIH0BIHxAO8AdgC3Pvsk35xNunXyOcW6
WPRsXfxCz3qfNcSeHQmBJe20mQAAAYVtRSRtAAAEAwBHMEUCIQDmUhMAFtFb2zly
5pl+6M9h9O4mKw0OQ/3D7xLeC4UEXQIgQHDHvHK1GXGO26pX9tTAWDm5FBiu/j5L
x+hyhR4n/CwAdQDoPtDaPvUGNTLnVyi8iWvJA9PL0RFr7Otp4Xd9bQa9bgAAAYVt
RSZeAAAEAwBGMEQCIDL3+ElYFK7YjSFm90/2fB65zSRMja8MH+8s2Eul/KtbAiBV
UFrBFXavwzDyvf7bx8Ac5HU2I9wmwV3gq47UukGNHDANBgkqhkiG9w0BAQsFAAOC
AQEAc5n5XsUA4o52KFGA0wVvG0KT98FxY2qx9g2VvNuZ8sUy8k77gVJQyCFOiNEh
zh/sF9rs/qqEltrOfkzzh95O3uEMTQlQf99i8pgzL+lvAd4MDAK6ZkfUenJWvMRQ
dojdB+XdCN8MxdWM36IhAXf1uXE1sBpgBrgclZ4Rx1n8v6pBExznSDVppQviptid
pIp8dXKxXHPsIu1XLB2wUJ9xOuVpSrXjKA3A5sDQxZYPmcBxZrtEt1EN0jQn/AxY
U5OO85HG7LMlrnBldhiW0Ubxrphu3VgacUBpzJjg7lnln05aKLP/1sJObDCximFp
ugFlnzOt++E34vxYsd5tW6VPoA==
-----END CERTIFICATE-----
```

The DER-encoded binary data of the _Subject Public Key Info_ of the above certificate can be represented by the following hex dump:

```none
00000000  30 82 01 22 30 0d 06 09  2a 86 48 86 f7 0d 01 01  |0.."0...*.H.....|
00000010  01 05 00 03 82 01 0f 00  30 82 01 0a 02 82 01 01  |........0.......|
00000020  00 a1 45 ba 29 6f 3a 79  4b 75 1a 54 a4 ca 6f 1c  |..E.)o:yKu.T..o.|
00000030  de dc 9e cc 1b 3c 61 fe  2d e9 62 ea 46 d9 e1 34  |.....<a.-.b.F..4|
00000040  33 e8 78 a2 9f 9e b2 b7  aa 4c 48 90 3c 3b b3 33  |3.x......LH.<;.3|
00000050  6d 83 37 3b cc 7b c5 cb  77 eb b2 10 f9 68 4e e2  |m.7;.{..w....hN.|
00000060  24 9c db c6 bb fa 2a 89  03 bc eb 43 d9 06 98 a2  |$.....*....C....|
00000070  67 b5 14 a4 29 cf ca 3f  fc 78 b4 33 73 79 32 2f  |g...)..?.x.3sy2/|
00000080  a0 e8 4d 11 53 61 ef b6  b1 0d 57 f7 b2 88 c7 24  |..M.Sa....W....$|
00000090  11 b9 53 d1 b5 3d 70 45  b6 de a8 c4 8b 73 92 8a  |..S..=pE.....s..|
000000a0  f1 83 fc 5c 15 f3 03 07  d5 11 7c 37 6a 71 ea 1b  |...\......|7jq..|
000000b0  38 38 e9 fd 87 32 9d 42  de cf b9 55 5c 9c cb fd  |88...2.B...U\...|
000000c0  b6 fb 59 51 b2 c7 b6 cb  05 ae 9c 9d 13 54 fe 97  |..YQ.........T..|
000000d0  f3 62 51 e2 7e 95 c4 f9  cf 9e c3 72 44 c0 92 23  |.bQ.~......rD..#|
000000e0  5a 1b 2a c1 80 54 f8 c2  be eb 7a 99 4d 62 5b be  |Z.*..T....z.Mb[.|
000000f0  40 ce 26 dd 97 51 bc 2a  16 4e 9c 22 74 ce c5 33  |@.&..Q.*.N."t..3|
00000100  9b 3d d4 98 ef 49 40 13  4c 3a fc 3e 0f 9b 2b b6  |.=...I@.L:.>..+.|
00000110  0c 9c 55 a8 81 ca 0c fd  16 80 3d a9 ed 53 4a f1  |..U.......=..SJ.|
00000120  7f 02 03 01 00 01                                 |......|
00000126
```

Finally, the SHA2-256 digest of the above looks like this in its hexadecimal form:

```none
A67924AFCD895B9661C4C5D67A83215F60B7D0E1DA8A30B67EEC6BD623A5B57C
```

The final data of the resource record looks like this:

```none
3 1 1 A67924AFCD895B9661C4C5D67A83215F60B7D0E1DA8A30B67EEC6BD623A5B57C
```

The exact process is explained in [section @sec:implementation-tools-openssl].

<!-- Usage -->
The **Usage** does not play a role in generating the _content_ of the **Certificate data**, but it very much plays a role in the client validation approach.

<!-- Location -->
The standard allows the presentation format to include arbitrary whitespace and parentheses in the RR data.
The above certificate happens to be issued for `www.huque.com`, therefore the full record may look like this:

```none
_443._tcp.www.huque.com.   7200  IN  TLSA        3 1 1 ( A67924AFCD895B9661C4C5
                                                         D67A83215F60B7D0E1DA8A
                                                         30B67EEC6BD623A5B57C )
```

### Usages and recommendations {#sec:protocols-tlsa-usages}

<!-- Usage intro -->
The Usage field can be split into two bits:

| Bit | `.0` (“-TA”) | `.1` (“-EE”) |
|-|-|-|
| `0.` (“PKIX-”) | `PKIX-TA` | `PKIX-EE` |
| `1.` (“DANE-”) | `DANE-TA` | `DANE-EE` |

In a nutshell,

* “PKIX-” usages should be used for certificates issued or owned by a publicly known CA,
* “DANE-” usages should be used for self-issued or self-owned certificates (i.e. without a publicly known CA),
* “-TA” usages should be used for certificates that issue other certificates, and
* “-EE” usages should be used for leaf certificates.

More specific guidelines are summarized below.

<!-- DANE-EE vs. PKIX-EE -->
A `TLSA` RR with the `DANE-EE` usage (`3`) must directly match an end-entity certificate (leaf certificate) or public key provided by the TLS server.
A `TLSA` RR with the `PKIX-EE` usage (`1`) works similarly, but imposes additional requirements.
The major difference between “PKIX-” and “DANE-” usages is that validation using “PKIX-” usages requires domain name checks, validity period checks from the certificate itself, as well as certification path checks against pre-installed CA certificates in a trust store.
For the “DANE-” usages, no checks against pre-installed CA certificates in a trust store are needed, allowing self-issued certificates to be used.
For the `DANE-EE` usage, _all_ of these checks are optional; the certificate can even be expired—although keeping expired certificates for too long is not recommended.
The domain name checks refer to matching the domain names included in the certificate and the domain name a client uses to reach the server (the “reference identifier”); again, these names need not match with `DANE-EE`.[^uks]
This is also why `CNAME` indirection works best for the “DANE-” usages.
For “PKIX-” usages, indirection is a bit more complicated; this is explained in section 6 of RFC 7671 [@rfc7671].
Furthermore, Certificate Transparency checks are applicable to the “PKIX-” usages, but not to the “DANE-” usages.
In summary, `DANE-EE` is a more lenient, yet more fault resistant usage type than `PKIX-EE`.
The benefit of “PKIX-” usages over not using `TLSA` RRs _at all_ is that the `TLSA` RRs constrain the possible CAs involved in issuing end-entity certificates.

  [^uks]: `openssl` always performs name checks by default without the `-dane_ee_no_namechecks` flag.
  This willful violation of the standard is due to “unknown key share” attacks and cross-origin scripting restrictions [@opensslUKS].
  Unknown key-share (UKS) attacks exploit the lack of name checks.
  If only the public key is available in a `TLSA` RR, a DANE client will not perform proper identity proofing of the TLS server during the TLS handshake, tricking the client into believing the server has a different name, possibly circumventing firewall restrictions or same-origin restrictions [@barnes-dane-uks-00].

<!-- SNI -->
_Server Name Indication_ (SNI) is a TLS extension allowing a client to identify a specific service on a server that offers multiple services on the same IP address.
This extension is mandatory to be used and verified with all usages except `DANE-EE`.
A DANE client is required to specify the “base domain” of the `TLSA` RRSet when using SNI—for example, for `_443._tcp.www.example.com`, the base domain is `www.example.com`.
If name checks are required, then the TLS certificate must be issued for `www.example.com`.
In an `MX` record, a mail server can be identified; the TLS certificate can _also_ be issued for the mail server destination.

<!-- Selector and Matching type recommendations -->
The other two parameters, Selector and Matching type, also have a few recommendations.
For the hashing algorithm (matching type), SHA2-256 is mandatory for both client and server, whereas SHA2-512 is just recommended for the client—it is not to be exclusively provided in `TLSA` RRs.
Publishing the full certificate or public key in the DNS is usually not recommended because the `TLSA` RRs would become very large, presenting problems with UDP transport.

<!-- Scenarios -->
Some interesting scenarios are enabled for both client and server.
If a trust anchor certificate is published to the DNS within a `TLSA` RR, with no hashing performed, a client can retrieve the full trust anchor certificate from the DNS, so the TLS server need not necessarily send the full certificate chain.
The client might extract the public key from the `TLSA` RR, and, with a certificate with “-TA” usages, it will perform certification path checks against it.
However, sending the full certificate chain is still the recommendation for the TLS server in most cases, and DANE clients are not expected to fully support all of these scenarios.
As for the selector, having only a public key association in the DNS, instead of the full certificate, allows a TLS server to negotiate raw public keys (RFC 7250 [@rfc7250]) with a client; the TLS server, in theory, does not need to send the entire TLS certificate chain to the client.
It also allows the server to keep a `TLSA` RR with the same public key while renewing the certificate.
Raw public key negotiation is also possible with the full _certificate_ in the DNS, but clients are not expected to support this scenario.

<!-- DANE-TA -->
Briefly, a `TLSA` RR with the `DANE-TA` usage (`2`) can be used _in lieu of_ one with the `DANE-EE` usage to publish a trust anchor certificate instead of an end-entity certificate to the DNS, while the TLS server can serve one of (potentially) many end-entity certificates—and change them as frequently as desired—, without the need to publish them in the DNS.
A client is expected to use this certificate association to validate the end-entity certificate from the server against the trust anchor certificate from the DNS, forming a complete certificate chain.
A TLS server is expected to still serve the trust anchor certificate in its certificate chain, unless `2` `0` `0` `TLSA` RRs are used _exclusively_ (i.e. the full, unhashed certificate).
Again, self-issued certificates are encouraged with this usage, however, in this scenario, name checks must be performed against the end-entity certificate.
If a certificate has a corresponding `TLSA` RR with the `DANE-TA` usage (`2`) and this certificate is revoked, the corresponding `TLSA` RR must be removed as soon as possible.

<!-- PKIX-TA -->
Finally, a `TLSA` RR with the `PKIX-TA` usage (`0`) can be used to specify a specific root or intermediate trust anchor certificate of a publicly known CA.
It is possible that the specified trust anchor certificate is not found in the client’s trust store; a DANE client is expected to extend the certificate chain from the TLS server by considering the certificates found in the `TLSA` RR.

<!-- Recommendations -->
These are the recommended parameter combinations, based on RFC 7671 [@rfc7671].
For the Matching type field, `*` is either `1` or `2`.

<!-- Recommendations table -->
| Usage | Selector | Matching type | Recommendation |
|-|-|-|-------|
| `0` | `0` | `0` | Avoid: _record size too big_ |
| `0` | `0` | `*` | Likely good[^nr1] |
| `0` | `1` | `0` | Maybe: _compatible with raw public keys, size might be too big_ |
| `0` | `1` | `*` | Likely avoid\textsuperscript{3}<!-- MAKE SURE THIS MATCHES THE NUMBER GENERATED FOR FOOTNOTE [^nr1] --> |
| `1` | `0` | `0` | Avoid: _record size too big_ |
| `1` | `0` | `*` | Likely good\textsuperscript{3}<!-- MAKE SURE THIS MATCHES THE NUMBER GENERATED FOR FOOTNOTE [^nr1] --> |
| `1` | `1` | `0` | Maybe: _compatible with raw public keys, size might be too big_ |
| `1` | `1` | `*` | Good[^nr2] |
| `2` | `0` | `0` | Avoid: _record size too big_ |
| `2` | `0` | `*` | Good |
| `2` | `1` | `0` | Avoid: _clients might not be able to validate this_ |
| `2` | `1` | `*` | Avoid: _constraints from certificate are important_ |
| `3` | `0` | `0` | Avoid: _record size too big_ |
| `3` | `0` | `*` | Possibly good[^ler] |
| `3` | `1` | `0` | Maybe: _compatible with raw public keys, size might be too big_ |
| `3` | `1` | `*` | Good |

  [^nr1]: “`0` `0` `*`”, “`0` `1` `*`”, “`1` `0` `*`” do not have explicit recommendations by either RFC 6698 or RFC 7671, so their recommendation is based on similar parameter combinations.
  [^nr2]: “`1` `1` `1`” is used as a common example, therefore we can assume that “`1` `1` `*`” is a recommendation for the usage `PKIX-EE`.
  [^ler]: The IETF DANE Working Group has recommended against “`3` `0` `*`” on the basis of a lack of reliable automation.
  The short-livedness of _Let’s Encrypt_ end-entity certificates (90 days) requires a frequent change of the `TLSA` RR [@LE30X].

\newpage

In general,

* a Matching type of `1` is required, but `2` can also be used _additionally_;
* storing the full, unhashed certificate in a `TLSA` RR should be avoided due to the large size and associated problems with UDP packages;
* raw public keys can be negotiated, but avoid them if the other constraints found in the certificate are important, or if the TLS server needs to be forced to transmit the full certificate during the TLS handshake.

### Client and server requirements {#sec:protocols-tlsa-requirements}

<!-- Parameter combination Groups -->
The TLSA RRSet can be grouped by parameter combination.
For example, all TLSA RRs of a TLSA RRSet with the parameter combination `3` `1` `1` form the “`3` `1` `1` group”.
For _every_ such non-empty parameter combination group, the TLSA RRs within the group must be sufficient to authenticate a server.
This means, for each such non-empty group, there must be at least one `TLSA` RR corresponding to the _current_ certificate from the TLS server.
It is acceptable—and in transitional situations (e.g. key rollovers), _required_—to keep `TLSA` RRs of _future_ or _past_ certificates.
The SHA2-512 algorithm is not universally supported, so as a general rule, for every `TLSA` RR with Matching type `2`, there must exist a `TLSA` RR with the same parameter combination, except with the Matching type `1`.

<!-- List -->
The following list is a collection of other client and server requirements provided in RFC 7671.
In this context, `*` means _any value for the given field_.

* For “`0` `*` `*`”, “`1` `*` `*`”, and “`2` `*` `*`”, the validity period of the TLS certificate must _not_ be ignored for validation
* For “`2` `*` `*`”, if _not_ all `TLSA` RRs have “`2` `0` `0`”, then the full TLS certificate chain is required from the server
* For “`2` `*` `1`”, and “`2` `*` `2`”, the TLS server must include the TA certificate in the certificate chain
* For “`2` `1` `0`”, the TLS server _should_ include the TA certificate in the certificate chain
* For “`3` `*` `*`”, the validity period of the TLS certificate must be ignored for validation
* For “`3` `1` `*`”, and, optionally, “`3` `0` `0`” the TLS server need not necessarily send the TLS certificate

Parameter combinations “`0` `*` `*`” and “`1` `*` `*`” work similarly, but with additional PKIX-related requirements, explained earlier in this subsection.
Not having any `TLSA` RRs is, of course, allowed, but such a domain would not be supporting DANE (for TLS), by definition.
Publishing both “PKIX-” and “DANE-” usages is allowed.
It is currently recommended that a DANE client validates `TLSA` RRs of usages `2` (`DANE-TA`) and `3` (`DANE-EE`), but offers no support for `0` (`PKIX-TA`) and `1` (`PKIX-EE`) [@rfc7671].
There are some additional requirements that are not directly related to `TLSA` RR management, e.g. the requirement to use at least TLS version 1.0 and the recommendation to use TLS version 1.2 for clients.

<!-- DNS caches and general workflow -->
Because name servers cache RRs for some amount of time, the deployment of DANE-related RRs needs to consider the TTL of each record.
In particular, the general procedure of publishing a new certificate involves publishing the corresponding `TLSA` record _first_, then waiting some amount of time to ensure old `TLSA` records have expired from any caches, and _then_ publishing the certificate on the TLS side.
The following is a summary of the TLSA _publisher_ requirements from RFC 7671, Section 8.4 [@rfc7671]—this is what this thesis is trying to establish.

> In summary, server operators updating TLSA records should make one change at a time. The individual safe changes are as follows:
> 
> * Pre-publish new certificate associations that employ the same TLSA parameters […] as existing `TLSA` records, but match certificate chains that will be deployed in the near future.
> * Wait for stale `TLSA` RRSets to expire from DNS caches before configuring servers to use the new certificate chain.
> * Remove `TLSA` records matching any certificate chains that are no longer deployed.
> * Publish `TLSA` RRSets in which all parameter combinations […] present in the RRset match the same set of current and planned certificate chains.

<!-- Removal -->
A `TLSA` RR can simply be removed from the DNS.
Removing it will cause it to be eventually removed from all caches; at the latest, the validity period of the corresponding `RRSIG` RR will eventually expire.
In order to mitigate key compromise, it is recommended to keep the TTL small, so a revoked and removed `TLSA` RR will be removed from caches sooner [@rfc7671].

<!-- Other sections -->
In [section @sec:standards-idm-dane], various ways to _change_ a `TLSA` RRSet will be examined.
In [section @sec:implementation], the implementation aspects of these rules and the RRSet changes will be examined.

### Other DANE bindings: IPSECKEY, SSHFP, OPENPGPKEY, SMIMEA

<!-- Table -->
DANE is not “one thing”.
There are currently five different “bindings” for DANE, i.e. five different certificate or public key associations.
They are listed here:

| RR type | Standards | Usage |
|--|----|----|
| `IPSECKEY` | RFC 4025 | IPsec keying material |
| `SSHFP` | RFC 4255 | SSH fingerprint |
| `TLSA` | RFC 6698, RFC 7671, RFC 7672 | TLS certificate or public key |
| `OPENPGPKEY` | RFC 7929 | OpenPGP public key |
| `SMIMEA` | RFC 8162 | S/MIME certificate association |

## Other DNS- and TLS-related security protocols {#sec:protocols-other}

<!-- Questions -->
There are several other protocols related to Internet security that might need to be considered in coordination with DANE.
This section provides an overview and briefly describes them—these three types of questions will be addressed:

* _How is security aspect_ X _achieved?_
* _How is part_ Y _of the Internet secured?_
* _Where exactly does DANE fit into the realm of DNS security?_

<!-- Table -->
This is an overview of relevant protocols and their specifications:

| Protocol | Specification |
|---------|--|
| DoT (_DNS over TCP_) | RFC 7766 [@rfc7766] |
| DoH (_DNS over HTTPS_) | RFC 8484 [@rfc8484] |
| TSIG (_Transaction Signature_) | RFC 2845 [@rfc2845] |
| DNSCurve | RFC draft [@dempsky-dnscurve-01] |
| TLSA for SMTP (_Simple Mail Transfer Protocol_) | RFC 7672 [@rfc7672] |
| TLS server identity check for email-related protocols | RFC 7817 [@rfc7817] |
| DMARC (_Domain-based Message Authentication, Reporting, & Conformance_) | RFC 7489 [@rfc7489] |
| HSTS (_HTTP Strict Transport Security_) | RFC 6797 [@rfc6797] |
| HPKP (_Public Key Pinning Extension for HTTP_) | RFC 7469 [@rfc7469] |
| `CAA` RRs (_Certification Authority Authorization_) | RFC 6844 [@rfc6844] |
| `HTTPS` and `SVCB` RRs | RFC draft [@ietf-dnsop-svcb-https-12] |
| ACME (_Automatic Certificate Management Environment_) | RFC 8555 [@rfc8555] |

<!-- List -->
The purposes of these protocols are as follows:

* **DoT** specifies the usage of TCP as a transport protocol for DNS, as opposed to UDP.
  It allows larger message sizes (needed for DNSSEC) and certain security features.
* **DoH**, additionally, specifies DNS transport over HTTPS (and thereby also TLS), i.e. _encrypted_—this is what provides _confidentiality_ and additional _integrity_ in DNS transport.
* **TSIG** is used for authenticating _DNS transactions_, e.g. zone transfers.
  It involves timestamps to prevent replay attacks, i.e. intercepting and re-sending a packet—even if encrypted.
  However, it can only secure the immediate connection between two name servers; it does not provide end-to-end integrity like DNSSEC [@rfc3833].
* **DNSCurve** provides encryption and authentication of DNS packets between name servers.
  It attempts to provide _confidentiality_, _integrity_, and even some _availability_ by defending against replay attacks, discarding forged DNS packets.
* **TLSA for SMTP** uses DANE `TLSA` RRs for mail server authentication during e-mail transport using the SMTP.
* **RFC 7817** defines rules for a _“consistent TLS server identity verification procedure across multiple email-related protocols”_, which can be used in tandem with DANE.
* **DMARC** is a mechanism that allows mail servers to _“associate reliable and authenticated domain identifiers with messages, communicate policies about messages that use those identifiers, and report about mail using those identifiers”_.
* **HSTS** provides a policy for websites or web browsers to force a secure connection over HTTPS (and TLS) by default.
* **HPKP** is a deprecated protocol which was used to provide a policy for websites to force a specific certificate or public key to be used in a TLS connection.
* The **`CAA` RR** provides a policy for authorizing only specific CAs for a particular domain, reducing the risk of misissuing.
* The **`HTTPS` RR** and **`SVCB` RR** provides a mechanism to optimize connection negotiation between a client and a server via the DNS, and promises to improve performance and privacy.
* **ACME** offers automatic authentication of a domain for issuing a TLS certificate by receiving a token from the issuing CA and placing it either into the DNS zone (“DNS-01 challenge”) or the web service (“HTTP-01 challenge”).
  The CA will then attempt to request a `TXT` RR (used for arbitrary text) from a specific underscored domain or make an HTTP request to a specific end point, and check if the token is correct.

<!-- Model -->
[Figure @fig:protocols] is a rough visualization of where these protocols play a role.

![Protocols involved in DNS, Web, and Mail](Protocols.pdf){#fig:protocols width=480}

<!-- Risk assessment -->
DNSSEC and DANE are only part of the puzzle.
In order to maximize security, several protocols need to be considered together, as well as access control and physical server security.
But, as mentioned before, it is required to weigh the cost and benefit of every protocol.
Naively accumulating several individual protocols that promise to increase security, without proper risk assessment, might in turn impair performance or privacy, ironically _decreasing_ security (especially availability).

# The standards aspect of DANE identity management {#sec:standards}

<!-- Standards intro -->
This section examines various standards documents that are about how to approach identity management in general, not about any specific protocol or implementation.
First, we clarify the terminology needed to understand the field of identity management.
Next, we take a look at the identity life cycle and how it relates to DANE identity management.

## Terminology {#sec:standards-terminology}

<!-- Terminology intro -->
The title of this thesis implies that we are going to _manage identities_.
Before we can do that, we need to understand what these so-called “identities” _are_.
This will uncover the complex taxonomy involved in identity management.

### Attribute, identifier, entity, identity {#sec:standards-terminology-identity}

<!-- Definition identity -->
We can start with the term **identity**.
The Internet Security Glossary (RFC 4949) offers the following definition [@rfc4949]:

> The collective aspect of a set of attribute values (i.e., a set of characteristics) by which a system user or other system entity is recognizable or known.

It further explains that the set of attributes used to define identities _“should be sufficient to distinguish each entity from all other entities, i.e., to represent each entity uniquely”_ and that the set _“should be sufficient to distinguish each identity from any other identities of the same entity”_.

The ISO standard for _“A framework for identity management”_ (ISO/IEC 24760-1) defines _identity_ (or _partial identity_) as follows [@ISO24760-1]:

> Set of attributes related to an entity.  
> Note: An entity can have more than one identity.  
> Note: Several entities can have the same identity.

Whether one should be able to _uniquely_ identify an entity depends on the context.
Security vulnerabilities happen when a client is led to believe that a server has an identifier that, in reality, should only identify _another_ server—either accidentally or maliciously—; this is what impersonation, man-in-the-middle attacks, etc. are all about.
The uniqueness aspect of the _identity_ term, as well as how to approach the confusion between two identities, are both important to specify.

<!-- Definition entity -->
_Entity_ and _attribute_ are central terms that help define what an _identity_ is.
As for **entity** (or _system entity_), RFC 4949 defines it as [@rfc4949]:

> An active part of a system—a person, a set of persons (e.g., some kind of organization), an automated process, or a set of processes […]—that has a specific set of capabilities.

And the ISO standard says it is the following [@ISO24760-1]:

> Item relevant for the purpose of operation of a domain that has recognizably distinct existence.  
> Note: An entity can have a physical or a logical embodiment.  
> Example: a person, an organization, a device, a group of such items, a human subscriber to a telecom service, a SIM card, a passport, a network interface card, a software application, a service, or a website

<!-- Definition attribute -->
An **attribute** is simply defined by ISO [@ISO24760-1] as this:

> Characteristic or property of an entity  
> Example: An entity type, address information, telephone number, a privilege, a MAC address, a domain
name are possible attributes.

And, finally, RFC 4949 [@rfc4949] defines it as this:

> Information of a particular type concerning an identifiable system entity or object

<!-- Definition identifier -->
A special type of attribute is the **identifier**.
RFC 4949 defines it as [@rfc4949]:

> A data object—often, a printable, non-blank character string—that definitively represents a specific identity of a system entity, distinguishing that identity from all others.

And ISO defines it as [@ISO24760-1]:

> Attribute or set of attributes that uniquely characterizes an identity in a domain

Another common theme of the usage of the word “_identifier_” is the fact that an identifier is an _external_ description, meaning that the entity can be reached by it from an entity outside of the system that the target entity belongs to.

<!-- Identity model -->
In the Wikipedia article about _Identity management_, the relationship between _entity_, _identity_, and _attribute_ is explained by a so-called “axiomatic model”, shown here in [figure @fig:identity-model] [@enwiki1151190645].

::: {.figure #fig:identity-model}
```{.mermaid format=pdf theme=neutral background=transparent width=360}
erDiagram
  Identity }|--|| Entity: "corresponds to"
  Identity }|--|{ "Attribute or Identifier": "consists of"
```

“Axiomatic model” of an identity
:::

<!-- Summary identity -->
Essentially, an identity corresponds to an entity, i.e. a subject or object, and consists of a few attributes that describe the identity.
The identifier has the purpose of unique **identification** of an identity, i.e. _“recognizing an entity in a particular domain as distinct from other entities”_ [@ISO24760-1].

<!-- Perspective -->
RFC 6125 distinguishes two terms for identity (or identifier), based on _perspective_, as shown in the following table [@rfc6125].
Note that both terms refer to the _same_ identity: the identity of the server.
Based on who is referencing this identity, the term—and, in fact, the attributes, in some cases—will differ.

| Identity in question | Entity referencing this identity | Term |
|---|:-:|--|
| Identity of the server | Server | **Presented identity** |
| Identity of the server | Client | **Reference identity** |

<!-- Model -->
[Figure @fig:model] models the relationship between _Identity_ and different attributes.
For example, some _Public key_ can be an attribute for some usage.
The identifier is linked to one or more _DNS names_, transitively via one or more _TLS certificates_.
An actual TLS certificate also includes the subject’s name, their email, their organization’s name, etc., all of which are attributes associated with the identity.
The _entity_ is not part of the diagram, but a _Website_ or _Web service_ can be considered to be the relevant entity.
However, Web services are not in the scope of identity management that this thesis is about.
Instead, the _TLS certificates_ serve as a proxy for Web services in that diagram.

<!-- Identity guidelines -->
The ISO standard mandates that an identity represents an entity, and presumes that the identity is persistently stored [@ISO24760-1].
There are two immediate ideas as to what the _entity_ is that a DANE identity corresponds to: either a certificate or a public key.
But TLS certificates only have a limited lifetime.
Public keys can be persisted across several certificate renewals, yet it is also recommended to rotate them once in a while.
However, TLS certificates are also linked to one or more _names_, which is an important attribute of an identity.
Names—or, more specifically, _hostnames_—are persisted over a significantly longer lifetime.
The expected long lifetime of a hostname and the fact that a certificate applies to a specific set of hostnames, even after being renewed, suggest a third, arguably more intuitive, idea: the entity could be a “track” of TLS certificates which are issued for the same hostnames and being continually renewed.
Only one certificate can be the _currently used_ one; it will have a different serial number, a different validity period, maybe a different public key as the previous one—but it will belong to the same _track_ as the previous one.

<!-- Identity attributes -->
Ultimately, an identity is a collection of attributes, so, if we accept the third option as the definition of _entity_, we can deduce the relevant attributes required to describe the _identity_.
The ISO standard proposes the following checklist for what kind of attributes need to be supplied [@ISO24760-1]:

> * Any information required to facilitate the interaction between the domain and the entity for which the identity is created;
> * Any information required for future identification of the entity, including description of aspects of the physical existence of the entity;
> * Any information required for future authentication of the entity’s identity; or
> * One or more reference identifiers.

<!-- DANE identity attributes -->
The following list provides a minimal set of attributes relevant for DANE identity management; an actual implementation might require more knowledge.

* The _current_ certificate in use, from which one can derive:
  * the list of hostnames that the current certificate applies to, and
  * the validity period
* Information needed for renewal of the certificate, such as:
  * a stored certificate signing request (CSR), and
  * authentication tokens for CAs
* The deployment state of the current certificate
* The _current_ `TLSA` RRs that correspond to the certificate, including:
  * their deployment states
* The policies to be applied when generating the `TLSA` RRs
* References to future or past certificates needed for preparation of `TLSA` RRs, logging, etc.
* Perhaps a human-readable, easily identifiable label, which can serve as the identifier

### Authentication, Verification

<!-- Authentication intro -->
The central operation one can perform with an identity is _authentication_.
This is exactly what _authenticity_ is: communication with some entity is authenticated if and only if a subject has managed to prove that the entity that performs some action, such as sending a message, truly is the entity that the subject expects it to be; the subject is trying to _verify_ the entity’s identity.
And this, of course, is exactly what explains the “Authentication” in “DNS-based Authentication of Named Entities”.

<!-- Definition authentication -->
The ISO standard defines “**authentication**” as [@ISO24760-1]:

> Formalized process of verification that, if successful, results in an authenticated identity for an entity  
> Note: The authentication process involves tests by a verifier of one or more identity attributes provided by an entity to determine, with the required level of assurance, their correctness

RFC 4949 concurs with its definition of “**authenticate**” [@rfc4949]:

> Verify (i.e., establish the truth of) an attribute value claimed by or for a system entity or system resource.

Further, under the definition of _authentication_, it points out, that authentication consists of two steps:

1. Identification – _“Presenting the claimed attribute value […] to the authentication subsystem”_
2. Verification – _“Presenting or generating authentication information […] that acts as evidence to prove the binding between the attribute and that for which it is claimed”_

In DANE, the _identification_ is simple: the identifier is the hostname, e.g. `www.example.com`.
Additionally, the _claimed attributes_ can be found in the DNS at the name `_443._tcp.www.example.com.`.
Given a “`3` `1` `1`” `TLSA` RRSet group, for example, the _claim_ is quite simple: _one_ of the presented values in the RRSet group contains the SHA2-256 digest of the target server’s public key, _exactly_, _and_ the public key is a value that can be received via a TLS handshake.
The _verification_ is also simple: first, connect to the hostname obtained from the base name of the RRSet’s _name_ (i.e. the DNS name without the `_443._tcp.`) and perform a TLS handshake with that server in order to receive its public key in DER format (or receive its end-entity TLS certificate and extract it); then, simply hash the public key using the SHA2-256 algorithm to obtain the relevant digest.
If one of the `TLSA` RRSets in the “`3` `1` `1`” group is found, whose certificate data field is an exact match to this digest, then the server is successfully authenticated; otherwise, it is not.
The verification process is, of course, a bit more involved with the other `TLSA` usages, as described in [section @sec:protocols-tlsa-usages].

<!-- Control aspect -->
The NIST standard about _Digital Identity Guidelines_ (NIST SP 800-63-3) offers a definition for _digital authentication_ as follows [@NIST800-63-3]:

> Digital authentication establishes that a subject attempting to access a digital service is in control of one or more valid authenticators associated with that subject’s digital identity.

It further describes the authentication process, which includes the important first step of _“the claimant demonstrating to the verifier possession and control of an authenticator that is bound to the asserted identity”_.

<!-- Control via DNSSEC -->
The NIST standard focuses more on authenticating and authorizing _users_ rather than _servers_, but it offers the valuable insight that during authentication, the verifier establishes whether the entity, who claims to be a specific identity, is in _control_ of an authenticator (i.e. a “token”, such as a digest).
Control is demonstrated via DNSSEC: the DNS zone owner owns their public and private keys for their zone, and trust in these keys is established via the chain of trust described in [section @sec:protocols-dnssec].
Only a signed zone is trusted; only with those private keys, it is possible to sign the zone; only the DNS zone owner knows those private keys.

<!-- Verification -->
The ISO standard has more detailed guidelines attached to its definition of “**verification**” [@ISO24760-1]:

> Verification typically involves determining which attributes are needed to recognize an entity in a domain, checking that these required attributes are present, that they have the correct syntax, and exist within a defined validity period and pertain to the entity

<!-- RFC 6125: general authentication guidelines -->
RFC 6125 provides the following guideline about _authentication_ [@rfc6125]:

> In general, a client needs to verify that the server’s presented identity matches its reference identity so it can authenticate the communication.

RFC 6125 also provides and aggregates other important concepts and general guidelines about authentication.
For example, it references RFC 2818, which contains the following guideline on how name checks against a TLS certificate should be performed:

> If a `subjectAltName` extension of type `dNSName` is present, that MUST be used as the identity.
> Otherwise, the (most specific) `Common Name` field in the `Subject` field of the certificate MUST be used.
> Although the use of the `Common Name` is existing practice, it is deprecated and Certification Authorities are encouraged to use the `dNSName` instead.

## Identity life cycle and identity management {#sec:standards-idm}

<!-- IdM intro -->
Even if the term “_identity_” has been clarified, there is still one _more specific_ definition pertaining to the system that we desire to establish: “**identity management**” (or “**IdM**” for short).

<!-- IdM -->
ISO/IEC 24760-1 provides the definition [ISO24760-1]:

> Processes and policies involved in managing the lifecycle and value, type and optional metadata of attributes in identities known in a particular domain.

<!-- Life cycle -->
The _life cycle_ defined in the ISO standard is shown in [figure @fig:life-cycle].
It is the state diagram of an identity that starts in the _Unknown_ state and becomes known by _enrollment_; it then goes through the process of _activation_, optionally _suspension_ and _reactivation_.
An identity manager can later _archive_ or _delete_ the identity.
The states change back and forth when one wishes to change the identity.
The rest of this section will look at each transition in detail and apply it to the identity which emerges from a DNS name and the DANE TLSA association corresponding to a TLS certificate.

::: {.figure #fig:life-cycle}
```{.mermaid format=pdf theme=neutral background=transparent width=480}
stateDiagram-v2
  Unknown --> Established: enrollment
  Archived --> Established: enrollment, restore
  Established --> Active: activation
  Active --> Established: identity adjustment
  Active --> Suspended: suspension
  Active --> Archived: archive
  Active --> Active: maintenance
  Active --> Unknown: delete
  Suspended --> Active: reactivation
  Archived --> Unknown: delete
```

Identity life cycle
:::

### Life cycle event considerations for a DANE TLSA identity {#sec:standards-idm-dane}

<!-- Life cycle intro -->
All states and transitions are explained in detail in ISO/IEC 24760-1 [@ISO24760-1].
We can now apply this standard to DANE identities.

<!-- Enroll, established -->
When a DANE identity is being **enrolled**, all required attributes must be specified before the identity can be **established**.
The entity attributes enumerated in [section @sec:standards-terminology-identity] must be provided.
Some attributes can be derived, some attributes need to be provided by the user.
The data provided by the user also needs to be verified in a process known as _identity proofing_.
As soon as all information necessary for automatically managing the DANE records and TLS certificates has been provided and verified, the identity has been **established**.

<!-- Activation, active -->
**Activation** is the process of adding the established identity to the list of identities which are _actively used_ for the core purpose.
For DANE, the core purpose is server authentication, so an **active** identity would be one with a _currently used_ TLS certificate and one or more _functioning, deployed_ `TLSA` RR associated with it.
A DANE client is expected to validate the server using an active identity.

<!-- Suspension, reactivation, suspended, archival, archived, restoration, deletion -->
**Suspension** requires the identity to be unusable, which can be reversed with **reactivation**.
A **suspended** identity must be indicated as such.
**Archival** is the reversible removal of the identity from the identity manager.
An **archived** identity exists for statistical purposes, e.g. logging, but can also serve the identity proofing process during **restoration**.
An archived identity is also useful to prevent conflicts with an identity about to be enrolled.
For example, reusing a hostname is theoretically possible, but certain DNS providers or TLS providers might disallow it.
The DANE identity manager must ensure that registering a hostname will succeed in both providers, even if the hostname has not been used before.
Finally, **deletion** is the permanent removal of an identity from the identity manager.

<!-- TLSA and TLS deletion -->
Since generating `TLSA` records is trivial, a record can simply be removed from the DNS and, later, generated anew.
TLS certificates can also easily be re-issued and renewed.
An expired or revoked certificate can simply be replaced by a renewed certificate within the same identity, which does not require archival or suspension of the DANE identity, because this is a normal part of a _TLS certificate life cycle_.
Deletion of DANE identities is useful if one wishes to discontinue a service.
Removal of DANE _support_ can be thought of as a suspension, especially regarding the requirement of a _functioning, deployed_ `TLSA` RR in the _active_ state.

<!-- Separation of identities for archival and suspension -->
However, the _TLS certificate_ would still need to be in use, even if DANE support is switched off (for whatever reason).
This calls for a separation between DANE identity management and TLS identity management.
Similarly, since various DNS entries are part of the identity of a _server_, their changes must also be considered, which calls for a separation between DANE identity management and DNS identity management.
Obviously, the management of DNS, of TLS, and of DANE are all intertwined, because TLS and DNS management needs to be synchronized in order for DANE to work properly.
Fortunately, there are plenty of existing DNS managers and TLS certificate managers; the challenge relevant here is to create a system, which uses DANE identities as a basis, and communicates with two _subsystems_—one being a DNS manager, the other being a TLS certificate manager, both of which might have their own identity interpretation.

### DANE identity adjustment and maintenance: the “state transitions” of a `TLSA` entry {#sec:standards-idm-transitions}

<!-- Identity adjustment and maintenance -->
In the life cycle shown in [figure @fig:life-cycle], there are two state transitions whose DANE-related interpretation is a bit more involved.
**Identity adjustment** and **maintenance** both involve changes in the identity attribute.
The difference between them is that the _identity adjustment_ is a change that justifies a change in the activation state.
In this subsection, we will look at possible changes to the identity attributes and determine what needs to be done in these cases.

<!-- Change in attributes -->
If the _current_ certificate changes, be it due to expiration or revocation, then a new `TLSA` RR could be generated, but does not always need to be.
For example, if all affected `TLSA` RRs only have the selector `1` (`SPKI`) and the public key did not change across certificate replacement, then nothing needs to be done.
If, however, any `TLSA` RR will change due to any kind of certificate change (or if a new record will be created), then updating the `TLSA` RR needs to be coordinated carefully.
Similarly, if one decides to change the desired TLSA parameters for any certificate, invoking some change in existing `TLSA` RRs, updating them needs to be done carefully.

<!-- TLSA state transitions -->
The following state transitions are covered by section 4.3 and section 8 in RFC 7671 and described there [@rfc7671].
Note that these procedures only need to be performed if a change in the TLS certificate would cause a change in the `TLSA` RR.
Also, note that publishing duplicate DNS records is implicitly ignored.

<!-- PKIX usages ↔ DANE usages -->
**Switching between “DANE-” and “PKIX-” usages.**
Switching between the usages `PKIX-TA` (`0`) and `DANE-TA` (`2`), or between `PKIX-EE` (`1`) and `DANE-EE` (`3`) is motivated due to a possible future recommendation that one usage should be preferred over the other.
During adoption of this recommendation, it is expected that some clients will still only support the less preferred set of usages.
A TLSA publisher should therefore gradually switch from one usage to the other by first adding the newly preferred usage.
Once enough clients have switched over to the preferred usage, the TLSA publisher can eventually remove the old usage.

<!-- Algo: Adding usage -->
Adding a usage _uNew_ for a certificate _cert_ means:

1. Let _tlsaRecordsOld_ be the list of all `TLSA` RRs that correspond to certificate _cert_.
2. For each _tlsaRecordOld_ of _tlsaRecordsOld_,
   1. let « _u_, _s_, _m_ » be the parameter combination of _tlsaRecordOld_,
   2. let _tlsaRecordNew_ be a new `TLSA` RR generated from _cert_ with the desired parameter combination « _uNew_, _s_, _m_ »,
   3. publish _tlsaRecordNew_.

The name and TTL of the new RR can be the same; the parameter combination will simply be different.
In this case, the certificate data does not necessarily change.
The algorithm for generating a new `TLSA` RR is described in [section @sec:protocols-tlsa-generating].
A lot of time can pass between the addition of the new usage and removal of the old usage, the point of which is to not cause any disruption of clients being able to use the existing set of `TLSA` RRs for validation.

<!-- Key Rollover and certificate change -->
**Server key rollover or certificate change with fixed TLSA parameters.**
This is where DNS caching becomes relevant: a new `TLSA` RR cannot simply _replace_ the old one.
Instead, some time needs to pass (at least twice the TTL) during a transitional state in which a `TLSA` for the new key and one for the old key needs to be published simultaneously.
The RFC only specifies the key rollover together with the recommendation of publishing a “`3` `1` `1`” `TLSA` RR.
If only the certificate got renewed, without changing the public key, this would not involve a change in any such `TLSA` RR.
However, given the general publisher requirements from [section @sec:protocols-tlsa-requirements], we can generalize the following process to a replacement of the certificate as well.
For example, this should work for “`3` `0` `1`” `TLSA` RR, which contain a digest of the full certificate.

<!-- Algo: switch cert or public key -->
Switching from the existing certificate _certOld_ to the new certificate _certNew_ involves the following steps:

1. Let _tlsaRecordsOld_ be the list of all `TLSA` RRs that correspond to _certOld_.
2. Let _tlsaRecordsNew_ be an empty list.
3. For each _tlsaRecordOld_ of _tlsaRecordsOld_,
   1. let « _u_, _s_, _m_ » be the parameter combination of _tlsaRecordOld_,
   2. let _tlsaRecordNew_ be a new `TLSA` RR generated from _certNew_ with the desired parameter combination « _u_, _s_, _m_ »,
   3. append _tlsaRecordNew_ to _tlsaRecordsNew_,
   4. publish _tlsaRecordNew_.
4. Wait at least 2× TTL.
5. For each _tlsaRecordNew_ of _tlsaRecordsNew_,
   1. assert that _tlsaRecordNew_ is now populated in DNS caches.
6. Publish _certNew_.
7. Assert that TLS server now serves _certNew_.
8. For each _tlsaRecordNew_ of _tlsaRecordsNew_,
   1. delete _tlsaRecordNew_.

<!-- Switching TLSA params general -->
**Switching TLSA parameters in general.**
This use case is only an addendum to the previous use case of a server key rollover.
If one wishes to rotate the server key while switching to different TLSA parameters, the RFC explains this briefly:

> In this case, publish the new parameter combinations for the server’s existing certificate chain first, and only then deploy new keys if desired.

Switching from “`1` `1` `1`” to “`3` `1` `1`” is the example provided by the RFC, however the RFC is not perfectly clear here.
In the end, this works out fine, because publishing the `TLSA` RR with the new parameter combinations indicating the new usage will not “break” validation.
The key rollover involves the _waiting_ step which guarantees that caches contain the fresh data.
This process also does not contradict the first state transition (_Switching between “DANE-” and “PKIX-” usages_), because the motivation is different: here, we simply want to replace the usage right away (but with the required minimum waiting time), with no regard to client support; in the other case, we wanted to gradually migrate to a new recommendation, while still supporting as many clients as possible.

<!-- 3 1 X → 2 0 X -->
**Switching TLSA parameters from “`3` `1` `*`” to “`2` `0` `*`” while migrating to a new certificate chain.**
Specifically, the RFC explains a relatively complicated change with the following example.
A given DANE identity _currently_ uses

* a self-signed server certificate,
* a “`3` `1` `1`” TLSA record, and
* a specific public key

_After the change_, the DANE identity would use

* a new certificate chain,
* a “`2` `0` `1`” TLSA record, and
* a new public key

The RFC mandates that the change should involve a transitional “`3` `1` `1`” TLSA record, which needs to be populated into DNS caches, before deploying the new certificate chain and subsequently replacing the old TLSA RRs with the new, desired, “`2` `0` `1`” TLSA record.
If a new “`2` `0` `1`” TLSA record is introduced right away, then the old “`3` `1` `1`” TLSA record might still persist in a DNS cache, which would cause the “`2` `0` `1`” TLSA record to correspond to the _new_ certificate chain, whereas the “`3` `1` `1`” TLSA record would correspond to the _old_ public key.
This would violate the requirement of every TLSA parameter combination group to be sufficient to validate the TLS server, as discussed in [section @sec:protocols-tlsa-requirements].

The reason why the transitional TLSA record must have the parameter combination “`3` `1` `1`” is that clients might want to negotiate the exchange of raw public keys, based solely on the _presence_ of a “`3` `1` `1`” TLSA record.
This is not possible with a certificate corresponding to a “`2` `0` `1`” TLSA record.

Therefore, these are the steps for switching from the existing certificate _certOld_, the usage `3`, and the selector `1` to the new certificate _certNew_, the usage `2`, and the selector `0`:

1. Let _tlsaRecordsOld_ be the list of all `TLSA` RRs that correspond to _certOld_, have usage `3`, and selector `1`.
2. Let _tlsaRecordsTransitional_ be an empty list.
3. For each _tlsaRecordOld_ of _tlsaRecordsOld_,
   1. let _m_ be the matching type of _tlsaRecordOld_,
   2. let _tlsaRecordTransitional_ be a new `TLSA` RR generated from _certNew_ with the desired parameter combination « `3`, `1`, _m_ »,
   3. append _tlsaRecordTransitional_ to _tlsaRecordsTransitional_,
   4. publish _tlsaRecordTransitional_.
4. Wait at least 2× TTL.
5. For each _tlsaRecordTransitional_ of _tlsaRecordsTransitional_,
   1. assert that _tlsaRecordTransitional_ is now populated in DNS caches.
6. Publish _certNew_.
7. Assert that TLS server now serves _certNew_.
8. For each _tlsaRecordOld_ of _tlsaRecordsOld_,
   1. delete _tlsaRecordOld_.
9. For each _tlsaRecordTransitional_ of _tlsaRecordsTransitional_,
   1. let _m_ be the matching type of _tlsaRecordTransitional_,
   2. delete _tlsaRecordTransitional_,
   3. let _tlsaRecordNew_ be a new `TLSA` RR generated from _certNew_ with the desired parameter combination « `2`, `0`, _m_ »,
   4. publish _tlsaRecordNew_.

<!-- New matching type -->
**Introducing a new matching type.**
[Section @sec:protocols-tlsa-requirements] mentions the requirement to only publish TLSA records with matching type `2` if a TLSA record with matching type `1` and the same usage and selector exists as well.
There is also a requirement about _algorithm agility_ which states that if a TLSA record with matching type `2` and a specific usage and selector exists, then TLSA records must also exist _for all other_ usages and selectors, with matching type `2`.
A matching type must either entirely exist across all TLSA records or entirely _not_ exist across all TLSA records.
It cannot be used for _some but not all_ TLSA records.

Therefore, enabling the new matching type _m_, these steps need to be performed:

1. Let _certs_ be the list of all currently used certificates.
2. For each _cert_ of _certs_,
   1. Let _tlsaRecords_ be the list of all `TLSA` RRs that correspond to _cert_ and do not have matching type _m_.
   2. For each _tlsaRecord_ of _tlsaRecords_,
      1. let _u_ be the usage of _tlsaRecord_,
      2. let _s_ be the selector of _tlsaRecord_,
      3. let _tlsaRecordNew_ be a new `TLSA` RR generated from _cert_ with the desired parameter combination « _u_, _s_, _m_ »,
      4. publish _tlsaRecordNew_.

<!-- Remove matching type -->
**Removing a specific matching type.**
This applies the inverse of the same rule as stated above.

To disable the matching type _m_, these steps need to be performed:

1. Let _tlsaRecords_ be the list of all `TLSA` RRs that have matching type _m_.
2. For each _tlsaRecord_ of _tlsaRecords_,
   1. delete _tlsaRecord_.

<!-- Waiting during initial activation -->
Note that, in general, the step _“Wait at least 2× TTL”_ is not needed during _initial_ activation.

<!-- Identity adjustment or maintenance? -->
Now, the question is: are these _identity adjustments_ or _maintenance_?
First, the step _“Wait at least 2× TTL”_ might suggest that something is happening which makes the current certificate association temporarily unusable.
But every process is carefully crafted to _avoid_ this; otherwise, it might cause a temporary outage of the Web service.
The waiting does not actually impact the identity.
Certificate expiration might cause an outage, though not with usage `3` (`DANE-EE`), and certificate revocation might play a role.
However, revocation does not imply immediate invalidity of the certificate.
Server administrators still have some time to replace a revoked certificate, and they have the option to supply a replacement in advance—all this can also be automated.
And expiration of the current certificate should never realistically happen if renewal is automated, because it is good practice to renew the certificate several weeks before its expiration [@leFAQ].
The only scenario in which a certificate becomes unusable, with no replacement issued, is if the identity management application is unavailable for several weeks.
Such a disastrously exceptional scenario is where the life cycle and identity management model breaks down, regardless.

<!-- Identity adjustment must not happen -->
None of the transitions described above justify temporarily deactivating the DANE identity, even if all of them require changes in the identity attributes.
One change that might be considered identity adjustment, and therefore justify deactivation, would be the manual full replacement of a certificate, e.g. due to a change in the identifying information embedded in the certificate.
Since removing obsolete TLSA records is recommended by RFC 7671, any identity adjustment would start with this.
But since TLSA records are kept in DNS caches for a while, they will correspond to the certificate that is about to be removed.
Since this would cause DANE authentication to fail, this case is not allowed to happen.
Instead, the manually replaced certificate should only be _scheduled_ to be deployed; then the identity manager will simply follow the same steps as described in _Server key rollover or certificate change with fixed TLSA parameters_.
It is possible that no _identity adjustment_ ever needs to occur.

<!-- Other DANE bindings -->
Other DANE bindings—IPSECKEY, SSHFP, OPENPGPKEY, and SMIMEA—would have their own identities, i.e. “tracks” that keep track of the _currently used_ DNS records and certificate or public key, but this thesis only focuses on the TLSA binding for now.

# User-centered design implementation using web technologies {#sec:implementation}

<!-- Intro -->
This section includes a few concrete ideas for an implementation of the identity manager to be established.
The purpose of the identity manager is to automate the TLSA publishing process and help avoid user errors.
A user-centered design approach shall provide an intuitive workflow in which the user is guided through the process.
We will look at a few helpful tools and briefly discuss the back end and the front end design.

## Tools {#sec:implementation-tools}

<!-- Intro -->
There are a few helpful cryptography- and network-related tools that can be used in an identity manager.

### DNS look-up utilities

<!-- host and dig intro -->
`host` [@host] and `dig` [@dig] are two DNS look-up utilities.
`host` is a simple utility for quick look-ups whereas `dig` is a more flexible tool useful in debugging.

<!-- host -->
A simple command like this reveals the IP addresses associated with a domain name:

```sh
host 'www.example.com.'
```

```none
www.example.com has address 93.184.216.34
www.example.com has IPv6 address 2606:2800:220:1:248:1893:25c8:1946
```

<!-- ANY -->
With one of the following commands, it is possible to look up all RRs of the `example.com` DNS zone from the name server configured on the system:

```sh
host -a 'example.com.'
dig 'example.com.' 'ANY'
```

<!-- AA -->
A different name server can be specified; the one in the `NS` RR of `example.com` (`a.iana-servers.net`) is the authority for that domain name and should provide an _authoritative answer_ (with the `aa` bit set, as specified in RFC 1035 [@rfc1035]).

```sh
host 'example.com.' 'a.iana-servers.net.'
dig '@a.iana-servers.net.' 'example.com.'
```

<!-- Parent zones -->
The parent zone is simply `com.`, the root zone is `.` (the dot).
Both of these can easily be queried; the `-t` flag can be used to query RRs of a specific type:

```sh
host -a 'com.'        # Welcome to the `com.` zone!
host -t 'DNSKEY' '.'  # Public keys of the root zone;
                      #   the one with flag 256 is the ZSK
                      #   the one with flag 257 is the KSK.
```

<!-- TLSA -->
The `www` name is commonly used for the web service of a domain.
In order to reach the TLSA RR of the web service, the protocol and port need to be specified as well, forming the complete DNS name of `_443._tcp.www.example.com`.
A lookup of these RRs would be performed with one of these commands:

```sh
host -t 'TLSA' '_443._tcp.www.example.com'
dig '_443._tcp.www.example.com' 'TLSA'
```

Alternatively, the domain name can be dynamically constructed:

```sh
port='443'
protocol='tcp'
domain='www.example.com.'

host -t 'TLSA' "_${port}._${protocol}.${domain}"
```

<!-- Example -->
The real `example.com` does not currently support DANE; in this case we get a response involving `status: NXDOMAIN` and a single `SOA` RR, indicating that the name `_443._tcp.www.example.com` does not exist.
This is the example output for `www.huque.com`[^whc]:

  [^whc]: The “`ANSWER SECTION`” contains eight long lines which have been wrapped to fit into the page.

```sh
dig '_443._tcp.www.huque.com' 'TLSA'
```

```none

; <<>> DiG 9.18.13 <<>> _443._tcp.www.huque.com TLSA
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59501
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;_443._tcp.www.huque.com.	IN	TLSA

;; ANSWER SECTION:
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  7634E0EED7DFCA738733ED09C8BED00D4C5F4A3E15F76BC9E0B6E57F DBB9163D
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  6C85CC093C31221CBFF9E61CFF5E9CA14BFEB0F9BBC341A769529027 5D813CF4
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  CD98C1661E982DA43E414E9EEB7A3A545DEEC8859F5282E28CA3CBB3 82DDBE3E
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  33F5B0C36FBDE25D8B148623DEB22A4CD5D1A806EE1259F48DD9ABDF 36EE8A63
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  DE4369CF0866A1E7626D73DB36DBFC4B74097C3C70489A2D3351B6E7 5E99583A
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  A67924AFCD895B9661C4C5D67A83215F60B7D0E1DA8A30B67EEC6BD6 23A5B57C
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  244B484761B1CC2A390DB3F1031C1D512AF3C9C14DF47854E43E3C29 17EF671C
_443._tcp.www.huque.com. 7169	IN	TLSA	3 1 1
  EE7041856CC8AD23789D39462A7D11C0B7633C6142ADE119BC9B1BEF 708F7D0A

;; Query time: 16 msec
;; SERVER: 192.168.178.1#53(192.168.178.1) (UDP)
;; WHEN: Mon Apr 17 16:08:24 CEST 2023
;; MSG SIZE  rcvd: 428
```

### TLS look-up utility

<!-- nmap -->
`nmap` [@nmap] is a network and security utility.
The following command can extract the TLS certificate currently used by a host [@sf881415]:

```sh
nmap -vv -p '443' --script 'ssl-cert' 'www.huque.com'
```

### Cryptography, certificate management {#sec:implementation-tools-openssl}

<!-- openssl, nmap -->
OpenSSL [@openssl] and Certbot [@certbot] are two very versatile tools.

<!-- openssl PKI -->
The [appendix](#lst:openssl) shows how one can use the subcommands `openssl genpkey`, `openssl rsa`, `openssl pkeyutl`, `openssl req`, and `openssl verify` in order to create and verify certificates.

<!-- openssl client, DANE -->
`openssl s_client` provides an SSL/TLS client, which even provides DANE authentication using the `-dane_tlsa_domain` and `-dane_tlsa_rrdata` options.
This can be used to verify correct deployment of TLS certificates on the server and correct matching of TLSA records.

<!-- openssl X.509 -->
Finally, `openssl x509` and `openssl enc` can be used to extract public keys and transcode PEM into DER format, respectively.
Paired with basic tools such as `sha256sum` or `sha512sum`, `cut`, and `tr`, one can generate a `TLSA` RR as described in [section @sec:protocols-tlsa-generating].

```sh
openssl 'x509' -in 'huque.pem' -pubkey -nocert -noout |
  openssl 'enc' -d -base64 -out 'huque-public.der' &&
  sha256sum 'huque-public.der' |
  cut --delimiter=' ' --fields='1' |
  tr '[:lower:]' '[:upper:]'
```

## Back end {#sec:implementation-backend}

<!-- Concept -->
Conceptionally, a single system is required that has access to both DNS zone management as well as TLS certificate management for the same domain.
These two management “subsystems” might either be integrated into the same code base or communicate via an API.
The DNS management subsystem would have all the normal functionality of existing DNS managers, i.e. adding, removing, editing records, etc.
The TLS management subsystem would facilitate the automatic issuing and renewal of TLS certificates, as well as handle revocations—this also exists.
The crucial exception for the DNS manager is to _not allow_ direct manipulation of `TLSA`, `IPSECKEY`, `SSHFP`, `OPENPGPKEY`, and `SMIMEA` records; this is the job of the DANE identity manager.

<!-- Communication -->
For example, if the TLS manager wants to renew a certificate, it would first notify the identity manager, which would in turn notify the DNS management subsystem.
A new `TLSA` RR will be created; the DNS manager needs to be notified about the request to publish it.
The TLS manager needs to wait for its signal to publish the certificate.
[Figure @fig:renew-tls-update-tlsa] presents this rough idea visually.

::: {.figure #fig:renew-tls-update-tlsa}
```{.mermaid format=pdf theme=neutral background=transparent}
sequenceDiagram
  participant DNS as DNS mgmt.
  participant ID as ID mgmt.
  participant TLS as TLS cert mgmt.
  TLS->>ID: Notify cert is about to expire
  activate ID
  ID->>+TLS: Request renewal
  deactivate ID
  Note over TLS: Generate new TLS cert
  TLS->>-ID: New certificate
  activate ID
  ID->>+DNS: Request new TLSA RR
  deactivate ID
  Note over DNS: Publish new TLSA RR
  DNS->>-ID: Success
  Note over DNS: Wait 2× TTL
  DNS->>ID: Global DNS caches expired
  activate ID
  ID->>+TLS: Request publishing TLS cert
  deactivate ID
  Note over TLS: Publish new TLS cert
  TLS->>-ID: Success
```

Example communication sequence of publishing `TLSA` RRs while renewing TLS certificate
:::

Since a certificate can be issued for a wildcard domain (e.g. `*.example.com`), it is useful to explicitly enumerate the DNS names the certificate applies to, so that `TLSA` RR names can be more easily identified.

<!-- Logging -->
A DANE identity manager should log every step of any DNS or TLS management operation, which is an optional requirement from the ISO/IEC 24760-1 standard [@ISO24760-1].
This might aid in auditing and fault recovery.
In case the identity manager itself suffers an outage, as described in [section @sec:standards-idm-transitions], a few steps could be taken to mitigate the lack of _management_ after it becomes available again.
For example, the identity manager could audit all DANE identities and their current states, and

* validate their integrity,
* renew certificates which are revoked, expired, or about to expire,
* test if they are usable, i.e.
  * check if the certificate is being correctly served from the server and
  * check if the DANE records are deployed in the DNS,
* act as a DANE client and perform an authentication.

These checks could also occur periodically or before and after critical operations.

<!-- Migration -->
There should also be a way for the identity manager to accept an _existing_ identity state; for example, if  `TLSA` records already exist in a zone, they could be imported to the new identity manager, or they could be exported to some other identity manager.
Ideally, this should also be possible during an outage.

<!-- Future-proofing -->
Some steps can be taken to _future-proof_ TLSA identity management, so that changes or extensions in the DANE TLSA standards can be easily adapted to in the future:

* Matching type: Algorithm agility (also, crypto agility) is recommended [@rfc7671].
  Currently, only SHA2-256 and SHA2-512 are supported, but common digest algorithms used in related cryptography protocols can be prepared, even if they’re unlikely to ever be included in the TLSA standard.
  After all, digest algorithms being discovered to be “broken” is not unprecedented.
* Selector: Unlikely to ever change, but an X.509 parser and DER encoder (i.e. OpenSSL) should be available to a TLSA publisher to extract various fields from a certificate, just in case.
* Usage: Seems to be the most versatile since it already encodes _two_ pieces of information in a single field (2 bits).
  It’s plausible that extensions to PKIX or entirely new PKI approaches force a rethinking of how authentication needs to be performed.
  The third bit, fourth bit, and so on, could be used for other decisions—perhaps unforeseeable today.
  RFC 6698 states [@rfc6698]:
  
  > The certificate usages defined in this document explicitly only apply to PKIX-formatted certificates in DER encoding (X.690).
  > If TLS allows other formats later, or if extensions to this RRtype are made that accept other formats for certificates, those certificates will need their own certificate usage values.

* A new fourth field: unlikely to be backward-compatible.
  It’s more likely that anything _entirely new_ will be “crammed” into the Usage field, i.e. the Usage field being repurposed for new modes of TLSA authentication.
  Alternatively, a new record type will be established and `TLSA` deprecated.

## Web application and user interface {#sec:implementation-app}

<!-- Intro -->
Since the term “identity” is rather complex, it should be a concept that is only handled by the back end; the user should not be confronted with this concept.
DNS zone management and certificate management can be regarded as two “subsystems” of DANE identity management; these are more likely to be familiar to DNS administrators and server administrators.
Therefore, the user interface would involve these two subsystems.

<!-- deSEC -->
Here, the layout used by the deSEC prototype might be useful: one tab for DNS management, another for Identity management [@deSEC].
Since the other four DANE bindings should be considered as well, the second tab should not just be labeled “TLS management”—instead, “Identity management” is a suitable alternative.
This tab would then have the five subtabs corresponding to each DANE binding.
This reflects the two components that constitute the system: _DNS management_ and _key or certificate management_.

<!-- DNS manager -->
The DNS manager would be a simple editable table of Resource Records.
DANE-specific resource records would be marked specially, and would only be able to be edited _indirectly_ via the identity manager.

<!-- Identity manager, table -->
The key or certificate manager is the more interesting component.
The “TLS” subtab would show a table, each row corresponding to an identity, i.e. a “track” of certificates.
The columns would correspond to the identity attributes from [section @sec:standards-terminology-identity]:

* **Label**: human-readable identifier
* **Hostnames**: list of hostnames that the current certificate applies to
* **Valid from**
* **Valid until**
* **Type**: List of usage–selector combinations for generating the `TLSA` RRs
* **Status**: deployment state of the current certificate and the `TLSA` RRs

<!-- Identity manager, DANE policies -->
The _list_ referred to in the “Type” column could also be empty.
This would put the identity in the “No DANE” state.

<!-- Identity manager, status -->
The “Status” could be one of the following:

* **Inactive**: user will be prompted to configure and enable once; guide them (corresponds to “Established”)
* **No DANE**: if no DANE records will be deployed for the time being
* **Scheduled**: if new certificate is ready to be deployed, but the `TLSA` requires the necessary waiting period (corresponds to “Established” or “Active”, depending on whether `TLSA` RRs already have deployed)
* **Deployed**: happy state (corresponds to “Active”)
* **Revoked**: refers to the _identity_ being revoked (corresponds to “Archived”)
* **Error**: user will be prompted for action and informed via e-mail

<!-- Identity manager, attributes -->
Clicking on one of the identity rows displays all the above pieces of data and additionally:

* A stored certificate signing request (CSR)
* Authentication token for a CA
* The _current_ `TLSA` RRs that correspond to the certificate
* References to future or past certificates during a rollover

<!-- Identity manager, matching type -->
The matching type will not be a per-identity option, but a globally configurable list of enabled hashing algorithms; of course, SHA2-256 can never be disabled.

<!-- Identity manager, create identity, suggestions -->
The key or certificate manager would allow adding new identities of the desired DANE binding in the corresponding subtab.
For example, the user clicks on the “Identity manager” tab, then navigates to the “TLS / TLSA” subtab.
To create a new identity, the user would click on an “Add” button, upload a certificate via another button, or via drag & drop, or via some other process to generate certificates on the fly.
A dialog will prompt the user for the required identity attributes.
The identity manager will then suggest the TLSA record policy based on the uploaded certificate; the result is a _list_ of suggestions

* If the certificate is an end-entity certificate, suggest only usage `DANE-EE`, selector `SPKI`, and matching type `SHA2-256`
* Else, if the certificate is an end-entity certificate, suggest only usage `DANE-TA`, selector `Cert`, and matching type `SHA2-256`
* If `SHA2-512` is enabled globally, add these suggestions to the list.

The user should still be able to add and remove other options.
Later, when editing the identity, this list can always be adjusted, including removing all items.
The identity manager should always warn the user in case they select an option that is not recommended, as per [section @sec:protocols-tlsa-usages].
Editing the list of usages and selectors, as well as the global list of matching types, invokes the algorithms for state transitions of TLSA entries, described in [section @sec:standards-idm-transitions].

<!-- Identity manager, create identity, hostnames -->
The hostnames can be automatically extracted from the certificate.

* If the certificate is issued for `a.example.com` and `b.example.com`, create TLSA entries at `_443._tcp.a.example.com` and `_443._tcp.b.example.com`.
  Alternatively, coalesce them into a single TLSA record with a name including `_dane`, using indirection via CNAME, as per [section @sec:protocols-tlsa-rr]—perhaps using the human-readable label as well.
* If the certificate is issued for `*.example.com`, create corresponding TLSA entries for every DNS name that is _currently in use_, keeping track of future updates.

<!-- Identity manager, create identity, actions -->
When viewing one specific identity, the user will have options for different actions:

* **Remove**: terminates the identity, archives it, deletes all corresponding `TLSA` RRs.
* **Replace**: schedules new certificate to replace the current one with, as soon as the `TLSA` RR update allows.

In the [appendix](#apdx:wireframes), some of these ideas are visualized.

# Conclusion and future work {#sec:conclusion}

<!-- DANE summary -->
DANE provides an opportunity for a better trust model—if implemented correctly.
Time will tell if it really turns out to be a better option than the current public-key infrastructure model for TLS.

<!-- Thesis summary -->
We show that management can be simplified for DNS zone administrators.
The prototyped application follows the protocol specification as well as identity management standards.
The proposed ideas for an intuitive user interface and automation in the back end should contribute greatly to decreasing human error and thus be beneficial for security.

<!-- DANE future -->
In [section @sec:background-dnssec] and [section @sec:background-dane], a lot of doubt about DNSSEC and DANE can be seen by protocol implementers and client users.
Since a significant portion of this doubt stems from the high complexity involved in managing DNSSEC and DANE, hopefully, this thesis can serve as a useful starting point for future implementation efforts.
[Section @sec:implementation], especially, proposes how a possible integration into a real DNS management system might look like, simplifying implementation efforts for domain registrars, and, in turn, streamlining the process for DNS administrators, possibly leading to higher adoption rates in the near future.
The chicken-and-egg problem might start getting solved using this research.

<!-- Thesis future -->
However, implementers will still need to review the referenced standards and protocol specifications carefully.
Certain considerations, unfortunately, did not fit into the scope of this thesis; it is recommended that future research keeps these in mind.
For example:

* More complex hosting setups, e.g. DANE used with delegated subzones (e.g. RFC 7671)
* _A framework for identity management—Part 2: Reference architecture and requirements_ (ISO/IEC 24760-2)
* General information security guidelines (e.g. ISO 27001)
* DANE integration with `SRV` RRs (RFC 7673)
* Usability guidelines (e.g. ISO 9241, ISO/IEC 25066)
* Accessibility guidelines (e.g. WAI-ARIA)
* User testing
* User documentation
* Developer documentation
* Atomicity
* Interoperability
* Internationalization

Further research into possible roadblocks during implementation is desired, as well as an evaluation in reference to this thesis.
In tandem, registrar support for DANE and deployment of DANE in real domains need to be continually measured—both in quantity and quality.

# References {-}

::: {#refs}
:::

# Appendix {-}

## Code samples {- #lst:openssl}

A small example using TLS certificate PKI with OpenSSL, written in Bash.

```sh
#!/bin/bash

mkdir -- 'CA' 'Alice' 'Bob'

# Step 1: Make certificates

(
  cd -- 'CA'
  openssl 'genpkey' -algorithm 'RSA' -pkeyopt 'rsa_keygen_bits:2048' \
    -out 'private.pem'
  openssl 'pkey' -in 'private.pem' -pubout -out 'public.pem'
  openssl 'req' -x509 -noenc -key 'private.pem' -sha256 -days '1024' \
    -out 'root.pem'
)

(
  cd -- 'Alice'
  openssl 'genpkey' -algorithm 'RSA' -pkeyopt 'rsa_keygen_bits:2048' \
    -out 'private.pem'
  openssl 'pkey' -in 'private.pem' -pubout -out 'public.pem'
  openssl 'req' -new -key 'private.pem' -out 'request.csr'
)

(
  cd -- 'Bob'
  openssl 'genpkey' -algorithm 'RSA' -pkeyopt 'rsa_keygen_bits:2048' \
    -out 'private.pem'
  openssl 'pkey' -in 'private.pem' -pubout -out 'public.pem'
  openssl 'req' -new -key 'private.pem' -out 'request.csr'
)

openssl 'x509' -req -in 'Alice/request.csr' -CA 'CA/root.pem' \
  -CAkey 'CA/private.pem' -CAcreateserial -out 'Alice/certificate.pem' \
  -days '512' -sha256
openssl 'x509' -req -in 'Bob/request.csr' -CA 'CA/root.pem' \
  -CAkey 'CA/private.pem' -CAcreateserial -out 'Bob/certificate.pem' \
  -days '512' -sha256

# Step 2: Verify certificates, get SPKI

(
  cd -- 'Alice'
  openssl 'verify' -CAfile '../CA/root.pem' '../Bob/certificate.pem'
  openssl 'x509' -pubkey -in '../Bob/certificate.pem' -noout > 'public-Bob.pem'
)

(
  cd -- 'Bob'
  openssl 'verify' -CAfile '../CA/root.pem' '../Alice/certificate.pem'
  openssl 'x509' -pubkey -in '../Alice/certificate.pem' \
    -noout > 'public-Alice.pem'
)

# Step 3: Alice picks a symmetric key, encrypts it using Bob’s public key,
#         hashes it and creates a signature using her private key

(
  cd -- 'Alice'
  openssl 'rand' -base64 -out 'symmetric.pem' '32'
  openssl 'pkeyutl' -encrypt -in 'symmetric.pem' -pubin \
    -inkey 'public-Bob.pem' -out 'symmetric.enc'
  openssl 'dgst' -sha1 -sign 'private.pem' -out 'signature.bin' 'symmetric.pem'
)

# Step 4: Bob receives encrypted symmetric key and signature from Alice

(
  cd -- 'Bob'
  openssl 'pkeyutl' -decrypt -in '../Alice/symmetric.enc' \
    -inkey 'private.pem' -out 'symmetric.pem'
  openssl 'dgst' -sha1 -verify 'public-Alice.pem' \
    -signature '../Alice/signature.bin' 'symmetric.pem'
)


# Step 5: Alice encrypts data using the symmetric key,
#         Bob receives and decrypts it

(
  cd -- 'Alice'
  openssl 'enc' -aes-256-cbc -pbkdf2 -pass 'file:symmetric.pem' -p -md \
    'sha256' -in 'plaintext.txt' -out 'ciphertext.bin'
)

(
  cd -- 'Bob'
  openssl 'enc' -aes-256-cbc -pbkdf2 -d -pass 'file:symmetric.pem' -p \
    -md 'sha256' -in '../Alice/ciphertext.bin' -out 'plaintext.txt'
)
```

\newpage

## Design wireframes {- #apdx:wireframes}

![](Design%201.pdf)

Clicking on the pencil icon in `TLSA` RR rows in _DNS management_ or on the pencil icon in _Identity management_ will bring up the dialog _TLS identity (DANE TLSA)_.
Clicking on the _Settings_ button will bring up the _Global DANE TLSA settings_ dialog.

![](Design%202.pdf)
