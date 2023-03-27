## How do DNSKEY, DS, NSEC, and RRSIG interact?

<https://www.imperva.com/learn/application-security/dnssec/>
<https://www.imperva.com/learn/wp-content/uploads/sites/13/2019/01/DNSSEC.jpg.webp>
<https://www.cloudflare.com/dns/dnssec/how-dnssec-works/>

> It’s actually this full RRset that gets digitally signed, [as] opposed to individual DNS records.
> Of course, this also means that you must request and validate all of the AAAA records from a zone with the same label instead of validating only one of them.

**RRSIG** has signature of entire RRSet of any one type.

ZSK is private–public key pair. Private: signs one RRSet. Public: is one of the **DNSKEY**s.

KSK is private–public key pair. Private: signs specifically the DNSKEY RRSet. Public: is another one of the **DNSKEY**s.

Child zone: _owns_ **DNSKEY** with public KSK. Parent zone: _owns_ **DS** with hash of public KSK from DNSKEY.
Ownership is distinct from identified domain: DS record is owned by parent zone, but has domain of child zone associated with it.

**NSEC** just tells you what the next child zone in alphabetical order is.
This record only exists so it can be authenticated in case no other RRSet exists under a specific domain name.

→ [Where does my ds record originate from?](https://serverfault.com/a/734034/1006139)

### Example

```sh
dig dwc-amsterdam.com IN DS @d.gtld-servers.net
```
