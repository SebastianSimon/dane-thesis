## Idea

An identity is identified by a name.
That “name” is sometimes synonymized with “hostname” or “reference identifier”.
This name is included in a TLS certificate.
The subject, email, organization name, canonical name might also identify the identity.
However, there should be a definition that considers the case where these pieces of data do not _uniquely_ identify a server — either coincidentally, mistakenly, or maliciously.
Asking a client to verify a server’s identity essentially means verifying that a target server is “the right one” and not an imposter.

The proper way to think about what an identity is is probably the following scenario:
You have an identifier.
You are told that this identifier points to a service that you are interested in.
You follow the signposts and you arrive at one unique destination.
However, at this point, you cannot be confident that this destination is the correct one.
This is where identity-proofing and authentication mechanisms would start being involved.

But in an abstract sense, an identity, in this context, is the destination you arrived at by following all the signs that correlated with a given identifier.
This needn’t even be about internet protocols.
Consider the identifier `Sherlock Holmes, 221B Baker Street, London, United Kingdom`.
First, you book a flight to London, then you look for street signs that read “Baker Street”, then you walk through that street until you find a house with the number “221B”.
That house _matches_ the identity you’ve been looking for, unless someone replaced some street signs or your pilot landed in the wrong city without telling anyone.

Careful: some resources might confuse the specific term relevant for this thesis with a more general (e.g. dictionary definition) term of “identity”.
For example, the digital identity of a server is not the same as the personal identity of a server administrator who requests signing a certificate for the server.

An identity is an entity that is known by an identifier.
An identity can be authenticated; this is exactly what Authenticity is for:
Communication with some entity is authenticated iff you have managed to prove that the entity that performs some action such as sending a message truly is the entity that you expect it to be — in other words: you’re trying to verify the entity’s identity.
Specifically, the commonality in the usages of the term “identity” from the articles below is:

* An identity is a set of descriptions matching an entity (user or object) which identifies said entity.
* Identification is indication of an identity.
* An identifier is an external description, meaning that the entity can be reached by it from outside.
* The set of descriptions might come in the compact form of an identifier which uniquely identifies an entity.
* Authentication is the process of verifying an identity.
* In an Internet context,
  * an identity is a “website” (web service, server, etc.),
  * an identifier is a hostname,
  * authentication happens via a public key infrastructure,
  * a certificate contains an identity and a public key, and
  * a certificate _binds_ an identity and a public key _to each other_.

RFC 4949 provides some intuitive description of what an identity is and what it’s used for.
RFC 6125 provides important concepts and guidelines about authentication.

Some related terms are:

* From RFC 4949:
  * Singular identity
  * Shared identity
  * Group identity
* From RFC 6125:
  * (Client’s) Reference identity (of server) (confer RFC 7671: “reference identifiers”)
  * (Server’s) Presented identity (of server)

RFC 6698 literally describes certificates as “the binding of an identity to a key”.

## Definition of identity

[RFC 4949](https://www.rfc-editor.org/rfc/rfc4949#section-4):

> The collective aspect of a set of attribute values (i.e., a set of characteristics) by which a system user or other system entity is recognizable or known.

### Related (RFC 4949)

Identifier:

> A data object — often, a printable, non-blank character string — that definitively represents a specific identity of a system entity, distinguishing that identity from all others.

Authenticate:

> Verify (i.e., establish the truth of) an attribute value claimed by or for a system entity or system resource.

Registration:

> A system process that
> * initializes an identity (of a system entity) in the system,
> * establishes an identifier for that identity,
> * may associate authentication information with that identifier, and
> * may issue an identifier credential (depending on the type of authentication mechanism being used).
> 
> Tutorial: Registration may be accomplished either directly, by the CA, or indirectly, by a separate RA.
> An entity is presented to the CA or RA, and the authority either records the name(s) claimed for the entity or assigns the entity's name(s).
> The authority also determines and records other attributes of the entity that are to be bound in a certificate (such as a public key or authorizations) or maintained in the authority's database (such as street address and telephone number).
> The authority is responsible, possibly assisted by an RA, for verifying the entity's identity and vetting the other attributes, in accordance with the CA's CPS.

### Usage

> An IDOC MAY apply this term to either a single entity or a set of entities.
> If an IDOC involves both meanings, the IDOC SHOULD use the following terms and definitions to avoid ambiguity:
> 
> * "Singular identity": An identity that is registered for an entity that is one person or one process.
> * "Shared identity": An identity that is registered for an entity that is a set of singular entities
>   * in which each member is authorized to assume the identity individually and
>   * for which the registering system maintains a record of the singular entities that comprise the set.
>   In this case, we would expect each member entity to be registered with a singular identity before becoming associated with the shared identity.
> * "Group identity": An identity that is registered for an entity
>   * that is a set of entities
>   * for which the registering system does not maintain a record of singular entities that comprise the set.

### Tutorial

> When security services are based on identities, two properties are desirable for the set of attributes used to define identities:
> 
> * The set should be sufficient to distinguish each entity from all other entities, i.e., to represent each entity uniquely.
> * The set should be sufficient to distinguish each identity from any other identities of the same entity.
> 
> The second property is needed if a system permits an entity to register two or more concurrent identities.
> Having two or more identities for the same entity implies that the entity has two separate justifications for registration.
> In that case, the set of attributes used for identities must be sufficient to represent multiple identities for a single entity.
> 
> Having two or more identities registered for the same entity is different from concurrently associating two different identifiers with the same identity, and also is different from a single identity concurrently accessing the system in two different roles.

## Other research

<https://www.rfc-editor.org/rfc/rfc3207>

> Further, there is often a desire for two SMTP agents to be able to authenticate each others' **identities**.
> For example, a secure SMTP server might only allow communications from other SMTP agents it knows, or it might act differently for messages received from an agent it knows than from one it doesn't know.

<https://www.rfc-editor.org/rfc/rfc7817>

TLS – E-mail.
Solves a similar problem to the one DANE solves, but isn’t DANE.
Links to RFCs 2595, 3207, and 5804 which talk about server identities.

> During a TLS negotiation, an email client (i.e., an SMTP, IMAP, POP3, or ManageSieve client) MUST check its understanding of the **server identity (client's reference identifiers)** against the server's **identity** as presented in the server Certificate message in order to prevent man-in-the-middle attacks.

This is what RFC 6125 says, but in a more complicated way.

> The following inputs are used by the verification procedure used in [RFC6125]: 1. For DNS-ID and CN-ID identifier types, the client MUST use one or more of the following as "reference identifiers": (a) the domain portion of the user's email address, (b) the hostname it used to open the connection (without CNAME canonicalization). The client MAY also use (c) a value securely derived from (a) or (b), such as using "secure" DNSSEC [RFC4033] [RFC4034] [RFC4035] validated lookup.

> [RFC6186] provides an easy way for organizations to autoconfigure email clients. It also allows for delegation of email services to an email hosting provider. When connecting to such delegated hosting service, an email client that attempts to verify TLS server identity needs to know that if it connects to "imap.hosting.example.net", such server is authorized to provide email access for an email such as alice@example.org.

<https://tesmail.org/dns-based-authentication-of-named-entities-dane/>

> We have seen that STARTTLS leaves email servers open to man-in-the-middle attacks because of two shortcomings:
> 
> * […]
> * It does not provide a way to authenticate the destination server’s **identity** (hostname).

<https://www.security-insider.de/was-ist-dane-a-860135/>

> Typische Probleme mit CAs und Zertifikaten sind:
> 
> * unzureichende Prüfung der **Identität** von Zertifikatsantragstellern durch die CA

<https://www.infoblox.com/dns-security-resource-center/dns-security-faq/what-is-dane/>

> TLSA is not limited to only verifying website **identities**, it can also be used to verify other services such as mail. Figure FAQ-4 shows an example of a TLSA record that provides information about an IMAP server:
> 
> […]
> 
> The same principle applies here, if the user is using supported email software when it connects to the IMAP server, it double checks the **identity** of the IMAP server received over port 993 against the information it received over DNSSEC on port 53. If the two do not match, the email client does not proceed to send the email.

<https://www.rfc-editor.org/rfc/rfc7671/>

> In this section, the meaning of DANE-EE(3) is updated from [RFC6698] to specify that peer **identity** matching and validity period enforcement are based solely on the TLSA RRset properties.
> […]
> 
> Authentication via certificate usage DANE-EE(3) TLSA records involves simply checking that the server's leaf certificate matches the TLSA record.
> In particular, the binding of the server public key to its name is based entirely on the TLSA record association.
> The server MUST be considered authenticated even if none of the names in the certificate match the client's reference **identity** for the server.
> This simplifies the operation of servers that host multiple Customer Domains, as a single certificate can be associated with multiple domains without having to match each of the corresponding reference identifiers.

> [RFC6125] Saint-Andre, P. and J. Hodges, "Representation and Verification of Domain-Based Application Service **Identity** within Internet Public Key Infrastructure Using X.509 (PKIX) Certificates in the Context of Transport Layer Security (TLS)", RFC 6125, DOI 10.17487/RFC6125, March 2011, <http://www.rfc-editor.org/info/rfc6125>.

<https://www.rfc-editor.org/rfc/rfc6698>

> Some specifications for applications that run over TLS, such as [RFC2818] for HTTP, require that the server's certificate have a domain name that matches the host name expected by the client.
> Some specifications, such as [RFC6125], detail how to match the **identity** given in a PKIX certificate with those expected by the user.

> DNSSEC forms certificates (the binding of an **identity** to a key) by combining a DNSKEY, DS, or DLV resource record with an associated RRSIG record.
> These records then form a signing chain extending from the client's trust anchors to the RR of interest.

<https://kb.synology.com/en-global/DSM/tutorial/How_to_set_up_DANE_for_MailPlus>

> DANE (DNS-based Authentication of Named Entities) is a secure option for mail transport.
> DANE uses the DNSSEC infrastructure and works with TLSA records to authenticate the identity of recipient servers.
> By establishing a secure channel between the sender and recipient, DANE prevents emails from being intercepted in transit or landing in the wrong hands.

> DANE verifies the identity of a receiving server based on its A, MX, and TLSA records that are signed with DNSSEC.
> Validation outcomes and corresponding states are shown in the following table […].

<https://www.ibm.com/docs/en/zos/2.4.0?topic=SSLTBW_2.4.0/com.ibm.tcp.ipsec.ipsec.help.doc/nss/NssImageServerPs.BG_LocalId.htm>

> Server identity
> 
> This required value specifies the identity of the NSS server.
> Communication between the NSS server and NSS client must be protected using AT-TLS.
> During the AT-TLS handshake, the network security server will provide a certificate that will be used to authenticate its identity.
> The NSS client will interrogate this certificate and verify that the identity in the certificate matches the NSS server identity specified.
> 
> You can specify the identity as an IP address, a fully qualified domain name, a user ID at a fully qualified domain name, or an X.500 distinguished name.
> The default is an IP address.

<https://developer.fastly.com/reference/vcl/variables/server/server-identity/>

> server.identity
> 
> […]
> Same as server.hostname but also explicitly includes the POP identifier as the last three characters (e.g., cache-jfk1034-JFK or cache-iad-kiad7000001-IAD).
> Newer generations of cache servers now include the POP identifier in server.hostname, rendering server.identity largely redundant.

<https://help.stonesoft.com/onlinehelp/StoneGate/SMC/6.5.0/GUID-83E3E098-8148-4B6B-9D87-2913C34EED06.html>

> TLS server identity determines how SMC servers or NGFW Engines verify the identity of the external servers with which they communicate.

> TLS Server Identity Field: Select the server identity type field to be used.
> 
> * DNS Name — Use the DNS name of the server.
> * IP Address — Use the IP address of the server.
> * Common Name — Use the common name (CN) of the server.
> * Distinguished Name — Use the distinguished name (DN) of the server.
> * SHA-1 — Use SHA (Secure Hash Algorithm) hash function 1.
> * SHA-256 — Use SHA (Secure Hash Algorithm) hash function 256.
> * SHA-512 — Use SHA (Secure Hash Algorithm) hash function 512.
> * MD5 — Use MD5 Message-Digest Algorithm.
> * Email — Use the email address associated with the server.
> * User Principal Name — Use the user principal name (UPN) of the server.

> The Import Certificate dialog box for fetching the value of the server identity field from a certificate.
> Note: You can fetch the value of the server identity field from a certificate only if the server identity field is Distinguished Name, SHA-1, SHA-256, SHA-512, or MD5.

> Server Identity Value: Specifies the value for the selected field type.

<https://www.ibm.com/docs/en/ibm-mq/7.5?topic=ssl-how-tls-provide-authentication-confidentiality-integrity>

> The exchange of digital certificates during the SSL or TLS handshake is part of the authentication process.
> […]
> The certificates required are as follows, where CA X issues the certificate to the SSL or TLS client, and CA Y issues the certificate to the SSL or TLS server:
> For server authentication only, the SSL or TLS server needs:
> 
> * The personal certificate issued to the server by CA Y
> * The server's private key
> 
> and the SSL or TLS client needs:
> 
> * The CA certificate for CA Y
> 
> If the SSL or TLS server requires client authentication, the server verifies the client's **identity** by verifying the client's digital certificate with the public key for the CA that issued the personal certificate to the client, in this case CA X.
> For both server and client authentication, the server needs:
> 
> * The personal certificate issued to the server by CA Y
> * The server's private key
> * The CA certificate for CA X
> 
> and the client needs:
> 
> * The personal certificate issued to the client by CA X
> * The client's private key
> * The CA certificate for CA Y
> 
> Both the SSL or TLS server and client might need other CA certificates to form a certificate chain to the root CA certificate.

## Terminology: [Digital identity](https://en.wikipedia.org/wiki/Digital_identity)

> A digital identity is information used by computer systems to represent an external agent – a person, organization, application, or device.
> Digital identities allow access to services provided with computers to be automated and make it possible for computers to mediate relationships.

This article appears to talk about the same thing.
“Digital identity” sounds like a more specific term for “identity” as used in this context.
However, the next paragraph, confusingly, talks about _social_ or _online_ identities, i.e. _personal identities_.

> A digital identity may also be referred to as a digital subject or digital entity and is the digital representation of a set of claims made by one party about itself or another person, group, thing or concept.

_“a set of claims”_ is an interesting description.

Unfortunately, this article is not of much use.
It’s not very focused and doesn’t seem relevant.

## Closely related: Authentication

<https://www.rfc-editor.org/rfc/rfc6125>

> Many application technologies enable secure communication between two entities by means of Internet Public Key Infrastructure Using X.509 (PKIX) certificates in the context of Transport Layer Security (TLS).
> This document specifies procedures for representing and verifying the **identity** of application services in such interactions.

> When a client communicates with an application service using Transport Layer Security [TLS] or Datagram Transport Layer Security [DTLS], it references some notion of the server's **identity** (e.g., "the website at example.com") while attempting to establish secure communication.
> Likewise, during TLS negotiation, the server presents its notion of the service's identity in the form of a public-key certificate that was issued by a certification authority (CA) in the context of the Internet Public Key Infrastructure using X.509 [PKIX].
> Informally, we can think of these **identities** as the client's "reference identity" and the server's "presented identity" (these rough ideas are defined more precisely later in this document through the concept of particular identifiers).
> In general, a client needs to verify that the server's presented identity matches its reference identity so it can authenticate the communication.

> B.2. HTTP (2000)

> In general, HTTP/TLS requests are generated by dereferencing a URI.
> As a consequence, the hostname for the server is known to the client.
> If the hostname is available, the client MUST check it against the server's **identity** as presented in the server's Certificate message, in order to prevent man-in-the-middle attacks.

> If a subjectAltName extension of type dNSName is present, that MUST be used as the **identity**.
> Otherwise, the (most specific) Common Name field in the Subject field of the certificate MUST be used.
> Although the use of the Common Name is existing practice, it is deprecated and Certification Authorities are encouraged to use the dNSName instead.

> If the hostname does not match the identity in the certificate, […]

<https://en.wikipedia.org/wiki/Authentication>

> Authentication […] is the act of proving an assertion, such as the identity of a computer system user.

Here, we have another word: “assertion”; confer with “claim” above.

> In contrast with identification, the act of indicating a person or thing's identity, authentication is the process of verifying that identity.

> Digital authentication
> 
> The term digital authentication, also known as electronic authentication or e-authentication, refers to a group of processes where the confidence for user identities is established and presented via electronic methods to an information system.
> The digital authentication process creates technical challenges because of the need to authenticate individuals or entities remotely over a network.
> The American National Institute of Standards and Technology (NIST) has created a generic model for digital authentication that describes the processes that are used to accomplish secure authentication:
> 
> * Enrollment – an individual applies to a credential service provider (CSP) to initiate the enrollment process.
>   After successfully proving the applicant's identity, the CSP allows the applicant to become a subscriber.
> * Authentication – After becoming a subscriber, the user receives an authenticator e.g., a token and credentials, such as a user name.
>   He or she is then permitted to perform online transactions within an authenticated session with a relying party, where they must provide proof that he or she possesses one or more authenticators.
> * Life-cycle maintenance – the CSP is charged with the task of maintaining the user's credential over the course of its lifetime, while the subscriber is responsible for maintaining his or her authenticator(s).
