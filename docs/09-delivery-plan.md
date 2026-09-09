# 09 — Delivery Plan

Status: Draft for Phase 0 review
Owner: Shikhar

---

## 9.1 Phasing principle

Every phase produces standalone value. No phase assumes the next is funded.
This is deliberate: identity programmes at banks die in budget cycles, and
the ones that survive are the ones that were already paying for themselves
when the cycle turned.

    Phase 0   Discovery                         90 days,     1 squad
    Phase 1   Internal reuse                    6–9 months,  2 squads
    Phase 2   eIDAS + Business Passport         9–15 months, 3 squads
    Phase 3   Agent Passport + external RPs     12–20 months, 3–4 squads
    Phase 4   Ledger anchor                     OPTIONAL, gated, unsized

Durations from Phase 1 onward assume ADR-007 holds — that entity resolution
is genuinely reusable. If Phase 0 finds it is not, add roughly 6 months
across Phases 1–2 and revisit the competitive timing argument, because it
changes materially.

---

## 9.2 Phase 0 — Discovery (90 days, 1 squad)

Three deliverables. Two of them are not engineering work, and they run from
day one in parallel rather than after the technical assessment.

    D1  Legal and Risk position on warranty feasibility        day 30
        Binary answer: will Amex underwrite an identity assertion at any
        cap? Plus the adjacent question C-01: can a relying party lawfully
        rely on an Amex attestation in each target jurisdiction?
        These gate the entire external revenue model and require zero
        engineering to obtain. Start them on day one.

    D2  Real internal cost base                                day 60
        Replace every [ASSUMPTION] in the business case with actual
        per-verification cost, actual refresh volume, actual commercial KYB
        spend. Plus C-02: does continuous monitoring satisfy periodic
        refresh obligations? Bucket A2 depends on it.

    D3  Record-linkage reuse assessment                        day 45
        Can the existing Fellegi-Sunter + calibrated LightGBM inference
        service serve identity resolution? What is the feature-set delta
        (06-ml D1–D4)? This is the largest single variance in the whole
        plan — 9-month platform versus 18-month platform.

    Supporting engineering work in the same 90 days
      - credential schema v0 and forbidden-key validator, with CI
      - threat model review with Information Security
      - build-vs-buy assessment for M4 (PAD and injection detection)
      - spike: SD-JWT VC issuance and verification against a real device
        secure enclave, end to end, one claim. Proves the hard parts of the
        crypto path work on real hardware before anyone commits to a date.

    GATE to Phase 1
      Addressable internal cost base confirmed at or above $150M/year
      AND Legal has not ruled out warranty in principle
      AND the enclave spike works

      If the cost base comes in materially lower, the internal-savings case
      weakens and the programme should be re-argued on revenue defence
      alone. That is a defensible pitch but a different one, and it should
      be made honestly rather than by quietly adjusting assumptions.

---

## 9.3 Phase 1 — Internal reuse (6–9 months, 2 squads)

Amex is the only relying party. No external dependency of any kind. This is
what makes Phase 1 fundable independently of anyone else's adoption.

    Epics
      E1.1  atp-issuer, atp-gateway, HSM adapter, issuance registry
      E1.2  Wallet in the Amex app: keygen, device attestation, storage,
            per-claim consent UI
      E1.3  atp-revocation: status lists, publish pipeline, CDN, 60s
            propagation
      E1.4  atp-verifier and the JVM SDK
      E1.5  Entity resolution delta (06-ml D1–D4)
      E1.6  Document intelligence and forgery detection for new applicants
      E1.7  PAD and injection detection integration
      E1.8  atp-audit and event streams
      E1.9  Internal RP integration: Amex product onboarding
      E1.10 Internal RP integration: periodic refresh
      E1.11 DPIA, Art. 22 appeal path, accessibility review

    Sequencing note
      E1.1 through E1.4 form the critical path and should be one squad.
      E1.5 through E1.7 are the ML squad and can run fully parallel — they
      do not block issuance to existing cardmembers, who skip document
      capture entirely. This parallelism is why an existing-member-first
      launch is both the better product decision and the faster one.

    Pilot design
      Cohort: existing cardmembers, single market, opt-in. Start small
      enough that a full revocation and re-issue is operationally trivial,
      because the first pilot is the most likely to need one.

    Exit criteria
      M1 > 60% of periodic refreshes satisfied by credential
      M2 > 70% reduction in multi-product onboarding time
      M4 measured cost per satisfied verification < $0.30
      G1–G4 guardrails green
      Key compromise playbook (08-ops 8.5) rehearsed at production scale

---

## 9.4 Phase 2 — eIDAS and Business Passport (9–15 months, 3 squads)

    Epics
      E2.1  ARF conformance: OpenID4VP acceptance, mDL interop
      E2.2  EU regional deployment, residency controls, regional keys
      E2.3  Relying-party registration in target member states
      E2.4  Business Passport schema, issuance, officer key binding
      E2.5  Beneficial ownership assertion pipeline
      E2.6  Commercial and merchant onboarding integration
      E2.7  Continuous risk scoring (M5) and event-driven revalidation
      E2.8  QTSP assessment (C-06)

    Exit criteria
      M5 > 50% KYB cycle time reduction
      M6 ARF conformance certified in target markets
      Late-2027 eIDAS acceptance obligation demonstrably satisfied by this
        platform rather than by a parallel compliance programme

    The eIDAS deadline is the hard external date in the plan. Everything
    else can slip and be re-argued. That one cannot.

---

## 9.5 Phase 3 — Agent Passport and external relying parties

Protocol work should start during Phase 1, not after Phase 2. Visa shipped
TAP in October 2025; Mastercard shipped Verifiable Intent in 2026. If Phase
3 lands after mid-2027, Amex integrates into an agent-commerce ecosystem
whose conventions two competitors already set.

    Epics
      E3.1  atp-mandate: issuance, scope evaluation, cumulative tracking
      E3.2  Authorisation path integration, 40ms budget, fail-open
      E3.3  Agent protocol adapters (ADR-008) — priority per OQ-04
      E3.4  Liability and chargeback treatment for attested agent traffic
      E3.5  RP SDK: JS and Python; pricing and metering; warranty tiers
      E3.6  RP onboarding, entitlements, webhooks
      E3.7  Consent UX for agent mandates — the hardest UX problem in the
            programme. A member must understand a delegation of spending
            authority to software well enough that their consent is
            meaningful. Get this wrong and the liability assertion in
            consent.human_present does not hold up.

    Exit criteria
      M7  external RPs in production
      M8  agent transactions carrying valid mandates on Amex rails
      M9  measurable authorisation-rate delta, attested vs unattested
      M10 attestation revenue per verification

---

## 9.6 Team shape

    Phase 0     1 squad: 1 tech lead, 2 backend, 1 ML, plus part-time
                Legal, Compliance, Finance, Privacy
    Phase 1     Squad A platform: 4 backend, 1 mobile, 1 SRE
                Squad B ML: 2 ML engineers, 1 data engineer
                Shared: 1 product, 1 security engineer, 1 designer
    Phase 2     +1 squad for Business Passport and EU deployment
    Phase 3     +1 squad for agent and RP integration

    Named non-engineering dependencies, without which phases stall:
      Legal            OQ-01, C-01, C-03, C-04 — critical path in Phase 0
      Compliance       C-02, periodic refresh position
      Privacy          DPIA, disclosure of the pairwise re-linkage
                       asymmetry
      Fraud Risk       every model threshold, AR-01 through AR-05 sign-off
      Model validation independent validation before any production
                       decisioning use

    The Fraud Risk dependency is chronically underestimated in plans like
    this. Five models, each with an operating point that Risk owns and must
    validate. Book that capacity in Phase 0, not when the models are ready.

---

## 9.7 Test strategy

    Unit                  standard coverage expectations

    Contract              every RP-facing endpoint has a published contract
                          test suite. RPs run it in their CI. This is how
                          integrations stay working across our releases.

    Cryptographic         known-answer tests for SD-JWT issuance,
                          disclosure and key binding, against the IETF test
                          vectors. Plus a specific test asserting per-claim
                          salt entropy ≥ 128 bits and salt uniqueness — the
                          most likely serious implementation error in the
                          build (05-security 5.3).

    Schema / privacy      CI test asserting no forbidden key can appear in
                          any credential or event payload. Run against the
                          forbidden-key list in 04-data 4.1.1. This test
                          failing is a build break, never a warning.

    Device                real-hardware matrix: iOS Secure Enclave and
                          Android StrongBox across a device tier spread.
                          Emulator testing does not prove enclave
                          behaviour and must not be accepted as evidence.

    Load                  authorisation path at 5x peak with p99 asserted
                          at 40ms. Verification path at 5x peak at 150ms.

    Chaos                 ATP down, verified fail-open. HSM unreachable,
                          verified issuance-fails-closed and
                          verification-continues. Status list stale,
                          verified alerting fires. Region loss, verified
                          failover.

    Adversarial           quarterly red team against M3 and M4 with
                          current-generation tooling. Results feed model
                          refresh.

    Fairness              M2, M4, M5 evaluated by demographic and
                          jurisdiction before each release. Material
                          disparity blocks the release.

---

## 9.8 Top delivery risks

    R1  Record-linkage reuse fails
        Impact: +6 months, timing argument weakens
        Mitigation: assess by Phase 0 day 45, before commitments are made
        Owner: engineering

    R2  Legal will not underwrite any assertion
        Impact: external revenue model collapses to a commodity
        signature-check business
        Mitigation: answer by Phase 0 day 30; Phases 1–2 remain viable on
        internal savings alone
        Owner: Legal

    R3  No relying-party distribution
        Impact: Bucket B never materialises
        Mitigation: never let the business case depend on Phase 3. Phases
        1–2 pay for themselves with Amex as the sole relying party.
        Owner: GMNS strategy

    R4  Scope creep into a ledger programme
        Impact: governance overhead outlives the funding
        Mitigation: ADR-002 is decided; Phase 4 is separately gated. Say
        this out loud in every steering meeting, because it will be
        proposed repeatedly.
        Owner: programme lead

    R5  Fraud Risk capacity becomes the critical path
        Impact: models built but not approved for production decisioning
        Mitigation: book capacity in Phase 0
        Owner: programme lead

    R6  Agent standards converge somewhere unexpected
        Impact: adapter rework
        Mitigation: ADR-008 keeps protocol at the transport layer, so
        rework is adapters, not architecture
        Owner: engineering

    R7  Enrolment integrity failure at scale
        Impact: severe. Synthetic identities holding warranted credentials
        is the failure mode that ends the programme rather than delaying
        it.
        Mitigation: seven-layer enrolment controls (05-security 5.4),
        standing adversarial red team, guardrail G1 with launch-blocking
        authority
        Owner: Fraud Risk

R7 is the one to worry about. Every other risk on this list costs time or
money. R7 costs the thing the whole product is selling.
