Idea:

* For the software user, it doesn’t really matter how “identity” is defined exactly, since this software — directly or indirectly — manages TLS certificates, public keys, TLSA RRs, subdomains, and DNS zones, all of which are — directly or indirectly — interrelated with identities, if not identical.
* Need correct management of and interaction with CNAME and other DNS records.
* Let’s Encrypt already supports auto-renew — possibly other CAs as well.
* “The state” to be communicated to the user:
  * Inactive (Established) → Prompt user to configure and enable once; guide them
  * Scheduled (If TLSA needs to be published before TLS)
  * Active
  * Expired (Might just be replaced by the next cert with state “Scheduled”, unless auto-renew is not possible)
  * Revoked (Archived)
  * Error → Inform user via e-mail
* Need to find out best practice approach: should key pair always be replaced with each certificate renewal? Does Let’s Encrypt support that?

## What is being managed?

* The identity term is quite complex, so this might just confuse the user. Therefore I would avoid immediately confronting the user with a list literally titled “Identities”.
* TLS certificates are something more concrete and users are more likely expected to be familiar with them.
* Subdomains need to be explicitly enumerated. They belong to DNS zones which the user needs to demonstrate ownership for; maybe this can be done implicitly through the subdomains. <!-- In JS, maybe `someSubdomain.dnsZone.authority`? -->
* TLSA RRs and RRSets never need to be explicitly managed.
* Public keys are bound to TLS certificates, so they never need to be explicitly managed (in the context of TLSA).
* If TLS certificates are being explicitly managed, then OpenPGP keys, SSH fingerprints, S/MIME keys, and IPSec keys should also be explicitly managed.
* The deSEC layout might be useful: one tab for subdomain management, another for identity management; this reflects the two components that constitute the system: DNS management and key (or certificate) management.

---

The app might have the following UI components:

## Config

* [x] Enable DANE (should probably not even be an option)
* [x] Use SHA2-512 (recommended) \[Use an additional hash to the default SHA2-256 one\]
* Let’s Encrypt API key (not sure how this would work from an API perspective; the app would probably need to store API keys securely somehow)

## “Identities” or “Keys” or “Certificates”

List of identities corresponds to list of _current_ certificates.

| Certificate      | Zones            | Usage / TLSA                              | Valid from | Valid until | Status        |
|------------------|-----------------:|-------------------------------------------|------------|-------------|---------------|
| The \*.x.com one | y.x.com, z.x.com | { PKIX, DANE }-{ TA, EE }, { Cert, SPKI } | 2023-01-01 | 2023-04-01  | ⟨“The state”⟩ |
| The x.com one    |            x.com | { PKIX, DANE }-{ TA, EE }, { Cert, SPKI } | 2022-01-01 | 2024-12-01  | ⟨“The state”⟩ |

Upload certificate with Drag & Drop area.

## Certificate view

When clicking on a certificate row:

Name: The \*.x.com one
Zones: y.x.com, z.x.com
…
Valid from 2023-01-01, til 2023-04-01.
Auto-renew enabled; renews on date X.

### List of corresponding TLSA RRs

* 3 1 1 XYZ
* 3 1 2 XYZ
* …

(If state is “Inactive”:)

* { PKIX, DANE }-{ TA, EE }, { Cert, SPKI }
* [ ] Publish TLSA as Full matching type (not recommended)

<kbd>Queue Publishing</kbd>

(Otherwise:)

<kbd>Revoke certificate</kbd> → Must have replacement ready.

Publish next time with:

* { PKIX, DANE }-{ TA, EE }, { Cert, SPKI }
* [ ] Publish TLSA as Full matching type (not recommended)

<kbd>Save for next time</kbd>
<kbd>Queue Publishing _now_</kbd> → Replaces current certificate?
