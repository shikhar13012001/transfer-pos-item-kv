# 08 — Operations Runbook

Status: Draft, to be completed before Phase 1 pilot
On-call: platform team, follow existing Amex escalation

---

## 8.1 Service level objectives

    Service              SLI                    SLO        Error budget
    ─────────────────────────────────────────────────────────────────────
    atp-mandate          availability           99.99%     4.3 min/month
                         p99 latency            < 40ms
    atp-verifier         availability           99.99%     4.3 min/month
                         p99 latency            < 150ms
    status list (CDN)    availability           99.995%
                         p99 latency            < 20ms
    atp-issuer           availability           99.9%      43 min/month
                         p95 end-to-end         < 30s
    atp-audit            write durability       no loss    zero tolerance

Error budget policy: budget exhaustion freezes feature deploys to that
service until the budget recovers. Applies to atp-mandate and atp-verifier
without exception — they sit next to the payment path.

---

## 8.2 Golden signals and alerting

    Alert                              Threshold          Severity
    ─────────────────────────────────────────────────────────────────────
    mandate authorize p99 > 40ms       5 min sustained    Sev-2
    mandate authorize p99 > 100ms      2 min sustained    Sev-1
    verify error rate > 1%             5 min              Sev-2
    status list stale > 120s           immediate          Sev-1
    issuer signing failure             any                Sev-1
    HSM unreachable                    any                Sev-1
    PII detected in credential payload any                Sev-1  see 8.6
    PII detected in event stream       any                Sev-1  see 8.6
    entity resolution confidence       distribution       Sev-2
      distribution shift               drift threshold
    PAD false-accept spike             > 2x baseline      Sev-1
    verification volume anomaly, one   > 5x RP baseline   Sev-2
      RP
    fail-open rate > 0.1% of traffic   15 min             Sev-2

The last one deserves attention on-call: elevated fail-open is either an
ATP problem or an attack attempting attestation stripping (05-security
T15). Both need investigation; only one is an outage.

---

## 8.3 Runbook — status list stale

Symptom: verifiers serving revocation decisions from a list older than 120s.
Impact: revoked credentials may verify as active. Direct fraud exposure.

    1  Confirm scope: which list type and period, which regions.
    2  Check atp-revocation publish pipeline health and last publish time.
    3  Check CDN purge and origin fetch.
    4  If publish pipeline is down: publish manually from origin. The
       procedure is scripted; run it, do not hand-craft a list.
    5  If CDN is stale but origin is current: purge affected paths.
    6  If neither recovers within 15 minutes: notify Fraud Risk. Elevated
       manual review thresholds apply until resolved — this is the
       documented degradation, not an ad hoc decision.
    7  After recovery: identify every verification served during the stale
       window against a credential revoked during that window. Notify
       affected relying parties via the webhook channel.

Step 7 is not optional and is the part most likely to be skipped under
pressure. Automate it.

---

## 8.4 Runbook — elevated fail-open

    1  Determine cause: ATP unavailability, network partition, or RP-side
       timeout misconfiguration.
    2  If ATP-side: standard availability incident.
    3  If traffic-shaped — concentrated on specific RPs, specific
       geographies, or coinciding with a volume spike — treat as a possible
       attestation-stripping attack. Escalate to Fraud Risk immediately.
    4  Fraud Risk decides whether to raise manual review thresholds.
    5  Never resolve by making ATP a hard dependency. ADR-005 stands during
       incidents; the pressure to "just fail closed until this is fixed" is
       predictable and must be refused.

---

## 8.5 Runbook — issuer key compromise

Sev-1. Rehearse this before Phase 1 pilot. A playbook first executed during
a real incident is not a playbook.

    1  Declare Sev-1. Engage Information Security and Legal immediately.
    2  Halt issuance on the affected key: disable the key ID in the HSM
       adapter. Issuance fails closed; verification continues.
    3  Identify blast radius:
         SELECT credential_id FROM credential
         WHERE issuer_key_id = :compromised AND revoked_at IS NULL;
       Scope is bounded by credential type and region because keys are
       separated on both axes (05-security 5.5).
    4  Publish the compromise to the key transparency log.
    5  Bulk-revoke affected credentials, batched to respect status list
       publish capacity.
    6  Notify relying parties: bulk revocation event over webhooks; direct
       contact for significant RPs. Do not let RPs discover this from
       status lists alone.
    7  Rotate to a new key. Begin re-issuance to affected holders.
    8  Post-incident: was the pairwise secret in the same blast radius? If
       yes, this escalates to full re-issuance of every credential in the
       region, because subject_ref correlation is compromised.

Rehearsal requirement: steps 3–5 executed against a non-production key at
production scale, at least once before Phase 1 pilot, and annually after.

---

## 8.6 Runbook — PII leak into a credential or event

Sev-1 regardless of volume. One occurrence is a control failure, and the
control (schema validation, 04-data 4.1.1) is supposed to make it
impossible.

    1  Declare Sev-1. Engage Privacy and Legal within the hour — regulatory
       notification clocks may already be running.
    2  Stop the bleeding: halt issuance or the emitting producer.
    3  Determine scope: which field, how many records, which downstream
       consumers received it.
    4  For events: events are retained and may have been consumed
       downstream. Trace every consumer. This is why 04-data rule 1 exists.
    5  Revoke and re-issue affected credentials. A credential containing
       PII cannot be corrected in place — the holder has it, and so does
       every RP it was shown to.
    6  Root cause: how did schema validation not catch this? The validator
       is the control. If it was bypassed, disabled, or the forbidden-key
       list was incomplete, fix the control before resuming.
    7  Regulatory notification per Legal's assessment.

---

## 8.7 Routine operations

    Key rotation           annual per credential type per region; overlap
                           period until the last credential signed by the
                           old key expires
    Status list rollover   monthly period boundary; new list published
                           before the boundary, old list served until every
                           credential in it expires
    Model refresh          per 06-ml governance; drift-triggered or
                           scheduled; assurance degrades rather than
                           silently continuing during a gap
    Adversarial red team   quarterly against M3 and M4
    DR test                semi-annual regional failover, including a
                           verification-only failover with issuer down
    Capacity review        quarterly against the 5x peak design target

---

## 8.8 What good looks like on a normal day

    - fail-open rate below 0.01% of verification traffic
    - zero PII alerts
    - status list freshness under 60s at p99
    - mandate authorize p99 comfortably inside 40ms with headroom
    - entity resolution confidence distribution stable week over week
    - no manual status list publishes

Deviation in any of these is worth investigating before it becomes an
alert. The entity-resolution distribution in particular is a leading
indicator: it shifts before fraud metrics do.
