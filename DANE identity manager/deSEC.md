## [deSEC Documentation](https://desec.rtfd.io)

### [Domain Management](https://desec.readthedocs.io/en/latest/dns/domains.html)

Remember that Punycode exists and see the [Security Considerations](https://datatracker.ietf.org/doc/html/rfc3492#section-8) (Unicode canonicalization plays key role).

Of each domain entry, the `keys` entry contains the DNSSEC public key information (`DNSKEY` and `DS` record contents).

From [Identifying the Responsible Domain for a DNS Name](https://desec.readthedocs.io/en/latest/dns/domains.html#identifying-the-responsible-domain-for-a-dns-name):

> One use case of this is when requesting TLS certificates using the DNS challenge mechanism, which requires placing a `TXT` record at a certain name within the responsible domain.

Perhaps, this [example](https://desec.readthedocs.io/en/latest/dns/domains.html#example) can be simplified?

### [Retrieving and Creating DNS Records](https://desec.readthedocs.io/en/latest/dns/rrsets.html)

One RRset has Resource Records of a given record type for a given name.
Each RRset is uniquely identified by its `subname` and its `type` (endpoint `https://desec.io/api/v1/domains/{name}/rrsets/{subname}/{type}/`).
The JSON file therefore has this type:

```
[
  {
    "created": String,
    "domain": String,
    "subname": String,
    "name": String,
    "type": String,
    "records": [
      String
    ]: RRs,
    "ttl": Number,
    "touched": String
  }: RRset
]: RRSets
```

> A broad range of record types is supported, with most DNSSEC-related types (and the SOA type) managed automagically by the backend.

[Creating a TLSA RRset](https://desec.readthedocs.io/en/latest/dns/rrsets.html#creating-a-tlsa-rrset) seems to be correctly handled by the API…

> To generate the TLSA from your certificate, you can use a tool like <https://www.huque.com/bin/gen_tlsa>.
> We are planning to provide a tool that is connected directly to our API in the future.
> For full detail on how TLSA records work, please refer to [RFC 6698](https://datatracker.ietf.org/doc/html/rfc6698).

[Atomicity](https://desec.readthedocs.io/en/latest/dns/rrsets.html#atomicity) considerations.

> `DNSKEY`, `DS`, `CDNSKEY`, `CDS`, `NSEC3PARAM`, `RRSIG`
> 
> These record types are meant to provide DNSSEC-related information in order to secure the data stored in your zones.
> RRsets of this type are generated and served automatically by our nameservers.
> It is currently not possible to read or manipulate any automatically generated values using the API.

There are several caveats that arise from the DNSSEC RFCs. The user should be guided through these if relevant. <!-- Read the RFCs, so the user doesn’t have to. -->

### [Caveats](https://desec.readthedocs.io/en/latest/dns/rrsets.html#caveats)

`CNAME` records must be unique.
A consequence of this is that a `CNAME` is not allowed at the zone apex (empty subname), as it would always collide with the `NS` record (and the internally managed `SOA` record).

* Record types with priority field: need API design considerations?
* Almost certainly, touching `CNAME`, `NS`, and `SOA` record handling will not be part of the thesis.
* In `NS` records _“[t]he use of wildcard RRsets (with one component of subname being equal to `*`) of type `NS` is **discouraged**. This is because the behavior of wildcard `NS` records in conjunction with DNSSEC is undefined, per RFC 4592, Sec. 4.2.”_. <!-- If this ever gets to be part of the thesis, this is a super minor detail that can be pointed out in some UI. -->

### [Configuring your dynDNS Client](https://desec.readthedocs.io/en/latest/dyndns/configure.html)

* [DynDNS (Dynamic DNS, DDNS)](https://en.wikipedia.org/wiki/Dynamic_DNS) is a method of automatically updating a name server in the DNS. See [RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136).
* [TSIG (transaction signature)](https://en.wikipedia.org/wiki/TSIG) provides security. See [RFC 2845](https://datatracker.ietf.org/doc/html/rfc2845)

Routers have a DynDNS client. Some have built-in support for deSEC.

These are technical networking details; they may not be important for the thesis.

_“Token scoping is on our roadmap.”_ — Check if and how this has been solved already. This might set a precedent as to how the deSEC UI is extended.

### [IP Update API](https://desec.readthedocs.io/en/latest/dyndns/update-api.html)

[DynDNS2 protocol specification](https://stackoverflow.com/q/54039095/4642212) is mentioned and can be found on [SourceForge](https://sourceforge.net/p/ddclient/wiki/protocols/#dyndns2).

### [TLS Certificates with Let’s Encrypt](https://desec.readthedocs.io/en/latest/integrations/lets-encrypt.html)

[certbot with deSEC Plugin](https://desec.readthedocs.io/en/latest/integrations/lets-encrypt.html#certbot-with-desec-plugin) exists.
