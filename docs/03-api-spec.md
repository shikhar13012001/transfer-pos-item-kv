# 03 — API Specification

Status: Draft for Phase 0 review
Base URL: https://id.americanexpress.com
Auth: mTLS client certificate + OAuth2 client credentials
Versioning: URI path (/v1). Breaking changes require a new major path.

---

## 3.1 Conventions

    Content-Type          application/json; charset=utf-8
    Timestamps            RFC 3339, UTC, always with Z
    Money                 { "value": <integer minor units>, "currency": <ISO 4217> }
                          never a float, anywhere
    Idempotency           Idempotency-Key header required on all POSTs that
                          create state; 24h retention; replay returns the
                          original response with Idempotency-Replayed: true
    Correlation           X-Request-Id echoed on every response
    Rate limits           per RP, returned in X-RateLimit-* headers

---

## 3.2 Endpoint summary

    Issuance
      POST   /v1/credentials/issue              mint a credential
      POST   /v1/credentials/{id}/revoke        revoke
      GET    /v1/credentials/{id}/status        issuance registry lookup

    Presentation and verification
      POST   /v1/presentation/request           build an OpenID4VP request
      POST   /v1/verify                         verify a presentation

    Agent mandates
      POST   /v1/mandate/issue                  mint a mandate
      POST   /v1/mandate/authorize              authorise a transaction
      POST   /v1/mandate/{id}/revoke            revoke a mandate
      GET    /v1/mandate/{id}                   mandate state and remaining scope

    Status
      GET    /status/{type}/{period}            signed Bitstring Status List
                                                public, CDN cached, no auth

    Relying party admin
      GET    /v1/rp/entitlements                what this RP may request
      GET    /v1/rp/usage                       billable verifications

---

## 3.3 POST /v1/verify

The primary billable endpoint.

Request

    POST /v1/verify
    Idempotency-Key: 8f3a...
    X-Request-Id: 01J9X8...

    {
      "presentation": "<SD-JWT VP with KB-JWT>",
      "nonce": "n-0S6_WzA2Mj",
      "audience": "rp_acme_bank",
      "required_claims": ["age_over_18", "jurisdiction"],
      "optional_claims": ["account_tenure_months_over_24"],
      "warranty_tier": "standard"
    }

    presentation      required. The VP as received from the wallet.
    nonce             required. Must match the nonce the RP issued in the
                      presentation request. Single use. 300s max age.
    audience          required. Must equal the RP resolved from the client
                      certificate. Mismatch is a hard failure — this is the
                      control that prevents one RP replaying another's
                      presentation.
    required_claims   claims the RP needs. Absence of any one produces
                      outcome claim_not_present.
    optional_claims   returned if disclosed, silently omitted if not.
    warranty_tier     none | standard | enhanced. Affects price and the
                      coverage cap. [OPEN pending Legal, PRD OQ-01]

Response 200 — verified

    {
      "outcome": "verified",
      "issuer": "americanexpress",
      "credential_type": "member-passport",
      "assurance_level": "high",
      "claims": {
        "age_over_18": true,
        "jurisdiction": "DE",
        "account_tenure_months_over_24": true
      },
      "subject_ref": "u_7Kx2mQ9vLp4nR8sT1wY6zB3cF5hJ0dG",
      "status": "active",
      "status_checked_at": "2026-09-09T10:15:03Z",
      "credential_issued_at": "2026-03-02T08:11:00Z",
      "credential_expires_at": "2027-03-02T08:11:00Z",
      "warranty": {
        "tier": "standard",
        "coverage_ref": "wty_01J9X8QR7M",
        "cap": { "value": 2500000, "currency": "USD" }
      },
      "billing_ref": "vrf_01J9X8QR7M4TDNK",
      "request_id": "01J9X8..."
    }

    subject_ref is pairwise. It is stable for this RP forever and differs
    for every other RP. Safe to use as a primary key. Not safe to share.

Response 200 — not verified

Non-verification is a 200 with an outcome, not a 4xx. This is deliberate:
these are business outcomes, not protocol errors, and integrators must
handle all four distinctly.

    { "outcome": "invalid_signature",     ... }
    { "outcome": "revoked",               "revoked_at": "...", "reason_code": "..." }
    { "outcome": "holder_binding_failed", ... }
    { "outcome": "claim_not_present",     "missing": ["jurisdiction"] }
    { "outcome": "expired",               "expired_at": "..." }

Outcome semantics — the integration guide must state these plainly:

    verified              proceed
    invalid_signature     credential is not authentic. Treat as forgery.
                          Log and alert. Do not retry.
    revoked               authentic but no longer valid. Do not proceed.
                          Route to full verification.
    holder_binding_failed authentic credential presented by someone who does
                          not hold the private key. Likely theft. Alert.
    expired               authentic, past expiry. Route to re-issuance,
                          not to full re-verification.
    claim_not_present     NOT AN ERROR. The holder declined to disclose.
                          The privacy model working as designed. The SDK
                          must not surface this as a failure and the
                          integration guide must say so, or integrators
                          will pressure users into over-disclosure.

Errors — 4xx/5xx are protocol failures only

    400  malformed_request      body failed schema validation
    401  authentication_failed  mTLS or OAuth failure
    403  entitlement_denied     RP not entitled to a requested claim
    409  nonce_replayed         nonce already consumed
    422  audience_mismatch      audience does not match the client cert
    429  rate_limited           Retry-After present
    503  service_unavailable    RP must fail open per ADR-005

---

## 3.4 POST /v1/mandate/authorize

The 40ms path. Inline in payment authorisation.

Request

    {
      "mandate": "<agent mandate VC>",
      "agent_assertion": "<JWS by agent key over sha256(txn_canonical)>",
      "transaction": {
        "amount": { "value": 14250, "currency": "USD" },
        "merchant_ref": "mrc_8821094",
        "mcc": "5812",
        "country": "US",
        "txn_ref": "txn_01J9XA..."
      }
    }

Response 200

    {
      "authorized": true,
      "decision_ref": "dec_01J9XA7B2C",
      "liability_shift": "issuer",
      "scope_remaining": {
        "cumulative": { "value": 185750, "currency": "USD" },
        "expires_at": "2026-10-09T10:14:22Z"
      },
      "risk_band": "low",
      "principal_assurance": "high"
    }

Decline

    {
      "authorized": false,
      "decision_ref": "dec_01J9XA7B2D",
      "decline_reason": "scope_exceeded",
      "detail": "amount exceeds max_amount_per_txn",
      "liability_shift": "none"
    }

    decline_reason values
      scope_exceeded        amount, category, geo or cumulative cap breached
      mandate_expired       past exp
      mandate_revoked       revoked by principal or by risk
      agent_signature_bad   assertion does not verify against agent key
      principal_revoked     the underlying Member Passport is revoked
      replay_detected       txn_ref already decided

Contract notes for the auth platform team:
  - This call must be wrapped in a 40ms timeout with fail-open on timeout.
    A timeout is not a decline; it is an unattested transaction.
  - decision_ref must be persisted on the transaction record. It is the
    evidence trail for any subsequent dispute and for liability assignment.
  - Cumulative spend is eventually consistent. Over-spend bounded at one
    transaction per mandate per consistency window. Accepted risk, sized
    and documented in 05-security section 5.8.

---

## 3.5 POST /v1/credentials/issue

Called by the Amex app, not by relying parties.

    {
      "subject": { "type": "cardmember", "ref": "<internal ref>" },
      "credential_type": "member-passport",
      "holder_key": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." },
      "attestation": "<device attestation blob>",
      "requested_claims": ["age_over_18", "jurisdiction", "sanctions_screened_at"],
      "consent_ref": "cns_01J9X7..."
    }

Response 201

    {
      "credential_id": "mp_01J9X7QW3E",
      "credential": "<SD-JWT VC>",
      "status_list": { "uri": "...", "idx": 91274 },
      "issued_at": "2026-09-09T10:12:00Z",
      "expires_at": "2027-09-09T10:12:00Z",
      "resolution_confidence": 0.997
    }

    holder_key must be accompanied by a valid platform device attestation
    (Apple App Attest / Android Key Attestation) proving the key was
    generated in hardware and is non-exportable. Issuance without a valid
    attestation is rejected — this is the control behind FR-05 and there is
    no override path.

    resolution_confidence is the calibrated entity-resolution score.
    Issuance threshold is a risk-policy parameter, not a model constant.
    See 06-ml section 6.2.

---

## 3.6 GET /status/{type}/{period}

Public, unauthenticated, CDN-cached. This is what makes revocation checking
cheap enough to do on every presentation.

    GET /status/mp/2026-09

    200 OK
    Content-Type: application/statuslist+jwt
    Cache-Control: public, max-age=60

    <signed JWT containing a gzip+base64url Bitstring Status List>

    max-age 60 is what delivers the 60s propagation guarantee in FR-13.
    Do not raise it without a corresponding change to that requirement.

---

## 3.7 SDK expectations

Ship JVM, JavaScript and Python. Each must:

    - verify presentations locally where possible, calling /v1/verify only
      when the RP wants the warranty attachment. Local verification needs
      only the issuer JWKS and the status list, both public. This keeps
      the free tier genuinely free and the paid tier clearly differentiated.
    - expose the five outcomes as a sealed type / enum, never as a boolean.
      A boolean API guarantees integrators collapse claim_not_present into
      failure.
    - default to fail-open on transport error, with a loud log line.
    - never expose a method that returns raw disclosed claims without the
      outcome attached.

---

## 3.8 Webhooks

RPs may subscribe to revocation events for subjects they have verified.

    POST <rp endpoint>
    X-ATP-Signature: <JWS over body>

    {
      "event": "credential.revoked",
      "subject_ref": "u_7Kx2mQ9vLp4nR8sT1wY6zB3cF5hJ0dG",
      "occurred_at": "2026-09-14T02:31:00Z",
      "reason_class": "risk"
    }

    subject_ref is the pairwise identifier for that RP only.
    reason_class is coarse — risk | member_request | expiry | fraud —
    deliberately. A precise reason code would leak information about the
    member to the RP.
