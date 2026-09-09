# 02 — Architecture

Status: Draft for Phase 0 review
Owner: Shikhar

---

## 2.1 Design principles

    1  Systems of record do not change.
       ATP reads over the existing KYC store, card master, merchant master
       and AML platform. No migration, no dual-write, no data leaving the
       regulated estate. This is what makes the programme deliverable
       without a multi-year data programme underneath it.

    2  PII never enters a credential.
       Enforced by schema validation at mint time, not by reviewer
       discipline. A credential payload that contains a name field should
       fail CI.

    3  Standards over invention.
       Anything proprietary is dead on arrival with eIDAS and with any
       partner's architecture review.

    4  Every phase stands alone.
       Phase 1 delivers measurable value with Amex as the only relying
       party. No phase assumes the next is funded.

    5  Fail open, never fail closed.
       ATP sits beside the authorisation path, not across it. An ATP outage
       produces unattested transactions, never declined ones.

---

## 2.2 Service topology

    ┌────────────────────────────────────────────────────────────────┐
    │ CLIENTS                                                         │
    │  Amex app wallet   Partner SDK (JVM/JS/Py)   Agent runtime      │
    └───────────────────────────┬────────────────────────────────────┘
                    OpenID4VCI / OpenID4VP / REST
    ┌───────────────────────────▼────────────────────────────────────┐
    │ atp-gateway                                                     │
    │  mTLS termination · RP registry · entitlements · rate limiting  │
    │  request signing verification · billing event emission          │
    └───┬──────────┬──────────┬──────────┬──────────┬────────────────┘
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
    ┌────────┐┌──────────┐┌───────────┐┌──────────┐┌──────────┐
    │  atp-  ││   atp-   ││    atp-   ││   atp-   ││   atp-   │
    │ issuer ││ verifier ││ revocation││  mandate ││  audit   │
    │        ││          ││           ││          ││          │
    │ mint   ││ verify   ││ status    ││ agent    ││ append   │
    │ VCs    ││ VPs      ││ lists     ││ scope    ││ only     │
    └───┬────┘└────┬─────┘└─────┬─────┘└────┬─────┘└────┬─────┘
        │          │            │           │           │
        └──────────┴─────┬──────┴───────────┘           │
                         ▼                              │
    ┌────────────────────────────────────────────────┐  │
    │ atp-identity-core                               │  │
    │   entity resolution   (REUSE existing service)  │  │
    │   document intelligence (VLM)                   │  │
    │   PAD + injection detection                     │  │
    │   sanctions / PEP orchestration                 │  │
    │   continuous risk scoring (GNN)                 │  │
    └────────────────────────┬───────────────────────┘  │
                             ▼                          │
    ┌────────────────────────────────────────────────┐  │
    │ EXISTING SYSTEMS OF RECORD — unchanged          │  │
    │  KYC store · card master · merchant master ·    │  │
    │  AML platform · sanctions screening             │  │
    └────────────────────────────────────────────────┘  │
                                                        │
    ┌────────────────────────────────────────────────┐  │
    │ atp-hsm-adapter → existing Amex HSM             │  │
    │  issuer key custody · signing · rotation        │  │
    └────────────────────────────────────────────────┘  │
                                                        │
    ┌────────────────────────────────────────────────◄──┘
    │ Kafka: atp.events.*  → audit sink, billing, risk   │
    └────────────────────────────────────────────────────┘

    PHASE 4, OPTIONAL, GATED
    ┌────────────────────────────────────────────────────┐
    │ atp-anchor  → ledger. Status pointers and schema    │
    │               IDs only. Never PII, never hashes of  │
    │               PII.                                  │
    └────────────────────────────────────────────────────┘

---

## 2.3 Service responsibilities

    atp-gateway
      Single ingress for relying parties and wallets. Terminates mTLS,
      resolves the RP from the client certificate, enforces entitlements
      (which credential types and which claims this RP may request), applies
      rate limits, emits billing events. Stateless. Horizontally scaled.

    atp-issuer
      Mints credentials. Orchestrates identity-core, obtains a signature
      from the HSM adapter, allocates a status-list index, writes the
      issuance registry record. Never returns PII to the caller.

    atp-verifier
      Verifies presentations. Signature check, key-binding check, nonce
      freshness, audience binding, status lookup. Returns the structured
      four-way outcome. Read-only against a cached status view — this is the
      hot path and must not touch the issuance database.

    atp-revocation
      Owns status lists. Publishes signed, compressed bitstrings to the CDN.
      Handles revocation requests from Risk, from the member, and from the
      continuous risk scorer. Guarantees 60s propagation.

    atp-mandate
      Agent-specific. Issues mandates, tracks cumulative spend, evaluates
      scope at authorisation time. The only service with a 40ms budget; runs
      colocated with the authorisation platform and holds mandate state in
      an in-memory store with write-behind persistence.

    atp-audit
      Append-only event sink. Every issuance, presentation, verification,
      revocation and mandate decision. Feeds compliance reporting and
      dispute resolution. Write path must never block the caller.

    atp-identity-core
      All ML and screening orchestration. See 06-ml-specification.md.

---

## 2.4 Architecture decision records

### ADR-001 — Credential format: SD-JWT VC, not JSON-LD with BBS+
[DECIDED]

Context. Selective disclosure can be achieved with SD-JWT (salted claim
hashes) or with BBS+ signatures over a JSON-LD credential.

Decision. SD-JWT VC.

Rationale. SD-JWT is on the IETF standards track, has production library
support across JVM, and uses only ES256 — which every mobile secure enclave
supports natively. BBS+ gives unlinkable multi-show presentation, which is
cryptographically superior, but has thin library support, no secure-enclave
support, and would put an unratified primitive in a regulated path.

Consequence. Multiple presentations of the same credential to the same RP
are linkable to each other. Mitigated by pairwise identifiers (ADR-004),
which prevent linkage across RPs — the threat that actually matters.
Revisit if BBS+ ratifies.

---

### ADR-002 — No distributed ledger before Phase 4
[DECIDED]

Context. The originating concept specified Ethereum Attestation Service.

Decision. No ledger dependency in Phases 0–3. Phase 4 is separately gated on
demonstrated external demand.

Rationale. Three reasons, any one sufficient.
  1. Regulatory. Immutable storage is irreconcilable with GDPR Art. 17
     erasure. Even salted hashes of PII are personal data under EDPB
     guidance when the input space is enumerable.
  2. Unnecessary. Every element of the value proposition — reuse, selective
     disclosure, no PII sharing, revocation — is delivered by SD-JWT VC
     with a hosted status list.
  3. Organisational. A ledger dependency converts this from a platform
     programme into "the Amex blockchain initiative," acquiring a review
     surface that will outlive its funding.

Consequence. Cross-ecosystem verification by counterparties who refuse to
federate with an Amex endpoint is not supported until Phase 4. Accepted.

---

### ADR-003 — Signing algorithm ES256
[DECIDED]

Context. Need an algorithm supported by mobile secure enclaves, HSMs, and
every partner language runtime.

Decision. ECDSA P-256 with SHA-256 (ES256) for issuer signatures and holder
key binding. SHA-256 for SD-JWT claim digests.

Rationale. Universal hardware support. Apple Secure Enclave and Android
StrongBox both generate and hold P-256 keys natively and non-exportably.
EdDSA is cleaner cryptographically but enclave support is inconsistent.

Consequence. Post-quantum migration is a known future work item. Credential
lifetimes are capped at 12 months (ADR-006), which bounds exposure and gives
a natural migration window. Track NIST PQC signature standardisation.

---

### ADR-004 — Pairwise pseudonymous subject identifiers
[DECIDED]

Decision. Each relying party receives a different subject identifier for the
same person, derived as:

    subject_ref = base64url(HMAC-SHA256(
                    key   = issuer_pairwise_secret (HSM-held),
                    msg   = internal_subject_id || ":" || rp_id
                  ))[0:32]

Rationale. Deterministic, so the RP gets a stable identifier for its own
returning-user logic. Unlinkable, so two RPs comparing databases cannot
determine they hold the same person. Without this the credential becomes a
cross-industry tracking key, and it will be treated as one by regulators and
by the press.

Consequence. Amex can re-derive linkage internally, which is required for
fraud investigation and lawful process. This asymmetry must be disclosed in
the privacy notice — it is a feature, but an undisclosed one is a scandal.

---

### ADR-005 — Fail open on verification unavailability
[DECIDED — requires explicit risk acceptance]

Decision. If ATP is unavailable, transactions proceed as unattested. ATP
never causes a decline.

Rationale. ATP sits beside the payment authorisation path. A 99.99% service
in front of a 99.999% path degrades the path. The failure mode of a missing
attestation is a transaction scored as it would have been before ATP
existed — the status quo. The failure mode of a hard dependency is lost
volume.

Consequence. An attacker who can DoS ATP can strip attestation and therefore
liability shift. Mitigated by: regional independence (NFR-07), edge-cached
status lists, and a documented Risk position that sustained unavailability
triggers manual review thresholds rather than open acceptance.

---

### ADR-006 — Credential lifetime 12 months, status checked every use
[PROPOSED]

Decision. Credentials expire at 12 months. Revocation status is checked on
every presentation against a status list with 60s propagation.

Rationale. Short lifetimes limit blast radius from key compromise and force
periodic re-attestation of underlying facts. Continuous risk scoring
(06-ml, model M5) can revoke inside the window when facts change, so the
12-month expiry is a backstop, not the primary control.

Open. Whether Business Passport warrants a shorter lifetime given faster
change in beneficial ownership. Owner: Compliance. Due: Phase 2 design.

---

### ADR-007 — Reuse existing record-linkage service for entity resolution
[PROPOSED — validation is Phase 0 OQ-03]

Decision. Use the existing production record-linkage inference API
(Fellegi-Sunter probabilistic linkage with calibrated LightGBM confidence
scoring) as the entity-resolution engine, rather than building new.

Rationale. It is deployed, calibrated, already exposed as a scored inference
service that other Amex applications threshold on, and solves the same
problem: does this record map to exactly one identity, with what confidence.
Calibration is the expensive part and it is done.

Consequence if true. 9-month platform.
Consequence if false. 18-month platform, and the competitive timing argument
changes materially. This single question is the largest variance in the plan
and is why it is a named Phase 0 deliverable with a day-45 due date.

---

### ADR-008 — Agent protocol adapters are pluggable, not foundational
[DECIDED]

Context. AP2, Visa TAP, Mastercard Verifiable Intent, OIDC-A and the IETF
AIP track are all live and unconverged. FIDO opened an Agentic
Authentication working group in April 2026.

Decision. The credential and mandate layer is built to W3C VC. Competing
agent protocols are implemented as transport adapters behind a stable
internal interface.

Rationale. Betting the architecture on one protocol winning is an
unnecessary risk in an unsettled standards landscape. Adapters are weeks of
work each; a foundational bet is a rewrite.

---

## 2.5 Sequence — issuance to an existing cardmember (J1)

    Member    App        gateway    issuer    id-core   HSM    revocation
      │        │            │         │          │       │         │
      │ opt-in │            │         │          │       │         │
      ├───────►│            │         │          │       │         │
      │        │ POST /issue│         │          │       │         │
      │        ├───────────►│         │          │       │         │
      │        │            ├────────►│          │       │         │
      │        │            │         │ resolve  │       │         │
      │        │            │         ├─────────►│       │         │
      │        │            │         │          │ reads KYC store │
      │        │            │         │◄─────────┤ score=0.997     │
      │        │            │         │          │       │         │
      │        │  generate keypair    │          │       │         │
      │        │  in secure enclave   │          │       │         │
      │        │            │         │          │       │         │
      │        │ pubkey     │         │          │       │         │
      │        ├───────────►├────────►│          │       │         │
      │        │            │         │ allocate status idx        │
      │        │            │         ├───────────────────────────►│
      │        │            │         │◄───────────────────────────┤
      │        │            │         │ sign     │       │         │
      │        │            │         ├─────────────────►│         │
      │        │            │         │◄─────────────────┤         │
      │        │◄───────────┤◄────────┤          │       │         │
      │ stored │            │         │          │       │         │
      │◄───────┤            │         │          │       │         │

    No document scan. Target p95 under 30 seconds, dominated by the
    consent UI, not by compute.

---

## 2.6 Sequence — presentation to a relying party (J3)

    RP        RP-SDK     gateway   verifier  revocation   Wallet   Member
     │          │           │         │          │          │        │
     │ request  │           │         │          │          │        │
     ├─────────►│           │         │          │          │        │
     │          │ OpenID4VP authz req │          │          │        │
     │          ├──────────────────────────────────────────►│        │
     │          │           │         │          │  show what is     │
     │          │           │         │          │  asked + by whom  │
     │          │           │         │          │          ├───────►│
     │          │           │         │          │          │◄───────┤
     │          │           │         │          │  per-claim consent│
     │          │           │         │          │          │        │
     │          │  VP: disclosed claims + KB-JWT over nonce  │        │
     │          │◄──────────────────────────────────────────┤        │
     │          │ POST /v1/verify     │          │          │        │
     │          ├──────────►├────────►│          │          │        │
     │          │           │         │ status?  │          │        │
     │          │           │         ├─────────►│ (edge cached)     │
     │          │           │         │◄─────────┤          │        │
     │          │           │         │          │          │        │
     │          │           │  verify: issuer sig, key binding,      │
     │          │           │  nonce freshness, audience, status     │
     │          │           │         │          │          │        │
     │          │◄──────────┤◄────────┤          │          │        │
     │◄─────────┤ {verified, claims, subject_ref, warranty} │        │

    p99 budget 150ms. Status lookup is edge-cached and must not touch the
    issuance database.

---

## 2.7 Sequence — agent transaction (J6), the 40ms path

    Agent    Merchant   Auth platform   atp-mandate   mandate store
      │         │             │              │             │
      │ checkout│             │              │             │
      ├────────►│             │              │             │
      │         │ auth request + mandate VC  │             │
      │         ├────────────►│              │             │
      │         │             │ authorize    │             │
      │         │             ├─────────────►│             │
      │         │             │              │ scope check │
      │         │             │              ├────────────►│
      │         │             │              │◄────────────┤
      │         │             │              │ verify agent sig
      │         │             │              │ over txn hash
      │         │             │◄─────────────┤             │
      │         │             │ {authorized, liability_shift,
      │         │             │  scope_remaining, risk_band}
      │         │             │              │             │
      │         │             │ continue normal auth       │
      │         │◄────────────┤              │             │
      │◄────────┤             │              │             │

    Hard budget p99 40ms. Design consequences:
      - atp-mandate colocated with the auth platform, same region, same AZ
      - mandate state in memory, write-behind to durable store
      - no synchronous call to identity-core, revocation, or audit on this
        path; audit is emitted asynchronously
      - cumulative spend counter is eventually consistent; over-spend within
        the consistency window is a bounded, accepted risk sized at one
        transaction per mandate

---

## 2.8 Deployment

    Regions        multi-region active-active for verifier and mandate;
                   issuer may be regional to satisfy data residency
    Data residency EU credentials issued and verified in EU region; no
                   cross-border PII movement (07-compliance, section 7.3)
    Runtime        Kotlin / Spring Boot, JVM 21, containerised
    Persistence    PostgreSQL — issuance registry, mandate store
                   Redis — status cache, mandate hot state
                   Kafka — atp.events.* audit and billing stream
                   CDN — signed status lists, public, cacheable
    Secrets        existing Amex HSM via atp-hsm-adapter; no signing key
                   material in application memory, ever

---

## 2.9 What is deliberately simple

Worth stating so reviewers do not add complexity that is not needed.

  - No credential storage server-side. Amex holds the issuance registry
    (who was issued what, when, status) but not the credential itself. The
    holder holds it. This is what makes it a wallet credential rather than
    a lookup API with extra steps.
  - No new identity store. Identity lives where it lives today.
  - No consensus, no ledger, no smart contracts in Phases 0–3.
  - No custom cryptography. Every primitive is standards-track with
    existing library support.
