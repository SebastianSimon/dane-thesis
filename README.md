# BSc computer science thesis about DNS-based Authentication of Named Entities (DANE) identity management

## Temporary structure

1. Abstract
2. Introduction
3. DANE theory
   1. Overview of DANE
   2. DNSSEC as basis for DANE
   3. Interaction between DNSKEY, DS, NSEC, and RRSIG
   4. DNSSEC requirements for DANE
   5. Definition of Identity, Authentication, Identity management and life cycle
   6. Added security value of DANE
   7. Related
      1. DNS over HTTPS
   8. Common misconceptions
4. Life cycle analysis
   1. TLS
      1. TLS certificates
         1. Life cycle events of a TLS certificate
      2. TLSA RRs
         1. Life cycle events of a TLSA RR
      3. Relationship between a TLSA record and its certificate
   2. SSH
      1. SSH fingerprints
         1. Life cycle events of an SSH fingerprint
      2. SSHFP RRs
         1. Life cycle events of an SSHFP RR
5. DANE identity manager
   1. The manual process of publishing a DANE RR
   2. The need for a DANE identity manager
   3. Requirements of a DANE identity manager
   4. Conceptual design of a DANE identity manager
      1. Automation
      2. User guidance
      3. Communicating the state of security
      4. API
6. Evaluation
   1. Usability
      1. User-centered design standards as a benchmark
      2. Evaluation of the DANE identity manager in terms of usability
         1. Accessibility
   2. Security
      1. Security standards as a benchmark
      2. Evaluation of the DANE identity manager in terms of security
   3. The added value of a DANE identity manager

| Current focus | File tree |
|:-------------:|:----------|
|               | `.` |
|               | `├── DANE identity manager` |
|               | `│   ├── Conceptual design of a DANE identity manager` |
|      ⭐        | `│   │   └── Conceptual design of a DANE identity manager.md` |
|               | `│   └── deSEC.md` |
|               | `├── DANE theory` |
|               | `│   ├── Added security value of DANE.md` |
|      ⭐        | `│   ├── Definition of Identity, Authentication, Identity management and life cycle.md` |
|      ⭐        | `│   ├── DNSSEC as basis for DANE.md` |
|      ⭐        | `│   ├── DNSSEC requirements for DANE.md` |
|      ⭐        | `│   ├── Interaction between DNSKEY, DS, NSEC, and RRSIG.md` |
|      ⭐        | `│   ├── Overview of DANE.md` |
|      ⭐        | `│   ├── Public-key cryptography.md` |
|      ⭐        | `│   └── TLSA RRs.md` |
|               | `└── Life cycle analysis` |
|      ⭐        | `    ├── Life-cycle analysis.md` |
|               | `    └── TLS` |
|      ⭐        | `        ├── Relationship between a TLSA record and its certificate.md` |
|               | `        ├── TLSA RRs` |
|      ⭐        | `        │   └── Life cycle events of a TLSA RR.md` |
|               | `        └── TLS certificates` |
|      ⭐        | `            └── Introduction.md` |
