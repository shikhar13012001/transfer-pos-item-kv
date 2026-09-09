# 06 — ML Specification

Status: Draft for Phase 0 review
Governance: every model here sits in a regulated decisioning path and is in
scope for model risk management from day one, not at productionisation.

---

## 6.1 Model inventory

    M1  Entity resolution          REUSE existing service     Phase 0/1
    M2  Document intelligence      build / fine-tune          Phase 1
    M3  Forgery and tamper         build                      Phase 1
    M4  Presentation attack and    build / vendor + build     Phase 1
        injection detection
    M5  Continuous risk scoring    build                      Phase 2

M1 and M5 are the two that only Amex can build well — M1 because the
customer graph exists, M5 because the closed loop sees both sides of every
transaction. M2–M4 are competitive with vendor offerings and a build-vs-buy
assessment is a legitimate Phase 0 output.

---

## 6.2 M1 — Entity resolution

    Status              REUSE. Existing production service.
    Existing form       Fellegi-Sunter probabilistic record linkage with
                        calibrated LightGBM confidence scoring; leakage-safe
                        feature engineering; business-metric-first
                        evaluation. Deployed in shadow mode against live
                        traffic. Already exposed as an inference API that
                        other Amex applications call and threshold on.
    Role in ATP         At issuance: does this record map to exactly one
                        identity in the customer graph, with what calibrated
                        confidence?
    Output              resolution_confidence ∈ [0,1], calibrated

    Why this matters disproportionately
      Calibration is the expensive, slow part of building a linkage model.
      An uncalibrated score cannot be thresholded by a risk policy, and a
      risk policy that cannot set its own threshold cannot own the decision.
      This work is already done and already validated against a business
      metric. Reusing it is the single largest de-risking factor in the
      programme (ADR-007).

    Delta required for ATP
      D1  Extend features beyond name-pair similarity to include document,
          device and address signals
      D2  Derive issuance-appropriate labels. The existing labels come from
          payment allocation outcomes; issuance needs "same person" labels.
          This is the substantive piece of work.
      D3  Re-calibrate on the issuance population, which differs in
          distribution from the payment-adjudication population
      D4  Extend the existing inference API contract with the new
          feature set

    Sizing: weeks, not quarters — IF the reuse assumption holds. Validating
    it is Phase 0 OQ-03, due day 45, and it determines a 9-month vs
    18-month platform.

    Thresholds are risk-policy parameters, not model constants
      confidence ≥ T_high     issue at assurance "high"
      T_low ≤ c < T_high      issue at "substantial", flag for review
      confidence < T_low      no issuance, route to full verification
      multiple strong matches no issuance, escalate — this is a signal of
                              either a data-quality problem or a synthetic
                              identity, and both need a human

    T_high and T_low are owned by Fraud Risk, set from the calibration
    curve, and changeable without a model release. Hardcoding them in the
    model service is an anti-pattern the existing service already avoids.

---

## 6.3 M2 — Document intelligence

    Task            parse government ID documents across jurisdictions
    Approach        fine-tuned vision-language model
    Deployment      on-device where hardware permits; document images do
                    not transit the network on the common path (NFR-14)
    Inputs          document image, capture metadata
    Outputs         structured fields, MRZ, per-field confidence
    Scope           only exercised for net-new applicants; existing
                    cardmembers never hit this path

    Requirements
      - multi-script, multi-jurisdiction, robust to rotation, glare, blur,
        partial occlusion
      - per-field confidence, not just extraction — a low-confidence field
        must be recoverable by re-capture, not silently accepted
      - MRZ checksum validation as an independent cross-check on OCR output
      - graceful degradation to assisted manual capture, never a hard fail

    Evaluation
      field-level accuracy by jurisdiction and document type
      MRZ checksum agreement rate
      recapture rate (UX metric, watched as a guardrail — an accurate model
        that demands three recaptures is a worse product than a slightly
        less accurate one that works first time)
      per-jurisdiction fairness: accuracy must not vary materially by
        issuing country. Uneven accuracy across jurisdictions is a
        discrimination exposure, not just a quality issue.

---

## 6.4 M3 — Forgery and tamper detection

    Task            is this document physically and digitally authentic
    Approach        CNN over print-noise residuals plus engineered forensic
                    features
    Output          calibrated fraud probability, NOT a binary

    Feature families
      print noise residuals and sensor pattern
      font rendering consistency against known issuer templates
      splice and edge artifacts, resampling traces
      microprint integrity at capture resolution
      compression history and metadata provenance
      template geometry conformance for the claimed document type

    Evaluation
      ROC-AUC overall and per document type
      TPR at fixed FPR — the operating point is set by Fraud Risk, and the
        model is evaluated at that point, not at the point flattering to
        the model
      performance against a held-out set of forgeries generated by
        current-generation tools, refreshed quarterly

    The threshold is a risk-policy parameter. The model reports a
    probability; the policy layer decides. Same discipline as M1, and for
    the same reason.

---

## 6.5 M4 — Presentation attack and injection detection

    Task            is a live human present, and is the feed authentic
    Two problems, often conflated, and the second is now harder

    PAD (is it a live human)
      passive: screen moiré, corneal reflection, rPPG blood-flow signal,
               skin texture micro-variation, depth cues
      active:  randomised challenge-response, checking motion naturalness
               and temporal consistency

    Injection detection (is the feed real)
      virtual camera detection
      hardware attestation of the capture path
      frame-level provenance and timing analysis
      This is the growth area. A perfect liveness model is fully bypassed
      by a synthetic feed injected below the camera API, and attacker
      tooling here has improved faster than detection. Treat injection
      detection as the primary control and PAD as the secondary, which is
      the reverse of how most vendor stacks are marketed.

    Evaluation
      ISO/IEC 30107-3 PAD metrics: APCER, BPCER, ACER
      separate reporting by attack instrument species — an aggregate ACER
        hides catastrophic failure against one attack type
      demographic fairness across skin tone, age and gender. Published
        NIST FRVT work shows material demographic variation in this class
        of model. Unequal BPCER by demographic is a direct discrimination
        exposure and a launch blocker, not a backlog item.

    Vendor consideration: this is the strongest build-vs-buy candidate.
    Specialist vendors have adversarial datasets Amex cannot easily
    assemble. A hybrid — vendor PAD, Amex-built injection and orchestration
    — is the likely answer. Phase 0 should test it.

---

## 6.6 M5 — Continuous risk scoring

    Task            maintain a live risk signal per identity, so a
                    credential can be revoked when facts change rather than
                    decaying silently until expiry
    Approach        graph neural network over the closed-loop transaction
                    graph
    Output          risk band plus a revocation recommendation

    This model is the technical expression of the closed-loop advantage.
    Amex sees the cardmember side and the merchant side of the same
    transaction. An open-loop network sees a routed message. The graph that
    can be built here cannot be built by a competitor, which makes this the
    most defensible model in the inventory.

    Signals
      transaction velocity and structure
      geographic consistency against the credential's jurisdiction claim
      counterparty risk, including sanctioned and high-risk entities
      wallet and account clustering
      device and credential presentation patterns

    Outputs feed
      re-verification triggers
      status revocation (FR-13, 60s propagation)
      risk_band returned in mandate authorisation

    Constraints
      - a revocation recommendation with member impact requires either a
        human review step or an explicit, documented auto-revoke policy
        with an appeal path. Automated revocation of a member's identity
        credential with no recourse is both a regulatory and a reputational
        problem.
      - explainability to model risk management standard is mandatory. GNN
        explainability is genuinely hard; budget for it explicitly rather
        than discovering it at the model validation gate.

---

## 6.7 Model governance — applies to all five

    Registry        every model versioned, with a model card: purpose,
                    training data provenance, evaluation, known limitations,
                    approved operating point, owner
    Validation      independent validation before any production decisioning
                    use, per existing Amex MRM standards
    Thresholds      owned by Fraud Risk, configurable without a model
                    release, versioned and audited
    Monitoring      per-model drift on input distribution and output
                    distribution; alerting on breach
    Drift response  degrade assurance level and route to manual review.
                    Never silently continue at the old assurance level.
    Fairness        M2, M4 and M5 report performance by demographic and
                    jurisdiction. Material disparity is a launch blocker.
    Retraining      no production feedback loop retrains without human
                    review — an automated loop trained on its own decisions
                    will amplify its own errors, and in a fraud context an
                    attacker can steer it deliberately
    Adversarial     standing quarterly red team against M3 and M4 with a
                    recurring budget. Non-negotiable. Capability on the
                    attack side improves faster than an annual model
                    refresh cycle.
