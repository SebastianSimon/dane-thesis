# Establishing a DANE identity manager with usable security

## Abstract

Similar to the one from Proposal.

## Table of contents

This.

## 1. Introduction

Brief overview of problem to be solved and the methods to solve it, only explaining enough for the subsequent sections to make sense.
_We want to establish a (prototype of a) DANE IdM application that synchronizes TLSA entries with TLS certificates automatically._
_We do this by creating a web app and OpenSSL in the backend._
_TLSA entries allow server authentication._
_In order to publish DANE identity associations correctly, we need to examine the full life cycle of an identity._

Provide overview over subsequent sections:
Section 2 examines the _protocol_ aspect, section 3 examines the _standards_ aspect, and section 4 examines the _implementation_ aspect of the proposed application.

## 2. Protocols involved in DANE identity management

### 2.1. Transport Layer Security (TLS) certificates

* Purpose
* PKIX
* CAs
* X.509
* PEM / DER
* Validity period
* …

### 2.2. The Domain Name System (DNS)

* Purpose
* Architecture
* DNS zones and DNS names
* RR types
* …

#### 2.2.1. Security extensions for the DNS (DNSSEC)

* Chain of trust
* Purpose of DNSSEC for DANE
* …

#### 2.2.2. The DANE TLS association (TLSA)

* Structure
* Usages
* Recommended parameter combinations
* Validity of a TLSA RRSet
* …

#### 2.2.3. Other DANE associations: SSHFP, IPSECKEY, SMIMEA, OPENPGPKEY

* Purpose of each one

### 2.3. Related problems solved by similar protocols

* DNS over HTTPS
* RFC 7817
* …

## 3. The standards aspect of identity management

### 3.1. Terminology

* Identity
* Identity management
* Authentication
* …

### 3.2. Identity life cycle and Identity management

* ISO/IEC 24760
* …

### 3.3. Information security

* Availability
* Maybe ISO 27001
* …

### 3.4. Usability (and accessibility)

## 4. User-centered design implementation using web technologies

* Avoiding user error

### 4.1. host, dig

### 4.2. nmap

### 4.3. OpenSSL

### 4.4. Node.js server and API

* Automation

### 4.5. Web application and user interface

* User guidance, UX, intuitive UI
* Communicating the security state to the user

## 5. Conclusion and future work

* Evaluation of how the proposed implementation follows the standards and the protocols
* Evaluation of how it increases security
* Possible integration into a real DNS manager
* Maybe prior work?
