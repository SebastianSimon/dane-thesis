# BSc computer science thesis _Management of Cryptographic Identities using DNS-based Authentication of Named Entities (DANE)_

Written 2023-05-24 by Sebastian Simon, for the _Technische Universität Berlin_, department for Security in Telecommunications (SecT).

## Abstract

The Domain Name System (DNS) provides a service to look up server addresses by name, but this service, by default, lacks authentication and is prone to forgery.
Various security protocols and extensions exist, but their complexity is high and, thereby, adoption is increasing only slowly.

This thesis focuses on automating and simplifying deployment of DNS-based Authentication of Named Entities (DANE), a set of security protocols which can be used by clients to authenticate a server.
In particular, we establish a prototype of an identity management application, driven by user-centered design, which helps in DNS zone administration with automatic synchronization between a TLS certificate on the server and a TLSA Resource Record in the DNS.
This synchronization provides the TLS certificate association which enables clients to cross-check cryptographic data, such as public keys or certificates, from the server itself and the DNS.
TLS certificates are generally issued by external Certificate Authorities; the purpose of this DANE binding is to be able to avoid trust in them.

The goal of the application is to minimize human error and increase security.
However, implementing DANE requires careful management; an implementation needs to obey different requirements and support different scenarios.
We examine the life cycle events of a server identity and review the relevant specifications and standards, all of which help ensure correct management.

## Repository information

The full thesis is included as [`Management of Cryptographic Identities using DNS-based Authentication of Named Entities (DANE).pdf`](Management%20of%20Cryptographic%20Identities%20using%20DNS-based%20Authentication%20of%20Named%20Entities%20%28DANE%29.pdf).
The version here does not include the signature in the sworn affidavit, and it does not include the email addresses in the title page.

Note that a few footnotes had to be manually entered using `\textsuperscript`, because [Pandoc does not support coalescing footnotes](https://github.com/jgm/pandoc/issues/224).
When generating the output, double check if these footnotes match.

File `ieee-with-url.csl.xml` comes from <https://www.zotero.org/styles/ieee-with-url>.

## Required tools and packages to compile the document from source

* `pandoc-cli` 0.1-51
* `texlive-most` 2023
* `pandoc-crossref` 0.3.15.2-10
* NPM `@marcelmaatkamp/mermaid-filter` 1.4.11 (_must_ use `mermaid-cli` version 9+)
  * NPM `chromium` 3.0.3 (installed _locally_; implicit dependency)
  * NPM `puppeteer` 19.9.0 (installed _locally_; implicit dependency)
* Font _Fira Code_
* Font _FreeSans_
* Font _FreeSerif_

## Pandoc CLI

```sh
pandoc 'Sworn Affidavit.md' -o 'Sworn Affidavit.tex'
pandoc 'Management of Cryptographic Identities using DNS-based Authentication of Named Entities (DANE).md' --pdf-engine='xelatex' --number-sections --toc -F 'pandoc-crossref' -F 'mermaid-filter' --citeproc --csl 'ieee-with-url.csl.xml' --include-before-body='Sworn Affidavit.tex' -o 'Management of Cryptographic Identities using DNS-based Authentication of Named Entities (DANE).pdf'
```

This should take about 20 to 30 seconds.
Make sure there are no errors printed in `mermaid-filter.err` or the terminal.
