# Amex Trust Passport — Documentation Set

Platform: Amex Trust Passport (ATP)
Relying-party mark: "Verified by Amex"
Owner: Shikhar
Status: Draft for Phase 0 review
Last updated: 2026-09-09

---

## What this is

Engineering documentation for a reusable, privacy-preserving identity
credential platform issued by American Express. It lets a cardmember, a
business, or an AI agent prove they were verified to Amex standard without
disclosing the underlying PII.

The business case lives in the separate pitch document. This set is what the
team builds from.

---

## Reading order

    New to the programme
      00  this file
      01  PRD — what we are building and for whom
      02  Architecture — how it fits together

    Building a service
      02  Architecture
      03  API specification
      04  Data model
      05  Security and threat model

    Building the ML components
      06  ML specification
      05  Security (sections 5.4, 5.7)

    Risk, Legal, Compliance review
      07  Compliance mapping
      05  Security and threat model
      01  PRD (section 1.8, out of scope)

    On call
      08  Operations runbook

    Planning
      09  Delivery plan

---

## Document index

    00-README.md                  this file
    01-PRD.md                     product requirements, personas, journeys,
                                  functional and non-functional requirements
    02-architecture.md            HLD, service topology, ADRs, sequence flows
    03-api-spec.md                endpoint contracts, error taxonomy, SDKs
    04-data-model.md              credential schemas, DDL, event contracts
    05-security-threat-model.md   STRIDE, cryptography, key management
    06-ml-specification.md        model cards, evaluation, model governance
    07-compliance-mapping.md      eIDAS, GDPR, BSA/AML traceability
    08-operations-runbook.md      SLOs, alerting, incident playbooks
    09-delivery-plan.md           phases, epics, sizing, test strategy

---

## Document status legend

Every section carries one of these markers where the status is not obvious.

    [DECIDED]     agreed, build to this
    [PROPOSED]    author's recommendation, needs review sign-off
    [OPEN]        genuine unknown, owner and due date named
    [ASSUMPTION]  placeholder awaiting a real internal number
    [PHASE 0]     to be resolved during discovery, do not build against yet

Anything marked [ASSUMPTION] must be replaced before a funding committee
sees it. Anything marked [OPEN] must have a named owner before the phase in
which it blocks work.

---

## Conventions

Service naming        atp-<function>, lowercase, hyphenated
Language              Kotlin / Spring Boot, consistent with existing estate
Credential format     W3C VC 2.0 with SD-JWT VC selective disclosure
Signing               ES256 (ECDSA P-256)
Time                  RFC 3339, UTC, always
Money                 minor units as integer plus ISO 4217 currency, never
                      floating point
Identifiers           ULID for internal refs, prefixed by type (mp_, bp_,
                      amd_, vrf_)

---

## Glossary

    ATP           Amex Trust Passport, the platform
    Assertion     a single verifiable claim, e.g. age_over_18 = true
    Attestation   a signed statement by Amex that assertions are true
    ARF           Architecture and Reference Framework, the eIDAS 2.0
                  technical specification
    Claim         see assertion
    Credential    a signed bundle of assertions held by a subject
    Holder        the party in possession of a credential (the cardmember)
    Issuer        Amex
    Key binding   cryptographic proof that the presenter holds the private
                  key the credential was issued to
    Mandate       an agent credential encoding delegated spending authority
    PAD           presentation attack detection (liveness)
    Pairwise ID   a subject identifier unique to one relying party, so two
                  relying parties cannot correlate the same person
    Predicate     a claim expressed as a true/false test rather than a value,
                  e.g. age_over_18 rather than date_of_birth
    Relying party the party verifying a credential (RP)
    SD-JWT        Selective Disclosure JWT — the credential wire format
    Status list   a compressed bitstring encoding revocation state for many
                  credentials at once
    VC / VP       Verifiable Credential / Verifiable Presentation
