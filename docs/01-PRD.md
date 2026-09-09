# 01 — Product Requirements

Status: Draft for Phase 0 review
Owner: Shikhar

---

## 1.1 Problem

Identity verification in financial services is re-purchased, never re-used.
Amex has verified ~87.6M proprietary cardmembers and underwritten a merchant
network past 170M locations. None of that verification is portable. It is
re-paid internally through periodic refresh and re-done from scratch by every
counterparty the same customer touches.

Separately, AI agents now initiate transactions with no mechanism to prove a
verified human delegated the purchase. Visa and Mastercard have shipped
answers. Amex has not.

---

## 1.2 Product goal

Issue a reusable credential that proves verification to Amex standard,
disclosing only the specific assertions a relying party is legally required
to check, and nothing else.

Success means: a cardmember never uploads a document to Amex twice, a
relying party gets a verified answer in under a second without receiving
PII, and an agent transaction carries provable human authorisation.

---

## 1.3 Personas

    P1  Existing cardmember
        Already KYC'd. Must never re-scan a document. Opt-in in the app,
        credential issued in seconds. Primary success metric: time to
        credential under 30 seconds.

    P2  New applicant
        No prior Amex relationship. Full document plus liveness capture.
        Credential issued as a by-product of normal onboarding, at no
        additional friction.

    P3  Business / merchant
        Legal entity, beneficial ownership resolved, sanctions screened.
        Credential held by an authorised officer, presentable on behalf of
        the entity.

    P4  Relying party integrator
        Backend engineer at a partner bank, fintech, marketplace or
        brokerage. Cares about: integration time, error semantics, latency,
        who is liable when the assertion is wrong.

    P5  AI agent runtime
        Software acting for P1 or P3. Holds a scoped mandate. Must present
        it inline in the authorisation path within a hard latency budget.

    P6  Amex internal consumer
        Amex product onboarding and periodic-refresh systems. The first and,
        through Phase 1, the only relying party.

---

## 1.4 Core journeys

    J1  Issue to existing cardmember
        App opt-in → consent → identity resolution against customer graph
        → device keypair generated in secure enclave → credential minted
        → stored in app wallet
        No document scan. No wait. Target: under 30s end to end.

    J2  Issue to new applicant
        Document capture → VLM parse → forgery check → liveness → sanctions
        screening → entity resolution → keypair → credential
        Runs inside existing onboarding. Adds no new user-visible step.

    J3  Present to relying party
        RP requests named predicates → wallet shows exactly what is asked
        and by whom → per-claim consent → selective disclosure with key
        binding → RP verifies signature, binding, freshness, status

    J4  Internal reuse (Phase 1, no external dependency)
        Cardmember applies for a second Amex product, or comes due for
        periodic refresh → existing credential satisfies the check → no
        re-verification performed

    J5  Agent mandate issuance
        Cardmember authorises an agent with explicit scope → biometric
        device confirmation → mandate minted, bound to both the member
        credential and the agent key

    J6  Agent transaction authorisation
        Agent presents mandate plus a signature over the transaction hash →
        ATP verifies scope, freshness, cumulative spend → returns authorised
        or declined with liability position
        Hard budget: p99 under 40ms, inline in the auth path.

    J7  Revocation
        Risk signal, member request, or fraud confirmation → status list
        entry flipped → propagated to edge caches within 60s

---

## 1.5 Functional requirements

    FR-01   Issue Member Passport to an existing cardmember without any
            document re-capture
    FR-02   Issue Member Passport to a new applicant within the existing
            onboarding flow
    FR-03   Support per-claim selective disclosure; the holder discloses
            each assertion independently
    FR-04   Express age, jurisdiction and tenure as predicates, never as
            underlying values
    FR-05   Bind every credential to a hardware-backed device key that
            cannot be exported
    FR-06   Issue a distinct, unlinkable pairwise subject identifier per
            relying party
    FR-07   Publish and serve revocation status via Bitstring Status List
    FR-08   Verify a presentation and return a structured result with a
            four-way outcome taxonomy
    FR-09   Issue Business Passport with beneficial-ownership and sanctions
            assertions
    FR-10   Issue scoped agent mandates bound to a verified human principal
    FR-11   Authorise or decline agent transactions against mandate scope
            inline in the authorisation path
    FR-12   Track cumulative spend per mandate and enforce the cap
    FR-13   Revoke any credential or mandate and propagate within 60s
    FR-14   Emit an immutable audit event for every issuance, presentation,
            verification and revocation
    FR-15   Accept eIDAS ARF-conformant credentials from EU wallets as a
            relying party
    FR-16   Attach a warranty reference and coverage cap to a verification
            response  [OPEN — depends on Legal position, see PRD 1.8]

---

## 1.6 Non-functional requirements

    Latency
      NFR-01  /v1/mandate/authorize      p99 < 40ms   (inline in auth path)
      NFR-02  /v1/verify                 p99 < 150ms
      NFR-03  status list fetch          p99 < 20ms   (edge cached)
      NFR-04  issuance, existing member  p95 < 30s end to end

    Availability
      NFR-05  verification and mandate authorisation: 99.99%
              These sit in the payment authorisation path. An ATP outage
              must degrade to "unattested transaction," never to "declined."
              Fail-open behaviour is a DECIDED requirement and must be
              explicitly risk-accepted.
      NFR-06  issuance: 99.9%
      NFR-07  no single-region dependency for verification

    Scale
      NFR-08  design target 500M verifications/year, peak 5x mean
      NFR-09  87.6M credentials in the issuance registry
      NFR-10  status lists sized for 100M entries with sub-second refresh

    Privacy
      NFR-11  no PII in any credential payload — enforced by schema, not
              by policy
      NFR-12  no PII on any ledger, encrypted or otherwise
      NFR-13  no shared persistent subject identifier across relying parties
      NFR-14  document images never leave the device on the common path

    Security
      NFR-15  issuer signing keys in HSM custody, non-exportable
      NFR-16  all RP traffic over mTLS with per-RP client certificates
      NFR-17  every verification request carries a fresh, single-use nonce

    Compliance
      NFR-18  every automated decision explainable to model risk management
              standard
      NFR-19  full audit trail retained per existing Amex retention policy
      NFR-20  ARF conformance for EU issuance and acceptance

---

## 1.7 Success metrics

    Phase 1 (internal reuse, no external dependency)
      M1  % of periodic refreshes satisfied by credential rather than
          re-verification            target > 60%
      M2  multi-product onboarding time for existing members
                                     target > 70% reduction
      M3  credential issuance opt-in rate among invited cohort
                                     target > 25%
      M4  measured cost per satisfied verification
                                     target < $0.30

    Phase 2
      M5  KYB cycle time for commercial onboarding
                                     target > 50% reduction
      M6  eIDAS ARF conformance certified in target markets   binary

    Phase 3
      M7  external relying parties in production
      M8  agent transactions carrying a valid mandate on Amex rails
      M9  authorisation rate delta, attested vs unattested agent traffic
      M10 attestation revenue per verification

    Guardrail metrics — watched from Phase 1, any regression blocks release
      G1  false-accept rate at enrolment (synthetic identity)
      G2  false-decline rate introduced into onboarding
      G3  p99 latency on the authorisation path
      G4  privacy incidents: any PII appearing in a credential payload —
          target is zero and any occurrence is a Sev-1

---

## 1.8 Out of scope

    Explicitly not in this programme
      - Any distributed ledger dependency before Phase 4, which is
        separately gated
      - Storing or transmitting document images to relying parties
      - Credit decisioning; ATP asserts identity, never creditworthiness
      - Replacing existing KYC systems of record; ATP reads over them
      - Consumer-facing wallet for third-party credentials (Amex issues its
        own; it does not become a general wallet in this scope)
      - Stablecoin issuance. ATP may attest counterparty identity for
        stablecoin settlement flows under GENIUS Act BSA obligations, but
        does not issue, hold or settle any digital asset.

    Blocking open questions
      OQ-01  Will Legal underwrite an identity assertion at any cap?
             Owner: Legal + Risk. Due: Phase 0 day 30. Blocks FR-16 and the
             entire external revenue model.
      OQ-02  Actual internal per-verification and refresh cost base.
             Owner: Compliance COO + Finance. Due: Phase 0 day 60.
      OQ-03  Is the existing record-linkage inference service reusable for
             identity resolution, and what is the feature-set delta?
             Owner: engineering. Due: Phase 0 day 45. Drives the 9 vs 18
             month estimate.
      OQ-04  Which agent protocol adapters ship first — AP2, TAP-compatible,
             or Amex-native? Owner: GMNS strategy. Due: before Phase 3.
