<https://www.rfc-editor.org/rfc/rfc7671.html#section-8>

A TLSA RR corresponds to either exactly one certificate or exactly one public key.

Q: Does it make sense to publish multiple TLSA RRs for a single certificate or public key?
If yes, which parameter combinations make sense together in the same zone?
The RFC explicitly allows multiple matching types to be used.
It also allows PKIX-EE and DANE-EE or PKIX-TA and DANE-TA to be used simultaneously.
What about the Selector field; does Full and SPKI make sense together?
Does TA and EE make sense together in the same zone?

A: …

Every change of any TLSA RR invokes a change of the RRSIG RR corresponding to the TLSA RRSet.
TLSA RRs can only really change “state” by being republished with different parameter combinations.

RFC 7671, section 8 provides guidelines for the correct operations if one wishes to change the parameters of specific TLSA RRs.
However, automating these operations requires tracking the state transitions of multiple TLSA RRs while maintaining the validity accross the entire TLSA RRSet.

TODO: Flowchart with “Validation” subgraph

## Validating the TLSA RRSet

TLSA RRSet can be grouped by parameter combination.

At any time, the TLSA RRSet must have the following requirements:

* For all parameter combinations (U, S, M) in { 0, 1, 2, 3 } × { 0, 1 } × { 0, 1, 2 }, for which at least one (secure, signed) TLSA RR exists:
  * If M is 2, then TLSA RRs with (U, S, 1) must also all exist and for all param combos (U’, S’, 2) for which TLSA RRs exist, (U’, S’, 1) must also exist, _and_
  * If U is 3 and (S is 1 or S is 0 and M is 0), then certificate is not strictly necessary (the (3, 0, 0) one is optional) (clients can negotiate authentication via raw public key) (but I consider this out of scope for now), _and_
  * If U is 3, the validity period of the TLS certificate must be ignored for validation (but this is probably not a publisher requirement), _and_
  * If U is 2 and _not_ all TLSA RRs have (2, 0, 0), then full certificate chain in TLS is required, _and_
  * If U is 2 and (S is 1 or M is not 0), then server must include TA in cert chain, _and_
  * If U is 2 and (S is 1 or M is 0), then server should include TA in cert chain, _and_
  * If U is 2, validity period of the TLS certificate is important, _and_
  * If U is 2, M should not be 0, _and_
  * If U is 2, S should be 0, _and_
  * If U is 1, then validity period of the TLS certificate is important, but everything else should be handled like U being 3, _and_
  * One TLSA RR must correspond to a currently used certificate, _and_
  * If U is 0, certain client requirements apply…
* Corollary:
  * Not having any TLSA RRs is allowed, but such a TLS server would not be supporting DANE (for TLS), by definition.
  * Publishing both PKIX and DANE usages is allowed.
  * If U is 3, including the full certificate chain in TLS is optional.

For every param combo, if a set of TLSA RRs with that parameter combination is not empty, then these TLSA RRs must be sufficient to match current cert chain.

## State

RFC 7671 provides:

* key rollover
* Switching to a different usage
  * 1 1 1 → 3 1 1
  * 3 x y → 2 x y
  * x y 1 → x y 2
* Removing a TLSA matching type

Server key rotation → Not necessarily invokes a change for DANE-TA (<https://www.rfc-editor.org/rfc/rfc7671#section-5.2>).

## Recommended TLSA parameter combinations according to RFC 7671

UU is the Usage field (in binary); S is the Selector field; M is the MatchingType field (* is either 1 or 2).

| UU | S |  M | Recommendation |
|----|---|----|----------------|
| 00 | 0 |  0 | avoid          |
| 00 | 0 | \* |                |
| 00 | 1 |  0 | suboptimal     |
| 00 | 1 | \* |                |
| 01 | 0 |  0 | avoid          |
| 01 | 0 | \* |                |
| 01 | 1 |  0 | suboptimal     |
| 01 | 1 | \* |                |
| 10 | 0 |  0 | avoid          |
| 10 | 0 | \* | okay           |
| 10 | 1 |  0 | avoid          |
| 10 | 1 | \* | avoid          |
| 11 | 0 |  0 | avoid          |
| 11 | 0 | \* |                |
| 11 | 1 |  0 | suboptimal     |
| 11 | 1 | \* | best           |
