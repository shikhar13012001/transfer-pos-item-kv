# 04 — Data Model

Status: Draft for Phase 0 review

---

## 4.1 Credential schemas

### 4.1.1 Member Passport

    {
      "iss": "https://id.americanexpress.com",
      "vct": "https://id.americanexpress.com/vc/member-passport/v1",
      "iat": 1757376000,
      "exp": 1788912000,
      "cnf": { "jwk": { "kty":"EC","crv":"P-256","x":"...","y":"..." } },
      "sub": "<pairwise subject_ref>",
      "status": {
        "status_list": { "idx": 91274,
                         "uri": "https://id.americanexpress.com/status/mp/2026-09" }
      },
      "_sd": [ "<hash>", "<hash>", ... ],
      "_sd_alg": "sha-256"
    }

Permitted disclosable claims — this list is the schema, and the schema is
the privacy control. Adding a field requires a privacy review, not a PR.

    verified_to_amex_standard      bool
    identity_assurance_level       "substantial" | "high"
    age_over_18                    bool
    age_over_21                    bool
    age_over_65                    bool
    jurisdiction                   ISO 3166-1 alpha-2
    jurisdiction_in_eea            bool
    sanctions_screened_at          date (day precision, never time)
    pep_status                     "clear" | "match" | "review"
    account_tenure_months_over_12  bool
    account_tenure_months_over_24  bool
    account_tenure_months_over_60  bool

Forbidden — no field may exist for these, and schema validation at mint
must reject a payload containing any key matching these patterns:

    name, given_name, family_name, full_name
    birth_date, dob, date_of_birth
    address, street, postal_code, city
    document_number, passport_number, national_id, ssn, tax_id
    email, phone, msisdn
    card_number, pan, account_number
    photo, image, portrait, document_image

Note the pattern: every age and tenure claim is a predicate. There is no
field that carries a birthdate or an account open date, so no
implementation error can leak one. jurisdiction carries a country only,
never a subdivision or postcode, because a postcode plus two other claims
is re-identifying.

sanctions_screened_at is day precision because minute precision, combined
with a screening batch schedule, narrows the subject population.

### 4.1.2 Business Passport

    vct: ".../vc/business-passport/v1"

    entity_verified                bool
    entity_jurisdiction            ISO 3166-1 alpha-2
    entity_type                    "corporation" | "llc" | "partnership" |
                                   "sole_trader" | "other"
    beneficial_ownership_resolved  bool
    beneficial_owners_screened     bool
    sanctions_screened_at          date
    pep_exposure                   "none" | "indirect" | "direct"
    trading_since_year             integer (year only)
    amex_relationship_over_24m     bool
    merchant_in_good_standing      bool

Held by an authorised officer, key-bound to that officer's device.
Officer change requires re-issuance. [OPEN: multi-officer holding, whether
an entity may have several concurrently valid credentials. Owner:
Commercial product. Due: Phase 2 design.]

### 4.1.3 Agent Mandate

    {
      "iss": "https://id.americanexpress.com",
      "vct": ".../vc/agent-mandate/v1",
      "mandate_id": "amd_01J9X7QW3E",
      "principal_ref": "<pairwise ref of the Member/Business Passport>",
      "principal_assurance": "high",
      "agent": {
        "agent_id": "did:web:agent.example.com:runtime:7f2a",
        "agent_key": { "kty":"EC","crv":"P-256","x":"...","y":"..." },
        "operator": "Example AI Inc.",
        "operator_verified": true
      },
      "scope": {
        "max_amount_per_txn":    { "value": 50000,  "currency": "USD" },
        "max_amount_cumulative": { "value": 200000, "currency": "USD" },
        "merchant_categories":   ["5812","4722"],
        "merchant_allowlist":    [],
        "merchant_blocklist":    [],
        "geo":                   ["US","CA"],
        "max_txn_count":         20,
        "single_use":            false
      },
      "consent": {
        "captured_at":   "2026-09-09T10:14:22Z",
        "method":        "biometric_device_confirmation",
        "human_present": true,
        "consent_ref":   "cns_01J9X7..."
      },
      "nbf": 1757376862,
      "exp": 1759968862
    }

consent.human_present is the field carrying the liability. It asserts a
verified human, on a device bound to a valid passport, affirmatively
approved this exact scope. It is what a merchant buys and what determines
chargeback treatment. It may only be set true by a flow that captured a
biometric device confirmation — never by an API caller, never by config.

---

## 4.2 Persistence — DDL

### 4.2.1 Issuance registry (PostgreSQL, atp-issuer)

Note what is absent: no PII column exists. The registry records that a
credential was issued to an internal subject reference, not who that is.

    CREATE TABLE credential (
      credential_id        TEXT PRIMARY KEY,
      credential_type      TEXT NOT NULL
                             CHECK (credential_type IN
                               ('member-passport','business-passport')),
      internal_subject_id  TEXT NOT NULL,
      holder_key_thumbprint TEXT NOT NULL,
      device_attestation_ref TEXT NOT NULL,
      status_list_uri      TEXT NOT NULL,
      status_list_idx      INTEGER NOT NULL,
      assurance_level      TEXT NOT NULL,
      resolution_confidence NUMERIC(5,4),
      issued_at            TIMESTAMPTZ NOT NULL,
      expires_at           TIMESTAMPTZ NOT NULL,
      revoked_at           TIMESTAMPTZ,
      revocation_reason    TEXT,
      issuer_key_id        TEXT NOT NULL,
      schema_version       TEXT NOT NULL,
      region               TEXT NOT NULL,
      UNIQUE (status_list_uri, status_list_idx)
    );

    CREATE INDEX idx_cred_subject ON credential (internal_subject_id)
      WHERE revoked_at IS NULL;
    CREATE INDEX idx_cred_expiry  ON credential (expires_at)
      WHERE revoked_at IS NULL;
    CREATE INDEX idx_cred_thumb   ON credential (holder_key_thumbprint);

    -- issuer_key_id is required for key rotation: on compromise, the
    -- revocation playbook selects by issuer_key_id and bulk-revokes.

### 4.2.2 Mandate store (PostgreSQL + Redis, atp-mandate)

    CREATE TABLE mandate (
      mandate_id           TEXT PRIMARY KEY,
      principal_credential_id TEXT NOT NULL REFERENCES credential(credential_id),
      agent_id             TEXT NOT NULL,
      agent_key_thumbprint TEXT NOT NULL,
      operator             TEXT NOT NULL,
      scope                JSONB NOT NULL,
      consent_ref          TEXT NOT NULL,
      human_present        BOOLEAN NOT NULL,
      cumulative_minor     BIGINT NOT NULL DEFAULT 0,
      cumulative_currency  CHAR(3) NOT NULL,
      txn_count            INTEGER NOT NULL DEFAULT 0,
      not_before           TIMESTAMPTZ NOT NULL,
      expires_at           TIMESTAMPTZ NOT NULL,
      revoked_at           TIMESTAMPTZ,
      created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
    );

    CREATE INDEX idx_mandate_principal ON mandate (principal_credential_id);
    CREATE INDEX idx_mandate_agent     ON mandate (agent_id)
      WHERE revoked_at IS NULL;

    CREATE TABLE mandate_decision (
      decision_ref    TEXT PRIMARY KEY,
      mandate_id      TEXT NOT NULL,
      txn_ref         TEXT NOT NULL,
      authorized      BOOLEAN NOT NULL,
      decline_reason  TEXT,
      amount_minor    BIGINT NOT NULL,
      currency        CHAR(3) NOT NULL,
      merchant_ref    TEXT,
      mcc             TEXT,
      decided_at      TIMESTAMPTZ NOT NULL,
      liability_shift TEXT NOT NULL,
      UNIQUE (mandate_id, txn_ref)   -- replay protection
    );

    -- The UNIQUE constraint on (mandate_id, txn_ref) is the replay control.
    -- It is a database constraint rather than application logic
    -- deliberately: it cannot be bypassed by a code path that forgets.

Redis holds hot mandate state keyed by mandate_id with write-behind to
PostgreSQL. TTL matches mandate expiry. On cache miss, load from Postgres
and warm — a miss costs latency, never correctness.

### 4.2.3 Relying party registry

    CREATE TABLE relying_party (
      rp_id                TEXT PRIMARY KEY,
      legal_name           TEXT NOT NULL,
      client_cert_subject  TEXT NOT NULL UNIQUE,
      entitled_types       TEXT[] NOT NULL,
      entitled_claims      TEXT[] NOT NULL,
      warranty_tier        TEXT NOT NULL DEFAULT 'none',
      pairwise_salt_version INTEGER NOT NULL DEFAULT 1,
      rate_limit_rps       INTEGER NOT NULL,
      webhook_url          TEXT,
      status               TEXT NOT NULL DEFAULT 'active',
      onboarded_at         TIMESTAMPTZ NOT NULL
    );

    entitled_claims is the enforcement point for data minimisation. An RP
    that has no lawful basis to know jurisdiction cannot request it, and
    the gateway rejects with 403 before the wallet ever sees the request.
    This is a regulatory control, not a commercial one — see
    07-compliance section 7.2.

---

## 4.3 Event contracts (Kafka)

Topic naming: atp.events.<domain>.v<n>

    atp.events.issuance.v1
      credential.issued        credential_id, type, assurance, region, ts
      credential.revoked       credential_id, reason_class, ts
      credential.expired       credential_id, ts

    atp.events.verification.v1
      presentation.verified    billing_ref, rp_id, type, outcome,
                               claims_disclosed[], warranty_tier, ts
      presentation.failed      billing_ref, rp_id, outcome, ts

    atp.events.mandate.v1
      mandate.issued           mandate_id, principal_ref, agent_id, scope, ts
      mandate.decided          decision_ref, mandate_id, authorized,
                               decline_reason, amount, ts
      mandate.revoked          mandate_id, reason_class, ts

    atp.events.risk.v1
      risk.score_changed       internal_subject_id, old_band, new_band, ts
      risk.revocation_advised  internal_subject_id, model_version, score, ts

Rules that apply to every event on every topic:

    1  No PII in any event payload. Ever. Enforced by a schema-registry
       validator in CI, same forbidden-key list as 4.1.1.
    2  No raw claim values in verification events — record which claims
       were disclosed, never what they said. Billing needs the count;
       nobody needs the content.
    3  Events are for audit, billing and risk. They are not an integration
       API for other Amex teams. Anything needing ATP data calls the API.
    4  Retention per existing Amex policy. Because no event contains PII,
       there is no erasure obligation against the event log — which is the
       whole reason for rule 1.

---

## 4.4 Data classification

    Credential payload            no PII by construction
    Issuance registry             pseudonymous; internal_subject_id is a
                                  reference, resolvable only inside Amex
    Mandate store                 pseudonymous plus transaction metadata
    Event streams                 pseudonymous
    Document images               PII. Never persisted server-side on the
                                  common path. Where transient
                                  server-side processing is unavoidable,
                                  memory only, no disk, no logs, deleted
                                  on completion of the issuance
                                  transaction.
    Pairwise secret               HSM-held. Compromise permits cross-RP
                                  correlation of every subject_ref ever
                                  issued and is a Sev-1 with a mandatory
                                  full re-issuance. Treated with the same
                                  controls as the issuer signing key.
