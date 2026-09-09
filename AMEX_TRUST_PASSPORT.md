# Amex Trust Passport
### Turning verification Amex has already paid for into a network revenue line

Internal platform name: Amex Trust Passport (ATP)
Relying-party mark: "Verified by Amex"
Consumer surface: extension of SafeKey

Author: Shikhar
Status: Proposal for review
Audience: Engineering, Risk, GMNS strategy


---

## 1. Executive summary

Amex has already run bank-grade KYC on roughly 87.6 million proprietary
cardmembers and underwritten identity on a merchant network now past 170
million locations. That verification work is a sunk cost, trapped inside
Amex, re-paid every year through periodic refresh, and re-done from scratch
by every other institution the same customer touches.

The proposal: make that verification portable. Issue a reusable,
privacy-preserving identity credential — a "passport" — that a cardmember,
a business, or an AI agent presents to any relying party, proving they were
verified to Amex standard without disclosing the underlying PII.

Three things make this urgent rather than interesting.

  1. Amex's own CEO has already named the problem. In the March 2026
     shareholder letter, Squeri wrote that agentic commerce makes
     "managing identity, authorization, fraud risk, and liability of
     paramount importance," and that the winners will go beyond basic
     payment functionality.

  2. Both competitors have already shipped. Visa launched Trusted Agent
     Protocol in October 2025 with Cloudflare. Mastercard shipped
     Verifiable Intent and folded it into Agent Pay in 2026. Amex has
     announced a merchant developer kit. We are behind on the layer the
     CEO called paramount.

  3. Regulation sets the clock. Under eIDAS 2.0, every EU member state
     must issue a digital identity wallet by 24 December 2026, and
     regulated private-sector firms — banking explicitly named — must
     accept it by late 2027. Amex will build wallet-acceptance
     infrastructure regardless. The only question is whether we build it
     as a compliance cost or as a product.

The strategic claim: Amex's closed loop, historically a distribution
disadvantage, is a decisive advantage in identity. Amex is the only major
network that holds first-party verified identity on both sides of the
transaction. Visa and Mastercard route transactions; the KYC lives at
thousands of issuing banks they do not control. They can verify a message.
They cannot attest to a person. Amex can.

The financial claim, modelled in Section 6: roughly $120–270M/year of
addressable internal cost takeout, a new attestation-fee line with
plausible $250M–850M/year potential at near-zero marginal cost, and
defence of an estimated ~$380M of discount revenue per 1% of billed
business that would otherwise route to networks that can service agent
identity.

The ask: a 90-day funded discovery, one squad, no external vendor
commitment. Details in Section 12.


---

## 2. The problem, stated properly

### 2.1 For the customer

A cardmember who applies for an Amex card, then a brokerage account, then a
neobank, then a business account, uploads the same passport four times and
waits four times. Industry onboarding abandonment in identity verification
flows runs around 34% — better than one in three prospects drops out before
finishing. Every one of those is a fully-loaded acquisition cost written off
at the last step.

### 2.2 For the institution

Each verification is re-purchased, never re-used. Published benchmarks:
retail document-plus-biometric checks run roughly $1–5 per customer at list,
closer to $4 all-in once re-verification, failed checks, PII storage and
compliance overhead are counted. Manual retail review runs $13–130 per case.
Commercial and institutional KYC is an order of magnitude worse — over half
of corporate and institutional banks spend $1,500–3,000 to complete a single
client KYC review, and one in five spend more than $3,000.

The recurring cost is the part usually missed in these business cases. KYC
is not a one-time event. Periodic refresh obligations mean a book of 87.6
million proprietary cardmembers generates tens of millions of re-reviews a
year forever, whether or not anything about the customer has changed.

### 2.3 For the system

Every institution that stores a copy of a customer's identity documents
becomes a honeypot. The industry's current architecture guarantees that a
single person's passport image sits in dozens of independently-breachable
stores. Reuse without copying is the only structural fix.

### 2.4 The new problem, which is the one that actually forces the decision

AI agents now initiate purchases. The acquirer at the far end has no way to
distinguish a legitimately delegated agent from a scripted attacker replaying
a stolen token. This is not theoretical: the x402 protocol processed on the
order of 165 million agent transactions in its first months, and Adobe
measured a ~4,700% year-over-year jump in generative-AI traffic to US retail
sites between July 2024 and July 2025.

Payment systems were built for a human pressing a button. When there is no
human pressing the button, liability has nowhere to land. Whoever supplies
the credential that says "a real, verified human authorised this agent,
within these limits" becomes the trust anchor for agent commerce — and
collects a fee on every transaction that needs one.

Visa and Mastercard understand this. Both have shipped. This is the window.


---

## 3. What we are building

One issuance and verification platform. Three credential products on top of
it. They share a single trust root, key hierarchy, revocation service and
audit spine — which is why this is one programme and not three.

### 3.1 Member Passport — the human credential

Who: proprietary cardmembers, already KYC'd.
What it proves: a set of assertions, disclosed selectively, one at a time.

    verified_to_amex_standard = true
    age_over(18) = true
    jurisdiction_in(EEA) = true
    sanctions_screened_within(30d) = true
    identity_assurance_level = high
    account_tenure_over(24m) = true

What it never carries: name, date of birth, address, document number,
document image. Not encrypted-on-chain. Not present at all.

The user proves a predicate. The relying party learns the answer to the
question it is legally required to ask, and nothing else. A bar checking
age learns "over 18" and does not learn the birthdate. A lender checking
jurisdiction learns "resident in Germany" and does not learn the street.

### 3.2 Business Passport — the KYB credential

Who: merchants on the network, Amex Business and Commercial customers.
What it proves: legal entity verified, beneficial ownership resolved,
sanctions and PEP screened, trading history attested, business age.

This is where the margin is. Commercial KYC at $1,500–3,000 per review is
the single most expensive identity operation in banking, it takes weeks, and
it is duplicated by every counterparty a business onboards with. A business
that has been through Amex Commercial underwriting has already cleared a
higher bar than most banks' KYB. Selling a portable attestation of that at
$100 against an incumbent cost of $2,000 is not a discount — it is a
different product category.

It also carries a benefit the consumer credential does not: beneficial
ownership resolution is the hardest, most manual part of KYB, and Amex has
already done it for its commercial book.

### 3.3 Agent Passport — Know Your Agent

Who: AI agents transacting on behalf of a cardmember or a business.
What it proves: a cryptographically signed mandate binding the agent to a
verified human principal, with scope.

    principal            = <Member Passport, pairwise pseudonym>
    agent_id             = <DID or registered agent key>
    max_amount           = 500.00 USD
    merchant_categories  = [travel, dining]
    valid_until          = 2026-10-01T00:00:00Z
    single_use           = false

The mandate travels with the transaction. Amex verifies it at authorisation.
A transaction carrying a valid mandate gets different fraud scoring,
different liability assignment, and different chargeback treatment than an
unattested agent transaction — which is exactly the model Mastercard has
already articulated, and exactly what merchants will pay to have.

Because Amex is closed-loop, we can do something neither competitor can: we
verify the agent's mandate *and* the merchant's Business Passport in the same
authorisation, from our own records, in one hop. Visa and Mastercard have to
federate that across issuers and acquirers they do not own.


---

## 4. What the client gets

### 4.1 Relying parties (banks, fintechs, brokerages, marketplaces)

  - Onboarding decision in seconds instead of days, at cents instead of
    dollars.
  - No PII received, therefore no PII to store, therefore that population is
    out of scope for a large part of their breach and GDPR surface. This is
    a liability transfer, and it is worth more to their CISO than the cost
    saving is to their CFO.
  - A named, solvent, regulated counterparty standing behind the assertion.
    Which brings us to the actual product.

### 4.2 The thing that makes this monetisable: the warranty

A free credential nobody stands behind is a curiosity. A credential with a
warranty is a product.

Amex's entire brand is standing behind transactions — dispute resolution,
backing the cardmember, "don't leave home without it." Extending "we stand
behind it" from payments to identity is brand-native in a way it would not be
for a pure technology vendor. Relying parties are not buying a signature
check. They are buying the right to point at Amex if the assertion was wrong.

That is why this can be priced at all. It is also the reason a startup cannot
replicate it: they can build the cryptography in a quarter and cannot supply
the balance sheet or the regulatory standing in a decade.

### 4.3 Merchants

  - Reduced onboarding friction on the acquiring side.
  - Ability to accept agent traffic with a liability model attached.
  - Fewer false declines on high-value card-not-present, because identity
    assurance travels with the transaction.

### 4.4 Cardmembers

  - Verify once. Reuse everywhere.
  - Documents stop being distributed to every counterparty.
  - A tangible premium benefit that justifies the fee — note that average fee
    per card rose 12% year over year to $131 in Q2 2026. Amex sells
    membership. "Your identity works everywhere and Amex stands behind it" is
    a membership benefit, not a compliance feature.


---

## 5. Why Amex, and why Amex first

### 5.1 The closed-loop argument

This is the core of the pitch and it should lead every leadership
conversation.

    Open loop (Visa, Mastercard)
      Network sees:      the transaction message
      Network KYC's:     nobody
      Identity lives at: thousands of issuing banks
      Can therefore:     verify a signature, route a claim
      Cannot:            attest to a person from first-party records

    Closed loop (Amex)
      Network sees:      the transaction
      Network KYC's:     the cardmember (issuer role)
      Network KYB's:     the merchant (acquirer role)
      Identity lives at: Amex
      Can therefore:     issue a first-party attestation for BOTH
                         counterparties, and verify both in one hop

Amex has spent forty years being told the closed loop is a coverage
disadvantage. In identity it inverts completely. Visa and Mastercard
structurally cannot issue this credential. They can only orchestrate other
people's.

### 5.2 The assets already in the building

  - ~87.6M proprietary cardmembers KYC'd to bank standard, ~155.1M
    cards-in-force total.
  - 170M+ merchant locations, with commercial underwriting already done on
    the business book.
  - Network partner relationships across ~110 countries — the distribution
    channel for a global credential already exists and is contractual.
  - Closed-loop transaction data: the strongest continuous-monitoring signal
    in the industry, and the thing that lets an attestation stay live rather
    than going stale the moment it is issued.
  - Existing production entity-resolution capability. The ML record-linkage
    system built for bill-payment adjudication — Fellegi-Sunter probabilistic
    linkage with calibrated LightGBM confidence scores, already deployed in
    shadow mode and already exposed as a plug-and-play inference API — is
    the same engine required for identity resolution across the customer
    graph. This is not a component we need to buy or build. It exists, it is
    calibrated, and it has a service contract other Amex applications
    already call. Reusing it is the single largest de-risking factor in the
    build plan.

### 5.3 The first-mover asymmetry

Identity credentials are a network good. The first credible bank-grade issuer
sets the schema, and every subsequent issuer either interoperates on those
terms or fights for a fragmented long tail. Amex does not need to win the
whole market. It needs to be the issuer whose schema the premium segment
standardises on — the same position it holds in payments.

Second place in an identity network is not half the value. It is a rounding
error.


---

## 6. The money

Every Amex figure below is from public filings or the March 2026 shareholder
letter. Every cost benchmark is a published industry figure and is cited as
such. Everything marked ASSUMPTION is a model input that Finance and the
Compliance COO must replace with internal actuals before this goes to a
funding committee. Nothing here is a forecast. It is a model with the
arithmetic shown so it can be argued with.

### 6.1 Baseline facts from filings

    Cards-in-force (Q2 2026)                     155.1M
    Proprietary cards-in-force                    87.6M
    Proprietary basic cards-in-force              67.5M
    Proprietary new cards acquired, H1 2026        6.1M  (~12.2M annualised)
    Billed business, Q1 2026                      $428.0B (~$1.71T annualised)
    Network volumes, Q1 2026                      $486.3B
    Discount revenue as % of billed business        2.23%
    Merchant locations                             170M+
    Average fee per card                            $131  (+12% YoY)

Derived: discount revenue ≈ $1.71T × 2.23% ≈ $38.2B/year.


### 6.2 Bucket A — internal cost takeout

Three cost pools.

A1. New-account consumer KYC
    ~12.2M new proprietary cards/year
    × $8 blended all-in per verification              [ASSUMPTION: industry
      all-in is ~$4 for automated retail; $8 reflects manual-review
      fallback, failed checks, storage and comp overhead at Amex scale]
    = ~$98M/year

    Reusable-credential effect: repeat and multi-product journeys collapse
    to a signature check at roughly $0.30. If 45% of new-account journeys
    are re-verifications of someone Amex or a partner already verified:
    ~$43M/year addressable, ~$41M/year recovered.

A2. Periodic KYC refresh — the large, invisible pool
    Regulated refresh cycles (roughly 1yr high / 3yr medium / 5yr low risk)
    across 87.6M proprietary cardmembers imply ~20–25M refreshes/year.
    × $7 per refresh                                  [ASSUMPTION]
    = ~$140–175M/year

    A live credential with continuous monitoring replaces batch refresh with
    event-driven revalidation. Only changed or flagged identities get
    re-reviewed. Conservative 50% reduction: ~$70–88M/year.

A3. Commercial and merchant KYB
    Published commercial KYC cost: $1,500–3,000 per review, >50% of
    corporate and institutional banks.
    Assume 100K commercial/merchant reviews per year at $2,000
                                                      [ASSUMPTION — replace
      with GMNS and Commercial actuals; this is the number most likely to be
      badly wrong in both directions and the one worth measuring first]
    = ~$200M/year
    Automation plus reuse at 35% reduction: ~$70M/year.

    Bucket A total, base case:  ~$180–200M/year recurring
    Conservative floor:         ~$120M/year
    Upside if A3 volume is materially higher: ~$270M/year


### 6.3 Bucket B — new revenue, attestation as a service

This is a network business with near-zero marginal cost. Verifying an
attestation is an elliptic-curve signature check and a status-list lookup.
The cost of the millionth verification is approximately the cost of the
first, which is approximately zero. Every incremental dollar of attestation
fee is close to gross margin.

B1. Consumer verification fees
    Price $0.50–1.50 per verification against a relying-party incumbent cost
    of ~$4 all-in. Sold on liability transfer, not price.
    500M verifications/year × $1.00 = ~$500M/year        [ASSUMPTION on
      volume; requires partner distribution, see roadmap Phase 3]
    Conservative ramp, year 3: 150M × $0.80 ≈ $120M/year

B2. Business Passport fees
    Price $100 per business verification against an incumbent $1,500–3,000.
    1M verifications/year = ~$100M/year
    Conservative, year 3: 250K × $100 ≈ $25M/year

B3. Agent mandate fees
    Per-mandate issuance and per-verification fee on agent transactions.
    Deliberately not sized here — the volume base does not exist yet and any
    number would be invented. Sized instead as option value in 6.4.

B4. Warranty tier
    Priced separately above the base fee: a capped indemnity per assertion.
    This is the line that no competitor can copy and the one with the most
    pricing power. Requires Risk and Legal to set the cap before it can be
    modelled at all.

    Bucket B, year-3 conservative:  ~$145M/year
    Bucket B, mature base case:     ~$600M/year
    Bucket B, upside with warranty: ~$850M/year


### 6.4 Bucket C — revenue defence, and the reason this is not optional

This is the number that should carry the leadership conversation, because it
is a loss that happens by default if nothing is built.

Agent-initiated commerce routes to rails that can service agent identity.
Visa has TAP. Mastercard has Verifiable Intent inside Agent Pay. If an agent
checkout flow can complete on those rails and degrades or declines on Amex,
volume migrates — silently, at the routing layer, without any customer ever
choosing to leave.

    Annualised billed business (from Q1 2026)      ~$1.71T
    Discount revenue at 2.23%                      ~$38.2B/year

    Every 1% of billed business that migrates
    away from Amex rails                           ~$17.1B volume
    Discount revenue forgone                       ~$382M/year

One percent. That single point of share loss exceeds the entire cost of
building this platform by roughly two orders of magnitude. And share loss in
routing is sticky: once a merchant's agent integration is built against a
competitor's protocol, it does not come back for a price concession.

Add the softer defence:
  - Onboarding abandonment at ~34% industry-wide. Each point recovered on
    ~12.2M annual acquisitions, against Amex's fully-loaded acquisition cost,
    is material. Finance should size this with the real CAC number.
  - Fraud loss reduction from identity assurance travelling with high-value
    card-not-present transactions.
  - False-decline recovery, which is pure lost revenue today.


### 6.5 Summary

    Bucket A  internal cost takeout      $120–270M/year recurring
    Bucket B  attestation revenue        $145M/yr (yr 3) → $600–850M/yr
    Bucket C  revenue defended           ~$382M/year per 1% of volume

    Against a build cost that is a single-digit-millions discovery and a
    low-tens-of-millions platform. The asymmetry is not close.

The honest framing for the funding committee: Buckets A and C justify the
build on their own, using only internal savings and defence, with no
dependence on anyone else adopting anything. Bucket B is the upside that
makes it a business rather than a cost centre — and it is the one with real
execution risk, because it requires partner distribution Amex does not yet
have. Do not let the pitch rest on B.


---

## 7. Technical architecture

### 7.1 Three design constraints, stated before anything else

These are the constraints that kill bank identity-ledger projects. Getting
them right in the design document is what gets this past Risk and Legal.

**Constraint 1 — No PII goes on any ledger. Ever. Not encrypted.**
Encrypted PII on an immutable ledger is PII with a time bomb attached: the
key leaks, or the cipher ages, and the data is permanently public with no
deletion path. GDPR Article 17 erasure and an append-only ledger are
irreconcilable. The ledger holds revocable status pointers and schema
identifiers. Nothing else. Every credential's PII stays in Amex's existing
regulated data estate, under existing controls and existing retention
policy.

**Constraint 2 — The blockchain is optional and comes last.**
The original concept led with Ethereum Attestation Service. For Amex, an
Ethereum dependency in v1 is a non-starter and will kill the proposal in the
first Risk review. It is also unnecessary: W3C Verifiable Credentials with
SD-JWT selective disclosure deliver the entire consumer value proposition —
reuse, selective disclosure, no PII sharing — with zero distributed-ledger
dependency, using standards that regulators already recognise. The ledger
anchor is a Phase 4 option that adds cross-ecosystem verifiability for
counterparties who will not federate with an Amex endpoint. Design for it,
do not depend on it.

**Constraint 3 — Standards, not invention.**
Anything proprietary here is dead on arrival with eIDAS. Build to:

    W3C Verifiable Credentials Data Model 2.0     credential format
    SD-JWT VC (IETF)                              selective disclosure
    OpenID4VCI                                    issuance protocol
    OpenID4VP                                     presentation protocol
    ISO/IEC 18013-5 mDL                           mobile document interop
    Bitstring Status List                         revocation
    eIDAS 2.0 ARF                                 EU wallet compatibility

The payoff is direct: an ARF-compatible credential means the eIDAS
acceptance obligation landing on Amex in late 2027 is satisfied by the same
platform that generates the revenue, rather than by a separate compliance
programme. One build, two outcomes.


### 7.2 Service topology

Kotlin/Spring Boot services, consistent with the existing backend estate.

    ┌──────────────────────────────────────────────────────────┐
    │ CLIENT                                                    │
    │ Amex mobile app (wallet) │ Partner SDK │ Agent runtime    │
    └────────────────────────┬─────────────────────────────────┘
                             │ OpenID4VCI / OpenID4VP
    ┌────────────────────────▼─────────────────────────────────┐
    │ atp-gateway            mTLS, rate limit, relying-party    │
    │                        registry and entitlements          │
    └────────────────────────┬─────────────────────────────────┘
                             │
       ┌──────────────┬──────┴───────┬──────────────┬──────────┐
       ▼              ▼              ▼              ▼          ▼
    ┌────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌────────┐
    │ atp-   │  │ atp-     │  │ atp-      │  │ atp-     │  │ atp-   │
    │ issuer │  │ verifier │  │ revocation│  │ mandate  │  │ audit  │
    │        │  │          │  │           │  │ (agents) │  │        │
    └───┬────┘  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └───┬────┘
        │            │              │             │            │
        ▼            ▼              ▼             ▼            ▼
    ┌──────────────────────────────────────────────────────────┐
    │ atp-identity-core                                         │
    │  · entity resolution  (REUSES existing record-linkage     │
    │                        inference API — Fellegi-Sunter +   │
    │                        calibrated LightGBM)               │
    │  · document intelligence (VLM)                            │
    │  · presentation attack + injection detection              │
    │  · sanctions / PEP screening orchestration                │
    │  · continuous risk scoring (GNN on closed-loop graph)     │
    └────────────────────────┬─────────────────────────────────┘
                             ▼
    ┌──────────────────────────────────────────────────────────┐
    │ EXISTING AMEX SYSTEMS (systems of record — unchanged)      │
    │ KYC store │ card master │ merchant master │ AML platform  │
    └──────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────┐
    │ HSM / key management     issuer signing keys, rotation,   │
    │ (existing Amex HSM)      transparency log                 │
    └──────────────────────────────────────────────────────────┘

    PHASE 4 ONLY, OPTIONAL:
    ┌──────────────────────────────────────────────────────────┐
    │ atp-anchor → ledger: status pointers + schema IDs only    │
    └──────────────────────────────────────────────────────────┘

The critical property of this topology: systems of record do not change.
ATP reads from them and issues over them. No migration, no dual-write, no
data movement out of the regulated estate. That is what makes a 90-day
discovery credible.


### 7.3 Issuance flow

    1  Cardmember opts in, in the Amex app.
    2  atp-issuer pulls the existing verified identity record. For an
       existing cardmember there is NO document re-scan — they were already
       verified. This is the whole point and the biggest UX win available.
    3  atp-identity-core runs entity resolution to confirm the record maps
       to exactly one identity across the customer graph, returning a
       calibrated confidence score.
    4  Device generates a key pair in the hardware-backed keystore. Public
       key returns to the issuer. Private key never leaves the secure
       element — this is what makes the credential non-transferable.
    5  atp-issuer mints an SD-JWT VC: claims individually salted and hashed,
       key-bound to the device public key, signed by the Amex issuer key
       from the HSM.
    6  Credential stored in the app wallet. Status entry allocated in
       atp-revocation.
    7  Cardmember is done. Elapsed time for an existing member: seconds.

    For a NEW customer with no prior Amex relationship, steps 2–3 are
    replaced by the full document + liveness capture path in 7.5.

### 7.4 Presentation flow

    1  Relying party requests specific predicates via OpenID4VP.
       "age_over_18, jurisdiction_in(EEA)" — not "identity document."
    2  Wallet shows the cardmember exactly which assertions are requested
       and who is asking. Consent is explicit and per-assertion.
    3  Wallet discloses only the requested claims, signs a nonce with the
       device key to prove holder binding.
    4  Relying party verifies: issuer signature, device key binding,
       nonce freshness, status-list entry not revoked.
    5  Optional: relying party calls atp-verifier for the warranty-backed
       response and the liability attachment. This is the billable call.

    Pairwise pseudonymous identifiers: each relying party receives a
    different, unlinkable subject identifier for the same cardmember.
    Two relying parties comparing notes cannot determine they are looking at
    the same person. This is not a nice-to-have — without it, the credential
    becomes a cross-industry tracking key and both the regulator and the
    press will treat it as one.


---

## 8. Data models and contracts

### 8.1 Member Passport credential

Illustrative SD-JWT VC payload. Every claim in `_sd` is individually salted
and hashed, so the holder discloses each one independently.

    {
      "iss": "https://id.americanexpress.com",
      "vct": "https://id.americanexpress.com/vc/member-passport/v1",
      "iat": 1757376000,
      "exp": 1788912000,

      "cnf": {
        "jwk": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
      },

      "sub": "<pairwise pseudonym, per relying party>",

      "status": {
        "status_list": {
          "idx": 91274,
          "uri": "https://id.americanexpress.com/status/mp/2026-09"
        }
      },

      "_sd": [
        "<hash: verified_to_amex_standard = true>",
        "<hash: identity_assurance_level = high>",
        "<hash: age_over_18 = true>",
        "<hash: age_over_21 = true>",
        "<hash: jurisdiction = DE>",
        "<hash: sanctions_screened_at = 2026-09-01>",
        "<hash: pep_status = clear>",
        "<hash: account_tenure_months_over_24 = true>"
      ],
      "_sd_alg": "sha-256"
    }

Note what is absent: no name, no DOB, no address, no document number, no
image reference. There is no field in this schema that could carry them.
That is a design property, not a policy.

### 8.2 Agent mandate

    {
      "iss": "https://id.americanexpress.com",
      "vct": "https://id.americanexpress.com/vc/agent-mandate/v1",
      "mandate_id": "amd_01J9X7...",
      "principal_ref": "<pairwise pseudonym of Member Passport>",
      "principal_assurance": "high",
      "agent": {
        "agent_id": "did:web:agent.example.com:runtime:7f2a",
        "agent_key": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." },
        "operator": "Example AI Inc."
      },
      "scope": {
        "max_amount_per_txn":   { "value": 500.00, "currency": "USD" },
        "max_amount_cumulative":{ "value": 2000.00,"currency": "USD" },
        "merchant_categories":  ["5812", "4722"],
        "merchant_allowlist":   [],
        "geo":                  ["US", "CA"],
        "single_use":           false
      },
      "consent": {
        "captured_at":   "2026-09-09T10:14:22Z",
        "method":        "biometric_device_confirmation",
        "human_present": true
      },
      "nbf": 1757376862,
      "exp": 1759968862
    }

`consent.human_present` is the field that carries the liability. It asserts
that a verified human, holding a device bound to a Member Passport,
affirmatively approved this scope. That assertion is what a merchant is
buying and what determines chargeback treatment.

### 8.3 Verification API

Request:

    POST /v1/verify
    Authorization: mTLS client cert + OAuth2 client credentials
    Content-Type: application/json

    {
      "presentation": "<SD-JWT VP>",
      "nonce": "n-0S6_WzA2Mj",
      "required_claims": ["age_over_18", "jurisdiction"],
      "warranty_tier": "standard"
    }

Response:

    {
      "verified": true,
      "issuer": "americanexpress",
      "assurance_level": "high",
      "claims": {
        "age_over_18": true,
        "jurisdiction": "DE"
      },
      "subject_ref": "<pairwise pseudonym, stable for THIS relying party>",
      "status": "active",
      "status_checked_at": "2026-09-09T10:15:03Z",
      "warranty": {
        "tier": "standard",
        "coverage_ref": "wty_01J9X8...",
        "cap_usd": 25000
      },
      "billing_ref": "vrf_01J9X8..."
    }

Failure responses must distinguish four cases, because relying parties will
build different logic for each and collapsing them causes bad integrations:

    invalid_signature      credential is not authentic
    revoked                authentic but no longer valid
    holder_binding_failed  authentic credential, wrong holder — likely theft
    claim_not_present      holder declined to disclose a requested claim

`claim_not_present` is not an error. It is the privacy model working. The
SDK must not surface it as a failure or integrators will pressure users to
over-disclose.

### 8.4 Agent authorisation hook

    POST /v1/mandate/authorize

    {
      "mandate": "<agent mandate VC>",
      "agent_assertion": "<signed by agent key over txn hash + nonce>",
      "transaction": {
        "amount": 142.50,
        "currency": "USD",
        "merchant_id": "<Amex merchant ref>",
        "mcc": "5812"
      }
    }

    → { "authorized": true,
        "liability_shift": "issuer",
        "scope_remaining": { "cumulative_usd": 1857.50 },
        "risk_band": "low",
        "decision_ref": "dec_01J9X9..." }

This call sits inline in the authorisation path. Latency budget: p99 under
40ms. That constraint drives most of the caching and colocation decisions and
should be treated as a hard requirement, not a target.


---

## 9. AI and ML components

Five models. Two already exist inside Amex.

**9.1 Entity resolution — REUSE, do not build.**
The record-linkage system built for bill-payment adjudication is the correct
engine here. Fellegi-Sunter probabilistic linkage with calibrated LightGBM
confidence scoring, leakage-safe feature engineering, business-metric-first
evaluation, already running in shadow mode against live traffic and already
exposed as an inference API that other Amex applications call and act on by
score threshold.

Identity resolution for passport issuance is the same problem with a
different label set: given a customer record, determine whether it maps to
exactly one identity in the graph, with a calibrated confidence that a
downstream service can threshold on. The calibration work is the expensive
part and it is done.

Delta required: extend the feature set beyond name-pair features to include
document, device and address signals, and derive issuance-appropriate labels.
Weeks, not quarters. This should be stated explicitly in any funding
request — it is the difference between an 18-month platform and a 9-month
one.

**9.2 Document intelligence.**
Fine-tuned vision-language model for parsing global government IDs. Runs
on-device where hardware permits, so document images never transit the
network for the common path. Handles MRZ extraction, multi-script, rotation,
glare, partial occlusion. Only needed for net-new customers — existing
cardmembers skip this entirely, which is a very large cost advantage over
any greenfield competitor.

**9.3 Tamper and forgery detection.**
CNN over print-noise residuals, font-consistency features, edge and splice
artifacts, microprint integrity, and metadata provenance. Outputs a
calibrated fraud probability, not a binary — the threshold belongs to the
risk policy layer, not the model.

**9.4 Presentation attack and injection detection.**
Passive liveness (screen moiré, corneal reflection, rPPG, skin texture),
active challenge-response, and — increasingly the harder problem — virtual
camera and injection attack detection. Deepfake resistance at enrolment is
the single point where the entire trust chain can be poisoned. If a
synthetic identity gets a Member Passport, Amex has warranted a fiction to
every relying party in the network simultaneously. This model gets the
adversarial red-team budget.

**9.5 Continuous risk scoring.**
Graph neural network over the closed-loop transaction graph, updating a live
risk signal per identity. Feeds re-verification triggers and status
revocation. This is the component that converts a static, decaying
credential into a live one, and it is only possible because Amex sees both
sides of the transaction. It is the technical expression of the closed-loop
advantage.

A note on model governance: every one of these sits inside a regulated
decisioning path. Model risk management, explainability, bias testing and
challenger models are in scope from day one, not bolted on at productionise
time. Budget for it in the original plan.


---

## 10. Threat model and what kills this

    Threat                        Mitigation
    ─────────────────────────────────────────────────────────────────────
    Synthetic identity at         Layered PAD + injection detection;
    enrolment                     document forensics; entity resolution
                                  against the existing customer graph;
                                  adversarial red team with standing budget

    Credential theft / sharing    Hardware-backed device key binding;
                                  private key never exportable; nonce
                                  challenge per presentation

    Replay                        Per-presentation nonce, short-lived VP,
                                  audience binding to relying party

    Cross-relying-party           Pairwise pseudonymous subject
    correlation                   identifiers; no shared persistent ID

    Issuer key compromise         HSM custody, scheduled rotation, key
                                  transparency log, staged revocation
                                  playbook rehearsed before launch

    Coerced disclosure            Per-claim consent UI; predicate proofs so
                                  the underlying value is never disclosed
                                  at all

    Regulatory erasure conflict   No PII on any ledger. Status pointers
    (GDPR Art. 17)                only. PII stays in the existing regulated
                                  estate under existing retention policy.

    Amex attests something        The warranty cap, priced per tier. This
    false                         is a risk to be underwritten, not
                                  avoided. Underwriting risk is what Amex
                                  does.

### What actually kills this programme

Not the cryptography. The cryptography is standards work and is the easiest
part of the build. The realistic failure modes, in order of likelihood:

  1. **No relying-party distribution.** A credential nobody accepts is worth
     nothing regardless of how good it is. Bucket B is entirely dependent on
     this. Mitigation: Phase 1 and 2 deliver value with Amex as the only
     relying party — internal cost takeout needs no external adoption at all.
     Never let the business case depend on Phase 3.

  2. **Legal will not sign the warranty.** The warranty is the monetisation
     mechanism. If Legal and Risk will not underwrite any assertion at any
     cap, Bucket B collapses to a commodity signature-check business with no
     defensibility. Get a preliminary Legal position in the first 30 days,
     before engineering commits. This is the highest-leverage early
     conversation and it is not a technical one.

  3. **Standards fragmentation.** AP2, TAP, Verifiable Intent, OIDC-A, the
     IETF AIP track and the FIDO Agentic Authentication working group are all
     live and not yet converged. Mitigation: build the credential layer to
     W3C VC and treat competing agent protocols as pluggable transport
     adapters. Do not bet the architecture on one of them winning.

  4. **Scope creep into a ledger project.** The moment this becomes "the
     Amex blockchain initiative" it acquires a governance overhead that will
     outlive its funding. Phase 4 is optional and explicitly gated. Say this
     out loud in every steering meeting.


---

## 11. Roadmap

Each phase produces standalone value. No phase depends on the next being
funded. This structure is deliberate — it is what makes the programme
survivable across budget cycles.

    PHASE 0 — Discovery                                     90 days, 1 squad
      Legal and Risk position on warranty feasibility  ← START HERE, day 1
      Measure the real internal KYC cost base (replaces every ASSUMPTION
        in Section 6)
      Assess record-linkage reuse; scope the feature-set delta
      Credential schema v0, threat model review
      GATE: internal cost base confirmed ≥ $150M/yr addressable AND Legal
            has not ruled out warranty in principle

    PHASE 1 — Internal reuse                                    6–9 months
      Issue Member Passport to a cardmember cohort
      Amex is the ONLY relying party: reuse the credential across Amex
        product onboarding and periodic refresh
      Value: Bucket A2 + part of A1. No external dependency whatsoever.
      GATE: measured reduction in refresh cost and multi-product onboarding
            time

    PHASE 2 — eIDAS and Business Passport                        9–15 months
      ARF-compatible wallet acceptance — satisfies the late-2027 obligation
      Business Passport for the commercial and merchant book
      Value: Bucket A3 + compliance obligation retired by the revenue
        platform rather than a separate cost programme
      GATE: EU acceptance certified; KYB cycle time reduced

    PHASE 3 — Agent Passport and external relying parties       12–20 months
      Know Your Agent mandates live in the authorisation path
      Relying-party SDK, pricing, warranty tiers
      Value: Bucket B opens. Bucket C defended.
      GATE: signed relying parties, agent transaction volume on Amex rails

    PHASE 4 — Ledger anchor                            OPTIONAL, gated, TBD
      Status pointers and schema IDs anchored for counterparties who will
        not federate with an Amex endpoint
      Fund only on demonstrated external demand. Not before.

Phase 3 is where the competitive clock actually bites. Visa shipped TAP in
October 2025. If Phase 3 lands later than mid-2027, Amex is integrating into
an agent-commerce ecosystem whose identity conventions were set by two
competitors. That is a materially worse position than the one available now,
and it argues for running Phase 3's protocol work in parallel with Phase 1
rather than strictly after it.


---

## 12. The ask

**Fund Phase 0. One squad, 90 days. No vendor commitment, no architecture
lock-in, no public position.**

Three deliverables:

  1. A Legal and Risk position on whether Amex will underwrite an identity
     assertion at any cap. Binary answer. Everything commercial depends on
     it and nothing technical does — which is why it runs first and in
     parallel with everything else.

  2. The real internal number. Section 6 is built on published industry
     benchmarks because internal figures were not available to me. Replace
     every ASSUMPTION with the actual per-verification cost, actual refresh
     volume, and actual commercial KYB spend. If the addressable base comes
     in under $150M/year, the internal-savings case weakens and the
     programme should be re-argued on Bucket C alone — which is defensible,
     but is a different pitch and should be made honestly as one.

  3. A build-time estimate grounded in the record-linkage reuse assessment.
     If entity resolution is genuinely reusable, this is a 9-month platform.
     If it is not, it is an 18-month one, and the whole timing argument
     against Visa and Mastercard changes.

**What I am not asking for:** a blockchain, a public announcement, a
partnership, or a commitment to any of the three credential products.

**The one-line version for the steering deck:**

Amex has already paid for the most valuable identity dataset in payments and
currently monetises none of it. Competitors who cannot build this are
shipping substitutes. One percent of routed volume is worth more than the
entire programme costs.


---

## Appendix A — Sources

Amex figures: Form 10-Q Q1 2026 and Q2 2026; Form 10-K FY2025; Squeri
shareholder letter, March 2026 (170M+ merchant locations; agentic commerce
and identity remarks).

KYC cost benchmarks: Fenergo global survey of 1,000+ corporate and
institutional banking executives ($1,500–3,000 per client review; >$3,000 for
21%); 2026 vendor list-price analysis ($1–5 per retail check, ~$4 all-in);
manual retail review $13–130 per case; ~34% identity-verification onboarding
abandonment.

Regulatory: Regulation (EU) 2024/1183 (eIDAS 2.0) — member-state wallet
issuance by 24 December 2026, private-sector acceptance obligations for
banking from late 2027. GENIUS Act enacted 18 July 2025, effective December
2026 or 120 days after final rules; OCC and Treasury NPRMs issued 2026.

Competitive: Visa Trusted Agent Protocol (October 2025, with Cloudflare);
Mastercard Verifiable Intent and Agent Pay (2026); Google AP2 with x402
stablecoin extension; FIDO Alliance Agentic Authentication working group
(April 2026); OIDC-A and IETF AIP standards tracks.

Agent commerce volume: x402 ~165M agent transactions in first months of
operation; Adobe Analytics ~4,700% YoY growth in generative-AI traffic to US
retail sites, July 2024 to July 2025.


## Appendix B — Naming

Recommended: **Amex Trust Passport** internally, **"Verified by Amex"** as the
relying-party mark, delivered through **SafeKey** as the consumer surface —
reusing existing brand equity rather than launching a fourth identity brand.

Alternatives considered: AXP Verify, Amex ID, TrustPass, Membership Verified.

Avoid anything with "chain," "crypto," "Web3" or "zero-knowledge" in a
customer-facing or leadership-facing name. The cryptography is an
implementation detail and naming it invites the wrong review board.