# 07 — Compliance Mapping

Status: Draft for Phase 0 review
Review required from: Legal, Compliance, Privacy, Regulatory Affairs

This document maps regulatory obligations to the specific design control
that satisfies them. It is written so a reviewer can trace an obligation to
a mechanism, not to an intention.

---

## 7.1 eIDAS 2.0 — Regulation (EU) 2024/1183

Timeline as understood at time of writing:
  20 May 2024      regulation in force
  24 Dec 2026      every member state must issue at least one certified
                   EUDI Wallet; public-sector relying parties must accept
  Late 2027        mandatory acceptance by regulated private-sector
                   entities, banking explicitly in scope

The strategic point: Amex must build EUDI Wallet acceptance regardless. The
only decision is whether it is a standalone compliance cost or the same
platform that generates revenue. ATP makes it the latter — one build, two
outcomes.

    Obligation                        ATP control
    ─────────────────────────────────────────────────────────────────────
    Accept EUDI Wallet credentials    atp-verifier implements OpenID4VP;
    for strong authentication         ARF-conformant presentation handling
                                      (Phase 2 scope)

    Interoperable credential format   W3C VC 2.0 + SD-JWT VC, ISO/IEC
                                      18013-5 mDL interop (ADR-001,
                                      02-arch section 7.1 constraint 3)

    Relying party registration        Amex registers as a relying party in
                                      each target member state; entitlement
                                      declarations must match the
                                      registered purpose

    Selective disclosure and user     per-claim consent in wallet, per-RP
    control                           entitlements at gateway (FR-03, FR-06)

    Data minimisation by relying      entitled_claims enforcement, 403
    parties                           before the wallet sees the request
                                      (04-data section 4.2.3)

    Open: whether Amex should also pursue Qualified Trust Service Provider
    status to issue ARF-conformant attestations of attributes into EU
    wallets, rather than only consuming them. This is the difference
    between compliance and market position in the EU.
    Owner: Regulatory Affairs. Due: Phase 2 planning.

---

## 7.2 GDPR

    Article / principle          ATP control
    ─────────────────────────────────────────────────────────────────────
    Art. 5(1)(c) minimisation    predicates not values. No field exists in
                                 any schema for a birthdate, address or
                                 document number, so minimisation is a
                                 structural property rather than a policy
                                 the implementation might violate.

    Art. 5(1)(b) purpose         per-RP entitlements bound to a declared
    limitation                   purpose; requests outside entitlement are
                                 rejected at the gateway

    Art. 6 lawful basis          issuance on member consent; verification
                                 on the RP's own lawful basis, which the RP
                                 declares at onboarding. Amex is controller
                                 for issuance, and the controller/processor
                                 split for verification must be settled by
                                 Legal.  [OPEN — owner: Legal, due Phase 1]

    Art. 7 consent               per-claim, granular, in the wallet,
                                 showing the requesting party by name;
                                 withdrawable by revoking the credential

    Art. 17 erasure              no PII exists in ATP credentials, events
                                 or registry, so there is nothing in ATP to
                                 erase. Erasure obligations continue to
                                 apply, unchanged, against the existing
                                 systems of record. This is the single
                                 most important reason ADR-002 excludes a
                                 ledger: an immutable store and an erasure
                                 right cannot coexist, and the standard
                                 workaround — storing hashes — fails
                                 because a hash of an enumerable input is
                                 still personal data under EDPB guidance.

    Art. 22 automated decisions  issuance decline is an automated decision
                                 with legal or similarly significant
                                 effect. Requires: human review path,
                                 explanation, and contestability. Design
                                 the appeal path in Phase 1, not later.
                                 [OPEN — owner: Legal + Product]

    Art. 25 by design/default    default disclosure is nothing; every claim
                                 is opt-in per presentation

    Art. 32 security             05-security in full

    Art. 35 DPIA                 required. This programme is high-risk
                                 processing on any reading — biometric
                                 processing, large scale, novel technology.
                                 DPIA must complete before Phase 1 pilot,
                                 not before general availability.
                                 [OPEN — owner: Privacy, due Phase 1 start]

    Chapter V transfers          EU credentials issued and verified in EU
                                 region; no cross-border PII movement
                                 (02-arch section 2.8)

---

## 7.3 Data residency

    EU              issuance and verification in EU region. Systems of
                    record access stays in region. Pairwise secret and
                    signing keys are region-specific per 05-security 5.5.
    UK              treated as a separate region post-UK GDPR divergence
                    [OPEN — confirm with Regulatory Affairs]
    India           DPDP Act obligations; local processing requirements to
                    be confirmed  [OPEN]
    US              no residency constraint; state privacy law
                    (CCPA/CPRA and successors) applies to the same
                    minimisation controls

---

## 7.4 BSA / AML and sanctions

    Obligation                    ATP position
    ─────────────────────────────────────────────────────────────────────
    CIP / CDD                     ATP does not replace CIP. It records and
                                  makes portable that CIP was performed to
                                  Amex standard. A relying party's reliance
                                  on an Amex attestation to satisfy its own
                                  CIP obligation depends on the reliance
                                  provisions in its jurisdiction and is the
                                  central legal question of the external
                                  business.
                                  [BLOCKING OPEN — owner: Legal. This is
                                  the same question as PRD OQ-01 viewed
                                  from the regulatory side. If reliance is
                                  not permissible, the external value
                                  proposition changes from "satisfies your
                                  KYC" to "materially accelerates your
                                  KYC" — still valuable, but priced
                                  differently and sold differently.]

    Periodic refresh              continuous risk scoring (M5) supports
                                  event-driven revalidation. Whether this
                                  satisfies periodic review obligations, or
                                  only supplements them, must be confirmed
                                  per jurisdiction. Bucket A2 of the
                                  business case depends on the answer.
                                  [BLOCKING OPEN — owner: Compliance]

    Sanctions screening           sanctions_screened_at asserts recency,
                                  not clearance in perpetuity. RPs must
                                  understand that a 30-day-old screening
                                  is a 30-day-old screening. The
                                  integration guide must state this
                                  explicitly; an RP that treats it as
                                  ongoing clearance has misread the claim.

    Recordkeeping                 atp-audit append-only log; retention per
                                  existing policy

    Suspicious activity           M5 outputs feed existing AML platform.
                                  ATP does not file; it signals.

---

## 7.5 GENIUS Act adjacency

Enacted 18 July 2025; effective December 2026 or 120 days after final
rules, whichever is earlier. OCC and Treasury NPRMs issued during 2026.

ATP does not issue, hold or settle any digital asset. It is explicitly out
of scope (PRD 1.8).

The adjacency is narrow and worth stating precisely so nobody overclaims it
in a leadership deck: permitted payment stablecoin issuers carry full BSA
and sanctions obligations, including transaction monitoring across on-chain
and off-chain flows. Counterparty identity is the hard part of that
obligation. A Business Passport is a credible answer to "who is the
counterparty" in a regulated stablecoin settlement flow.

That is an option, not a plan. It requires no ATP design change to preserve,
and should be preserved by not designing it out — nothing more.

---

## 7.6 Model risk management

Every model in 06-ml sits in a regulated decisioning path.

    Requirement            Control
    ────────────────────────────────────────────────────────────────
    Independent validation before production decisioning use, per
                           existing Amex MRM standards
    Documentation          model card per model, per version
    Explainability         decline reasons explainable to member and
                           regulator; GNN explainability (M5) is
                           genuinely hard and separately budgeted
    Ongoing monitoring     drift monitoring with defined response —
                           degrade assurance, never silently continue
    Fairness testing       M2, M4, M5 report by demographic and
                           jurisdiction; material disparity blocks launch
    Change control         threshold changes are risk-policy changes,
                           versioned and audited, not code deploys

---

## 7.7 Accessibility and consumer protection

Easily forgotten in an identity programme and expensive to retrofit.

    - The wallet consent UI is a consumer disclosure surface. It must meet
      accessibility standards and be comprehensible, not merely accurate.
      "Acme Bank is asking whether you are over 18" is a disclosure.
      "Requesting: age_over_18 predicate" is not.
    - There must be a non-digital path. A member without a compatible
      device must not be excluded from any Amex product. ATP is an
      accelerator, never a gate.
    - Device loss has no key recovery (05-security 5.5). Product copy and
      support scripts must say so plainly rather than promising recovery
      that does not exist.
    - Automated decline requires an appeal path (Art. 22, and good
      practice regardless of jurisdiction).

---

## 7.8 Open questions register

    ID      Question                                Owner        Due
    ─────────────────────────────────────────────────────────────────────
    OQ-01   Will Legal underwrite an assertion at   Legal+Risk   P0 d30
            any cap?  BLOCKS the revenue model.
    OQ-02   Actual internal KYC cost base           Fin+Comp     P0 d60
    OQ-03   Record-linkage reuse feasibility        Eng          P0 d45
            DRIVES 9 vs 18 month estimate.
    OQ-04   Agent protocol adapter priority         GMNS         pre-P3
    C-01    Third-party reliance permissible per    Legal        P0 d60
            jurisdiction?  BLOCKS external sale.
    C-02    Does continuous monitoring satisfy      Compliance   P0 d60
            periodic refresh?  DRIVES Bucket A2.
    C-03    Controller/processor split on           Legal        P1
            verification
    C-04    Art. 22 appeal path design              Legal+Prod   P1
    C-05    DPIA                                    Privacy      P1 start
    C-06    QTSP status in EU — pursue or not       Reg Affairs  P2
    C-07    UK / India residency requirements       Reg Affairs  P2

The two that genuinely gate the business, as opposed to the build, are
OQ-01 and C-01. They are the same question from two directions: can Amex
stand behind an assertion, and can a relying party lawfully rely on it.
Both are legal answers, neither requires any engineering to obtain, and
both should be pursued from day one of Phase 0 in parallel with everything
technical. If both answers are no, Phases 1 and 2 still pay for themselves
on internal savings — but the programme should be re-scoped honestly rather
than pitched on a revenue line that cannot exist.
