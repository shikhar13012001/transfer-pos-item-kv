# 05 — Security and Threat Model

Status: Draft for Phase 0 review
Review required from: Information Security, Fraud Risk, Legal

---

## 5.1 Trust boundaries

    B1  Device ↔ ATP
        The holder's device is not trusted. It is attested. Every claim
        about the device (key in hardware, key non-exportable, app not
        tampered) must come from a platform attestation, never from the
        app's own assertion.

    B2  Relying party ↔ ATP
        RPs are authenticated but not trusted. An RP may be compromised.
        Entitlements limit what a compromised RP can request; pairwise
        identifiers limit what it can correlate.

    B3  Agent ↔ ATP
        Agents are the least trusted principal in the system. An agent key
        is assumed to be stealable. Everything an agent can do must be
        bounded by a mandate a human explicitly approved.

    B4  ATP ↔ systems of record
        Internal, but ATP reads PII and must never emit it. The boundary is
        enforced by schema validation on the way out, not by code review.

    B5  ATP ↔ HSM
        Signing key material never crosses this boundary. ATP sends a
        digest and receives a signature.

---

## 5.2 STRIDE

    SPOOFING
      T1  Attacker presents a stolen credential
          → device key binding; KB-JWT over a fresh nonce; the credential
            alone is useless without the enclave-held private key
      T2  Attacker impersonates a relying party
          → mTLS client certificates; audience binding on every
            presentation; audience mismatch is a hard failure
      T3  Synthetic identity obtains a genuine credential at enrolment
          → THE critical threat, see 5.4
      T4  Attacker impersonates an agent
          → agent signature over the transaction hash; mandate binds a
            specific agent public key

    TAMPERING
      T5  Modified credential payload
          → issuer signature over the whole SD-JWT
      T6  Modified transaction in an agent flow
          → agent assertion signs sha256 of the canonical transaction; any
            field change invalidates it
      T7  Status list manipulation
          → status lists are signed JWTs; verifiers check the signature,
            not just the bit

    REPUDIATION
      T8  Member denies authorising an agent
          → consent_ref links to a biometric device confirmation; the
            mandate records method and human_present; decision_ref is
            persisted on the transaction
      T9  Amex denies issuing a credential
          → append-only audit log; issuer key transparency log

    INFORMATION DISCLOSURE
      T10 Credential leaks PII
          → no PII field exists in the schema; forbidden-key validation at
            mint and in CI; predicates not values
      T11 Two RPs correlate the same person
          → pairwise identifiers (ADR-004)
      T12 An RP over-collects
          → entitlements enforced at the gateway before the wallet sees
            the request
      T13 Timing or metadata leakage identifies a subject
          → day-precision timestamps on screening claims; constant-time
            comparison on all secret material
      T14 Ledger leaks PII permanently
          → no ledger before Phase 4, and Phase 4 anchors status pointers
            only. Non-negotiable.

    DENIAL OF SERVICE
      T15 Attacker DoSes ATP to strip attestation and liability shift
          → fail-open is the correct behaviour (ADR-005) but is itself the
            attack surface. Mitigations: regional independence, edge-cached
            status lists, and a documented Risk position that sustained
            unavailability raises manual-review thresholds rather than
            silently accepting everything. This is an accepted, named risk.
      T16 Verification flood from a compromised RP
          → per-RP rate limits, anomaly alerting on verification volume

    ELEVATION OF PRIVILEGE
      T17 RP requests claims beyond entitlement
          → 403 at gateway
      T18 Agent exceeds mandate scope
          → scope evaluated server-side on every transaction; the agent
            never self-asserts its limits
      T19 Issuer key compromise
          → HSM custody, rotation schedule, bulk revocation by
            issuer_key_id, rehearsed playbook (08-ops section 8.5)

---

## 5.3 Cryptography

    Issuer signature        ES256 (ECDSA P-256 + SHA-256)
    Holder key binding      ES256, key generated in Secure Enclave /
                            StrongBox, non-exportable
    SD-JWT claim digest     SHA-256, per-claim random salt ≥ 128 bits
    Pairwise derivation     HMAC-SHA256 with HSM-held secret
    Transport               TLS 1.3, mTLS for RP traffic
    Status list             signed JWT, gzip + base64url bitstring

    Explicitly not used
      No custom cryptography anywhere.
      No BBS+ before ratification (ADR-001).
      No encryption of PII into a credential as a substitute for
        omitting it. Encrypted PII in a long-lived credential is PII with
        a delay fuse.

    Salt requirement: a per-claim salt below 128 bits, or a salt derived
    from claim content, permits dictionary recovery of undisclosed claims
    from their digests. Boolean claims are the acute case — an unsalted
    hash of "true" is trivially recovered. This is the single most likely
    implementation error in the whole build and belongs in the code review
    checklist.

---

## 5.4 Enrolment integrity — the critical threat

If a synthetic identity obtains a Member Passport, Amex has warranted a
fiction to every relying party in the network simultaneously, and the
warranty makes that a direct financial liability rather than an
embarrassment. Every other control in this document assumes enrolment
integrity holds.

    Layered controls at enrolment
      L1  Existing cardmembers skip document capture entirely, inheriting
          the assurance of the original KYC. This is a large advantage: the
          highest-risk path is only exercised for net-new customers.
      L2  Document forensics — print-noise residuals, font consistency,
          splice and edge artifacts, microprint, metadata provenance
      L3  Presentation attack detection — passive and active liveness
      L4  Injection and virtual-camera detection — increasingly the harder
          problem than liveness itself; a perfect liveness model is
          bypassed entirely by a synthetic feed injected below the camera
          API
      L5  Entity resolution against the existing customer graph — a
          synthetic identity that resolves to nothing, or to too many
          things, is a signal in itself
      L6  Sanctions and PEP screening
      L7  Device attestation — hardware-backed, non-exportable key required

    No single layer is trusted. Assurance level "high" requires all seven.
    "substantial" is available with a documented subset for markets where
    a control is unavailable.

    Standing requirement: an adversarial red team with a recurring budget,
    testing L2–L4 against current-generation generative models on a
    quarterly cadence. Deepfake capability improves faster than any model
    refresh cycle that is planned annually. A control tested once at launch
    is a control that expires.

---

## 5.5 Key management

    Issuer signing keys
      Custody              existing Amex HSM, non-exportable
      Separation           distinct key per credential type AND per region.
                           Blast radius of a compromise is one type in one
                           region, not the whole platform.
      Rotation             every 12 months, plus emergency rotation
      Overlap              old key remains valid for verification until the
                           last credential it signed expires; new issuance
                           uses the new key immediately
      Publication          JWKS at a stable well-known URI, with key IDs
      Transparency         append-only log of issuer keys, so an RP can
                           detect a key that appeared without announcement

    Pairwise secret
      Same custody and controls as the signing key. Compromise permits
      retroactive cross-RP correlation of every subject ever issued.
      Rotation invalidates every subject_ref and forces full re-issuance,
      so rotation is an emergency procedure, not a scheduled one — which
      means the controls must be strong enough that scheduled rotation is
      not needed.

    Holder keys
      Generated on-device, never transmitted, never escrowed. There is no
      recovery path. Device loss means re-issuance, not key recovery. This
      is correct and must be stated in the product copy so support does not
      promise otherwise.

---

## 5.6 Privacy controls, mapped to the mechanism that enforces them

Stated as mechanisms rather than intentions, because an intention is not a
control.

    Data minimisation      predicates not values; the schema has no field
                           for a birthdate, so no bug can leak one
    Purpose limitation     per-RP entitlements, enforced at the gateway
    Unlinkability          pairwise identifiers derived per RP
    Consent                per-claim, in the wallet, showing who is asking
    Erasure                no PII in credentials, events or the registry,
                           so there is nothing in ATP to erase; erasure
                           obligations remain against the existing systems
                           of record where they already apply
    Transparency           the asymmetry that Amex can internally re-link
                           pairwise identifiers MUST be disclosed in the
                           privacy notice. It is necessary for fraud
                           investigation and lawful process. Undisclosed,
                           it is a scandal waiting for a journalist.

---

## 5.7 ML-specific security

    Model evasion          adversarial testing of forgery and PAD models on
                           a quarterly cadence, against current generative
                           capability
    Model inversion        entity-resolution scores are never returned to a
                           relying party; resolution_confidence is exposed
                           only to internal issuance callers
    Poisoning              training data provenance controlled; no
                           production feedback loop retrains a model
                           without human review
    Drift                  monitored per model card (06-ml); drift beyond
                           threshold blocks issuance at "high" assurance
                           and falls back to "substantial" plus manual
                           review, rather than silently degrading
    Explainability         every model in a decisioning path meets model
                           risk management standards; a decline must be
                           explainable to the member and to a regulator

---

## 5.8 Accepted risks

Named explicitly so they are decisions rather than oversights. Each needs a
signature before Phase 1 ships.

    AR-01  Fail-open on ATP unavailability permits attestation stripping
           via DoS. Accepted because the alternative — declining
           transactions on an identity-service outage — is materially worse.
           Owner: Fraud Risk.

    AR-02  Cumulative mandate spend is eventually consistent. Bounded
           over-spend of one transaction per mandate per consistency
           window. Sized: with a p99 write-behind of 200ms and per-txn caps,
           maximum exposure per mandate is one max_amount_per_txn.
           Owner: Fraud Risk.

    AR-03  Repeat presentations to the same RP are linkable to each other
           (consequence of ADR-001). Cross-RP linkage is prevented; same-RP
           linkage is not, and an RP knowing its own returning user is the
           intended behaviour anyway. Owner: Privacy.

    AR-04  Amex can internally correlate pairwise identifiers. Necessary
           and disclosed. Owner: Privacy + Legal.

    AR-05  A credential inherits the assurance of the original KYC, which
           may be years old. Mitigated by continuous risk scoring and
           12-month expiry, not eliminated. Owner: Compliance.
