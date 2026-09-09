# ZeroPass — Hackathon Build Specification

Working name: ZeroPass (short, says it, no "chain" in it)
Target: 36-hour hackathon, team of 3–4
Deliverable: one real end-to-end path, four mocked edges, a demo that
cannot be faked

---

## 0. The one thing that decides whether you win

Every identity hackathon project shows a slide saying "privacy-preserving."
Almost none show the judge the actual bytes the relying party receives.

Your demo's centre of gravity is a JSON blob on screen with a terminal next
to it:

    RELYING PARTY RECEIVED:
    {
      "verified": true,
      "age_over_18": true,
      "jurisdiction": "IN",
      "subject_ref": "u_7Kx2mQ9vLp4n..."
    }

    $ curl -s localhost:3000/rp/last-response | grep -iE "name|dob|birth|address|passport"
    $
    (no matches)

That grep returning nothing is the whole pitch, executed live, unfakeable.
Build toward that moment and cut anything that does not serve it.

Second unfakeable moment: revoke the credential from an admin panel, hit
verify again on the same credential, watch it flip to `revoked` in under a
second. That proves the system is live rather than a recorded video.

Third, if you have time: an agent tries to spend 600 against a 500 cap and
gets declined with `scope_exceeded`. This is the moment that says "we are
solving the 2026 problem, not the 2019 one."

---

## 1. Scope: what is real, what is mocked

Be ruthless here. A hackathon project that mocks the wrong thing is a
slide deck with a database.

### REAL — do not fake any of these, they are the substance

    R1  SD-JWT credential issuance with per-claim salted digests
    R2  Selective disclosure — holder chooses which claims to reveal
    R3  Holder key binding — non-extractable WebCrypto key, KB-JWT signed
        over a fresh nonce
    R4  Signature verification, key binding verification, nonce freshness
    R5  Status list revocation, live, sub-second
    R6  Agent mandate scope enforcement, server-side
    R7  One genuine AI call: document parse + forgery signal from a real
        image the judge hands you

If any of R1–R4 is faked you do not have a project, you have a demo of a
JSON API. The crypto is the thing that makes the privacy claim true rather
than asserted.

### MOCKED — fake these, and say so out loud

    M1  HSM             → local ES256 keypair from env var
    M2  Plaid / bank    → fixture JSON, 400ms artificial delay so it feels
        identity API      like a network call
    M3  Sanctions/PEP   → static JSON list, exact-match lookup
    M4  Amex systems    → seeded SQLite table of 20 fake cardmembers
        of record
    M5  Liveness / PAD  → stub returning pass, with an honest slide.
                          Real PAD is not a 36-hour build and pretending
                          otherwise is the fastest way to lose credibility
                          in Q&A.
    M6  Payment auth    → fake merchant checkout page
    M7  Entity          → rapidfuzz over the seeded table, returning a
        resolution        real similarity score. Cheap, and honest — it IS
                          fuzzy matching, just not a calibrated model.

Rule: mocks return realistic latency and realistic failure. A mock that
returns instantly and always succeeds makes the demo feel fake even when
the real parts are real.

### Put this in the README and on a slide

    REAL:   SD-JWT issuance, selective disclosure, holder key binding,
            signature + status verification, mandate scope enforcement,
            AI document analysis
    MOCKED: HSM (env keypair), bank identity API, sanctions list,
            cardmember database, liveness check, payment authorisation

Judges will ask what is mocked. Answering before they ask converts a
weakness into a signal that you know what you built. Getting caught
overclaiming is fatal and completely avoidable.

### DO NOT BUILD — the graveyard list

    ✗  Any blockchain, ledger, smart contract or on-chain anything.
       It adds a day of RPC pain, a wallet-connect flow, testnet faucet
       hunting, and zero demo value. Nothing about the privacy story
       needs it. If a judge asks, say: "PII on an immutable ledger is a
       GDPR erasure violation with a delay fuse — we anchor nothing, and
       that is a design decision, not a shortcut." That answer scores
       higher than a testnet transaction hash.
    ✗  React Native. Six hours of Xcode and Gradle before you render a
       button. Build a mobile-sized PWA instead — same demo, same feel,
       runs on the judge's phone from a QR code.
    ✗  Real ZK-SNARK circuits. Circom plus a trusted setup plus proof
       generation is a whole weekend by itself. SD-JWT selective
       disclosure delivers the identical demo. Say "predicate proofs over
       salted claim digests" and mean it.
    ✗  Microservices. One process, clean modules.
    ✗  Auth, user accounts, password reset, email. Seed a session.
    ✗  Kubernetes, Terraform, CI/CD.
    ✗  A fifth screen. Four apps is already ambitious.

---

## 2. Tech stack

Chosen for library maturity and hours-to-first-working-thing, not for
what you would pick at work.

    Language        TypeScript everywhere
                    One language across backend, wallet and RP apps means
                    shared types and zero context switching at 3am. This
                    is worth more in a hackathon than any other stack
                    consideration.

    Backend         Node 20 + Fastify
                    Fastify over Express for built-in schema validation —
                    you get request validation free, which prevents a
                    class of 2am debugging.

    Crypto          jose               ES256 sign/verify, JWKS
                    @sd-jwt/core       SD-JWT issuance + disclosure
                    @sd-jwt/sd-jwt-vc  the VC profile
                    These are the load-bearing dependencies. Install and
                    prove a round-trip in hour one, before anything else.

    Frontend        Next.js 14 App Router + Tailwind + shadcn/ui
                    shadcn gives credible UI without a designer. Judges
                    see polish; you spend zero hours on CSS.

    Holder keys     WebCrypto generateKey with extractable: false,
                    stored as a CryptoKey handle in IndexedDB
                    This is genuinely non-exportable in the browser. It
                    is a real security property, not a simulation, and it
                    is a strong answer when a judge asks "what stops
                    someone copying the credential?"

    Database        SQLite via better-sqlite3
                    Zero setup, synchronous API, one file you can delete
                    to reset the demo. Postgres buys you nothing here and
                    costs you Docker.

    AI              Anthropic API, Claude with vision
                    Document parse + forgery signals in one structured
                    call. Force JSON output.

    Deploy          Vercel (frontends) + Railway or Render (API)
                    FALLBACK: everything local + ngrok. Have this working
                    before you need it. Conference wifi has ended more
                    demos than bad code.

    Extras          zod           runtime validation, shared schemas
                    rapidfuzz-ts  mock entity resolution
                    pino          logs you can actually read on stage

---

## 3. Component design

One backend process. Four frontends. Modules, not services.

    ┌─────────────────────────────────────────────────────────────┐
    │ FRONTENDS (Next.js)                                          │
    │                                                              │
    │  wallet/          mobile-sized PWA. Holds credentials.       │
    │                   Enrolment, consent, disclosure UI.         │
    │                                                              │
    │  bank/            "Meridian Bank" — the relying party.       │
    │                   Onboarding flow that asks for predicates.  │
    │                                                              │
    │  shop/            merchant + AI agent checkout demo.         │
    │                                                              │
    │  console/         admin. Revoke, inspect, reset. THE DEMO    │
    │                   CONTROL PANEL — build this, it is how you  │
    │                   drive the live demo without a terminal.    │
    └───────────────────────────┬─────────────────────────────────┘
                                │ REST + JSON
    ┌───────────────────────────▼─────────────────────────────────┐
    │ BACKEND (single Fastify process, port 3000)                  │
    │                                                              │
    │  modules/issuer/      mint SD-JWT VC, allocate status index  │
    │  modules/verifier/    verify VP: sig, key binding, nonce,    │
    │                       audience, status                       │
    │  modules/status/      bitstring status list, sign, serve     │
    │  modules/mandate/     agent mandate issue + scope evaluation │
    │  modules/ai/          document parse, forgery signal         │
    │  modules/crypto/      keys, salts, pairwise derivation       │
    │                                                              │
    │  mocks/plaid.ts       fixture identity, 400ms delay          │
    │  mocks/sanctions.ts   static list                            │
    │  mocks/amex-core.ts   seeded cardmember table                │
    │  mocks/resolution.ts  rapidfuzz score                        │
    │  mocks/liveness.ts    stub pass                              │
    └───────────────────────────┬─────────────────────────────────┘
                                ▼
    ┌─────────────────────────────────────────────────────────────┐
    │ SQLite: credentials · mandates · decisions · rp_registry     │
    │         seed_cardmembers · audit_log                         │
    └─────────────────────────────────────────────────────────────┘

Module boundaries matter even in one process: they let three people work
without merge conflicts, and they make the architecture slide honest.

### Repo layout

    zeropass/
      apps/
        wallet/          Next.js
        bank/            Next.js
        shop/            Next.js
        console/         Next.js
      services/
        api/
          src/
            modules/     issuer verifier status mandate ai crypto
            mocks/
            db/          schema.sql, seed.ts
            server.ts
      packages/
        shared/          zod schemas + TS types used by everything
      scripts/
        seed.ts
        demo-reset.ts    ← wire this to a keyboard shortcut
        e2e.ts           ← headless proof the whole path works
      fixtures/
        ids/             sample ID images: 3 genuine, 3 tampered
        plaid/
      README.md

---

## 4. Data flow diagrams

### 4.1 Enrolment — new user with a document

    Wallet          API/issuer      ai        mocks         DB
      │                 │            │          │            │
      │ capture ID photo│            │          │            │
      ├────────────────►│            │          │            │
      │                 │ analyze    │          │            │
      │                 ├───────────►│          │            │
      │                 │            │ Claude vision:        │
      │                 │            │ parse fields +        │
      │                 │            │ forgery signals       │
      │                 │◄───────────┤          │            │
      │                 │  {fields, authenticity: 0.94,      │
      │                 │   anomalies: []}      │            │
      │                 │            │          │            │
      │                 │ liveness (STUB)       │            │
      │                 ├──────────────────────►│            │
      │                 │◄──────────────────────┤ pass       │
      │                 │            │          │            │
      │                 │ bank identity (MOCK PLAID, 400ms)  │
      │                 ├──────────────────────►│            │
      │                 │◄──────────────────────┤ match      │
      │                 │            │          │            │
      │                 │ sanctions (MOCK)      │            │
      │                 ├──────────────────────►│            │
      │                 │◄──────────────────────┤ clear      │
      │                 │            │          │            │
      │                 │ entity resolution (rapidfuzz)      │
      │                 ├──────────────────────►│            │
      │                 │◄──────────────────────┤ 0.96       │
      │                 │            │          │            │
      │  generate ES256 keypair, extractable:false           │
      │  store handle in IndexedDB  │          │            │
      │                 │            │          │            │
      │ POST /issue {holder_pubkey, session}   │            │
      ├────────────────►│            │          │            │
      │                 │ derive claims → PREDICATES ONLY    │
      │                 │ salt each claim, hash, sign        │
      │                 │ allocate status idx   ├───────────►│
      │                 │            │          │            │
      │◄────────────────┤ SD-JWT VC  │          │            │
      │ store in IndexedDB           │          │            │
      │                 │            │          │            │

    KEY MOMENT for the demo: between "parse fields" and "sign", the DOB
    becomes age_over_18:true and is DISCARDED. Show this transition on
    screen. It is where the privacy property is actually created, and
    most audiences have never seen it visualised.

### 4.2 Presentation — bank onboarding

    Bank app        API/verifier    status      Wallet       User
       │                 │            │            │           │
       │ needs: age_over_18, jurisdiction          │           │
       │ POST /presentation/request   │            │           │
       ├────────────────►│            │            │           │
       │◄────────────────┤ {nonce, required_claims, audience}  │
       │                 │            │            │           │
       │ render QR / deep link ──────────────────► │           │
       │                 │            │            │           │
       │                 │            │  "Meridian Bank wants  │
       │                 │            │   to know: are you     │
       │                 │            │   over 18? which       │
       │                 │            │   country?"            │
       │                 │            │            ├──────────►│
       │                 │            │            │◄──────────┤ approve
       │                 │            │            │  per claim │
       │                 │            │            │           │
       │                 │  build VP: disclose ONLY approved   │
       │                 │  claims + KB-JWT signed over nonce  │
       │                 │            │            │           │
       │ POST /verify {presentation, nonce, audience}          │
       ├────────────────►│            │            │           │
       │                 │ check sig  │            │           │
       │                 │ check key binding vs cnf            │
       │                 │ check nonce fresh + unused          │
       │                 │ check audience == rp                │
       │                 │ status?    ├───────────►│           │
       │                 │◄───────────┤ active     │           │
       │◄────────────────┤ {verified, claims, subject_ref}     │
       │                 │            │            │           │
       │ ACCOUNT OPENED — 1.2 seconds, no documents received   │

### 4.3 Agent mandate

    User      Wallet      API/mandate    Shop/agent    DB
      │          │             │              │         │
      │ "let my agent book dinner, max ₹500"  │         │
      ├─────────►│             │              │         │
      │          │ POST /mandate/issue        │         │
      │          ├────────────►│              │         │
      │  device biometric confirm             │         │
      │◄─────────┤             │              │         │
      ├─────────►│ human_present = true       │         │
      │          ├────────────►│              ├────────►│
      │          │◄────────────┤ mandate VC   │         │
      │                        │              │         │
      │                        │  agent attempts ₹420   │
      │                        │◄─────────────┤         │
      │                        │ scope check  │         │
      │                        │ verify agent sig       │
      │                        ├───────────────────────►│
      │                        ├─────────────►│ AUTHORIZED
      │                        │              │ liability_shift: issuer
      │                        │              │         │
      │                        │  agent attempts ₹600   │
      │                        │◄─────────────┤         │
      │                        ├─────────────►│ DECLINED
      │                        │              │ scope_exceeded
      │                        │              │         │

### 4.4 Revocation — the credibility moment

    Console        API/status      CDN/cache      Bank app
       │               │               │              │
       │ revoke mp_01J9X7             │              │
       ├──────────────►│               │              │
       │               │ flip bit at idx 91274        │
       │               │ re-sign list  │              │
       │               ├──────────────►│              │
       │◄──────────────┤ done          │              │
       │                               │              │
       │  same credential presented again ───────────►│
       │                               │◄─────────────┤ verify
       │                               │              │
       │                       {outcome: "revoked"}   │
       │                                              │

    Under one second, on stage, same credential. This is what separates
    a working system from a recorded walkthrough.

---

## 5. Contracts a developer needs

Cut down from the enterprise spec. Enough to build against.

### POST /issue

    req  { session_id, holder_jwk, credential_type: "member-passport" }
    res  { credential_id, credential: "<SD-JWT>",
           status: { uri, idx }, resolution_confidence }

### POST /presentation/request

    req  { rp_id, required_claims: [...], optional_claims: [...] }
    res  { request_id, nonce, audience, expires_at }

### POST /verify

    req  { presentation, nonce, audience, required_claims }
    res  { outcome, claims, subject_ref, status_checked_at }

    outcome ∈ verified | invalid_signature | revoked
            | holder_binding_failed | expired | claim_not_present

    Five outcomes, not a boolean. Build it this way from hour one —
    retrofitting a boolean API into an outcome enum on hour 30 is a
    predictable and avoidable disaster, and the enum is a talking point
    ("claim_not_present is not an error, it is the privacy model
    working").

### POST /mandate/issue

    req  { credential_id, agent_id, agent_jwk, scope, consent }
    res  { mandate_id, mandate: "<VC>" }

### POST /mandate/authorize

    req  { mandate, agent_assertion, transaction: { amount, mcc, merchant } }
    res  { authorized, decision_ref, liability_shift,
           scope_remaining, decline_reason? }

### GET /status/mp/current

    res  signed JWT containing gzip+base64url bitstring
         Cache-Control: max-age=5    ← 5s not 60s, so live revocation
                                        lands inside the demo

### POST /admin/revoke  (console only, no auth, it is a hackathon)

    req  { credential_id, reason }

---

## 6. Credential shape

    {
      "iss": "https://zeropass.demo",
      "vct": "https://zeropass.demo/vc/member-passport/v1",
      "iat": 1757376000,
      "exp": 1788912000,
      "cnf": { "jwk": { "kty":"EC","crv":"P-256","x":"...","y":"..." } },
      "sub": "<pairwise>",
      "status": { "status_list": { "idx": 42, "uri": ".../status/mp/current" } },
      "_sd": ["<hash>","<hash>","<hash>","<hash>"],
      "_sd_alg": "sha-256"
    }

Allowed claims — keep it to six, it is a demo:

    verified_to_standard   bool
    age_over_18            bool
    age_over_21            bool
    jurisdiction           ISO-3166 alpha-2
    sanctions_clear        bool
    account_tenure_over_12 bool

Forbidden — put this list in a zod refinement that THROWS at mint time:

    name given_name family_name dob date_of_birth birth_date
    address street postal_code city document_number passport_number
    national_id aadhaar pan email phone photo image

Write a unit test asserting the mint function throws on each. It runs in
two seconds, it is the strongest single piece of evidence that the privacy
claim is structural, and you can put the passing test output on a slide.

### Two implementation details that will bite you

    1  Salt each claim with 16 random bytes, independently. A hash of the
       boolean `true` with a short or reused salt is recoverable by
       brute force in milliseconds — which silently defeats selective
       disclosure while appearing to work. Use crypto.randomBytes(16)
       per claim. @sd-jwt/core does this if you let it; do not
       hand-roll the digest.

    2  Pairwise subject_ref, or you have accidentally built a tracking
       key and a sharp judge will say so:
           subject_ref = base64url(
             hmacSHA256(PAIRWISE_SECRET, internal_id + ":" + rp_id)
           ).slice(0,32)
       Two lines of code. Have the console show the same person's
       subject_ref differing across two RPs — it is a ten-second demo
       beat that lands hard with anyone who knows the space.

---

## 7. The AI component

One call, structured output, real image. Do not build a pipeline.

    modules/ai/analyzeDocument.ts

    Input:  base64 image
    Model:  Claude with vision
    Prompt: extract fields; assess authenticity; return ONLY JSON

    Response schema (enforce with zod, retry once on parse failure):
    {
      "document_type": "passport" | "drivers_license" | "national_id",
      "issuing_country": "IN",
      "fields": { "date_of_birth": "1996-04-12", "expiry": "2031-04-11" },
      "field_confidence": { "date_of_birth": 0.97 },
      "authenticity_score": 0.94,
      "anomalies": [
        { "type": "font_inconsistency", "region": "MRZ",
          "severity": "high", "detail": "..." }
      ],
      "recommendation": "pass" | "review" | "reject"
    }

Fixtures: prepare 3 genuine-looking and 3 deliberately tampered sample IDs.
Tamper them visibly enough that the model catches it — mismatched font in
one field, a spliced photo edge, an inconsistent MRZ checksum.

    Demo beat: run the genuine one → pass. Run the tampered one → the
    anomalies array populates and the flow stops. Two images, ten
    seconds, proves the AI does something.

Cost control: cache by image hash. You will run this a hundred times in
testing and the same six fixtures every time.

Honesty line for the slide and for Q&A: "This is a general vision model
prompted for forensic signals. Production would need a purpose-trained
forgery model on real forgery data — we can describe that architecture but
we did not train one in 36 hours." Say it before you are asked.

---

## 8. Validation — how you prove it works

### 8.1 scripts/e2e.ts — run this constantly

Headless, no browser, prints pass/fail per assertion. Run it after every
significant merge. If it goes red you know within 30 seconds instead of
discovering it during the demo.

    ✓  issue credential for seeded user
    ✓  credential contains no forbidden claim key
    ✓  each _sd digest has a unique 16-byte salt
    ✓  present with age_over_18 only → verifier returns verified
    ✓  disclosed payload does NOT contain jurisdiction
    ✓  tamper one byte of the JWT → invalid_signature
    ✓  present with a different holder key → holder_binding_failed
    ✓  reuse a spent nonce → 409
    ✓  present to wrong audience → audience_mismatch
    ✓  revoke, re-present → revoked, within 5s
    ✓  mandate at cap → authorized
    ✓  mandate over cap → declined, scope_exceeded
    ✓  replay same txn_ref → declined, replay_detected
    ✓  same user, two RPs → subject_ref differs

Fourteen assertions. Every one is a claim you make on stage. If the script
is green, every claim in the pitch is true — and that is a genuinely
useful property to have at 4am when someone asks "does the revocation
thing still work after your refactor?"

### 8.2 The grep test

Put this in the README and run it live:

    $ curl -s localhost:3000/rp/last-response \
      | grep -iE "name|dob|birth|address|passport|aadhaar"
    $

Empty output. That is the product.

### 8.3 Negative demo path

Have a deliberate failure ready. Judges trust a system more when they see
it refuse something. Best options, in order:

    1  present a revoked credential      → revoked
    2  agent exceeds its cap             → scope_exceeded
    3  tampered document at enrolment    → anomalies, blocked

Show at least one. Two if the room is engaged.

---

## 9. Hour-by-hour, 36 hours, 4 people

Roles: **C** crypto/backend · **W** wallet · **R** RP+shop+console · **A** AI

    H0–2    ALL   repo scaffold, shared zod schemas, SQLite seed,
                  env keypair. C proves an SD-JWT issue→disclose→verify
                  round trip in a scratch script. NOTHING ELSE HAPPENS
                  UNTIL THAT ROUND TRIP WORKS. If @sd-jwt/core fights
                  you, find out in hour one, not hour twenty.

    H2–8    C     issuer + crypto module + status list
            W     wallet shell, WebCrypto keygen, IndexedDB
            R     bank app shell, presentation request flow
            A     Claude vision call, zod schema, fixture images

    H8–14   C     verifier: sig, key binding, nonce, audience, status
            W     consent UI, disclosure builder, KB-JWT
            R     bank onboarding UI, QR handoff
            A     tampered fixtures, anomaly rendering

    H14–18  ALL   ★ FIRST INTEGRATION ★
                  wallet → bank → verify, one claim, end to end.
                  This is the checkpoint. If integration has not
                  happened by H18 you are behind — cut the agent
                  feature immediately and protect the core path.

    H18–24  C     mandate module, scope evaluation, replay guard
            W     mandate consent screen
            R     shop + agent checkout, console with revoke button
            A     wire AI into the real enrolment flow

    H24–28  ALL   e2e.ts to 14 green assertions
                  demo-reset.ts on a keyboard shortcut
                  fix whatever e2e finds

    H28–32  ALL   polish. Loading states, the DOB→predicate transition
                  animation, error copy. Deploy. Test on the actual
                  judging wifi. Test on a phone.

    H32–34  ALL   REHEARSE THE DEMO OUT LOUD, TWICE, TIMED, ON THE REAL
                  HARDWARE. Every team that skips this discovers a bug
                  on stage. Every one.

    H34–36  ALL   buffer. Slides. Sleep if any remains.

    Hard rule: feature freeze at H28. Anything not working by H28 gets
    cut, not fixed. A working three-feature demo beats a broken
    five-feature one every single time, and judges cannot see the
    features that did not run.

---

## 10. Demo script — 4 minutes

    0:00  "Every time you open a financial account you upload your
           passport again. Every company that stores it becomes a
           breach waiting to happen. We verified once and made it
           portable — without anyone ever seeing the document."

    0:20  ENROL. Judge's own ID photo if they will give you one,
           otherwise fixture. Claude parses it live.
           → point at the screen: "date of birth, 1996"
           → next frame: "age_over_18: true"
           → "the date of birth is now gone. We never store it. It is
              not in the credential. There is no field for it."

    1:00  TAMPERED DOC. Second image. Anomalies populate, flow blocks.
           "Font inconsistency in the MRZ. Rejected."

    1:20  ONBOARD AT A BANK. Open Meridian Bank. It asks two questions.
           Wallet shows exactly those two. Approve. Account opens.
           "One point two seconds. They received two booleans."

    2:00  THE GREP. Terminal, live.
           grep -iE "name|dob|birth|address" → nothing.
           Hold the silence for a beat. This is the moment.

    2:20  REVOKE. Console, click revoke. Bank verifies the same
           credential again → revoked.
           "Under a second. Live system, not a video."

    2:45  AGENT. Agent books dinner at ₹420 against a ₹500 mandate →
           authorised. Tries ₹620 → declined, scope_exceeded.
           "Visa shipped this in October 2025. Mastercard in 2026. This
           is the layer agentic commerce is missing and it is the same
           credential."

    3:20  WHAT'S REAL, WHAT'S MOCKED. Say it plainly, on a slide.
           "Real: the cryptography, selective disclosure, key binding,
           revocation, mandate enforcement. Mocked: the HSM, the bank
           API, the sanctions list, liveness. We did not fake the part
           that matters."

    3:40  THE ASK / THE VISION. One line. For a bank-sponsored
           hackathon: "A closed-loop network already KYC'd both sides
           of every transaction. It is the only one that can issue
           this."

    Timing discipline: the grep at 2:00 must land. If you are running
    long, cut the tampered document beat, not the grep.

---

## 11. What will go wrong, and the pre-built escape hatch

    Wifi dies                → everything runs on localhost. Rehearse
                               the whole demo offline. Have the API,
                               all four apps and SQLite on one laptop.

    Claude API slow or down  → cache fixture responses by image hash and
                               add a DEMO_MODE=cached env flag that
                               serves from cache only. Flip it if the
                               call takes over 3 seconds on stage.

    State is dirty from      → demo-reset.ts bound to a keyboard
    rehearsal                  shortcut. Reseeds, clears credentials,
                               resets status list. Run it right before
                               you present.

    QR handoff between       → have a "simulate scan" button in the
    devices fails              wallet as a fallback path. Wire it early.

    Someone asks "why not    → "Immutable storage plus a GDPR erasure
    blockchain?"               right cannot coexist, and hashed PII is
                               still PII when the input space is small.
                               We anchor nothing. That is a decision."

    "Is this actually ZK?"   → "It is selective disclosure over salted
                               claim digests, which gives predicate
                               proofs. Full ZK-SNARKs would let us prove
                               statements over hidden values without a
                               trusted issuer — that is the next step,
                               not this weekend's."
                               Do not claim ZK-SNARKs. Someone in the
                               room will know.

    "What stops credential   → open devtools, show the CryptoKey with
    copying?"                  extractable:false. The private key cannot
                               leave the browser. Real property, real
                               proof, ten seconds.

---

## 12. Definition of done, in priority order

Cut from the bottom when time runs out. Never from the top.

    P0  issue → disclose one claim → verify → verified          MUST
    P0  no forbidden key can enter a credential (test passes)   MUST
    P0  the grep returns nothing                                MUST
    P0  demo runs fully offline                                 MUST

    P1  revoke → same credential → revoked, under 5s
    P1  holder binding rejects a wrong-key presentation
    P1  AI parses a real document and flags a tampered one

    P2  agent mandate authorised at cap, declined over cap
    P2  pairwise subject_ref differs across two RPs

    P3  console polish, animations, deployed URLs, PWA install

If you have P0 and P1 at H28, you have a strong project. If you have P0
only, you still have a demo that makes its point — and a team that shipped
something true beats a team that shipped something impressive-sounding
that falls over in Q&A.