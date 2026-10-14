# PrimeCredence, Explained Simply

> A plain-English walkthrough of what PrimeCredence is, what problem it solves, and how each
> piece actually works. No prior knowledge of digital identity assumed.
>
> For the dense technical reference, see [`primecredence-product-notes.md`](./primecredence-product-notes.md).

---

## Start here: the problem nobody solved

Imagine you're a customs officer. A truck arrives with a shipment of batteries. The driver hands
you a PDF certificate saying the batteries meet EU safety rules.

Now ask yourself three questions:

1. **Is this PDF real, or did someone edit it in Photoshop?**
2. **Even if it's real — was the lab that signed it actually allowed to certify batteries?**
3. **Is it still valid, or was it revoked last week?**

Today, you mostly can't answer any of these. You call someone. You email a lab. You wait. Or —
more honestly — you just trust the paper and wave the truck through.

This is the gap. We've digitised documents, but we never digitised *trust*. We turned paper into
PDFs, which is the same problem with better formatting.

**PrimeCredence is a system for making that paper provable instead of trusted.**

---

## The one idea that explains everything

If you remember one sentence from this document, make it this one:

> **Cryptography proves identity. Governance proves authority.**

These are two completely different things, and almost everyone conflates them.

**Cryptography** can tell you: *"This document was signed by the holder of key #A7F3."* That's a
mathematical fact. It's unforgeable. Great.

But it tells you **nothing** about whether key #A7F3 *should* have signed it.

Here's an analogy. Suppose you receive a signed letter from "Dr. Smith." A handwriting expert
confirms the signature is genuinely Dr. Smith's — that's cryptography. But you still don't know:

- Is Dr. Smith actually a licensed doctor?
- Is their licence current, or was it revoked?
- Are they licensed to practise *this* specialty?
- Does the hospital recognise them?

A genuine signature from an unauthorised person is still worthless. **Most "blockchain identity"
products stop at the signature check and call it done.** PrimeCredence treats the second
question — *was this entity allowed to say this?* — as a separate, equally rigorous, and itself
machine-checkable problem.

That's the whole product, really. Everything else is plumbing for that idea.

---

## The three building blocks

Before anything else makes sense, you need three concepts. They're simpler than their names
suggest.

### 1. DID — Decentralised Identifier

**What it is:** An ID for an organisation or person that *they* control, not a platform.

Your email address is controlled by Google. Your LinkedIn profile is controlled by LinkedIn. If
either company deletes your account, that identity is gone.

A DID is different: it's an identifier where you hold the cryptographic keys. Nobody can
impersonate you, and nobody can take it away.

PrimeCredence uses only two flavours, deliberately:

| Type | What it looks like | Think of it as |
|---|---|---|
| `did:web` | `did:web:lab.example.com` | A domain name you already control, used as an identity. If you own the domain, you own the identity. |
| `did:key` | A raw public key | A throwaway/bootstrap identity for temporary use |

The clever bit about `did:web`: there's no new registry to trust. If you can prove you control
`lab.example.com`, you control that identity. The existing DNS and HTTPS system does the heavy
lifting.

### 2. VC — Verifiable Credential

**What it is:** A digital document that is signed, tamper-evident, and machine-checkable.

A PDF certificate can be edited and nobody can tell. A verifiable credential cannot — change a
single character and the signature breaks instantly and visibly.

Think of it as a digital document with a **mathematical wax seal**: break the seal, and everyone
can see it was broken.

It also carries structured data (not just pixels), so a computer can read it and reason about it:
`issuer`, `subject`, `claims`, `expiry`, and where to check whether it's still valid.

### 3. Trust Registry

**What it is:** A directory that answers one question: *"Is this issuer allowed to issue this
kind of credential?"*

This is the "governance" half of the core idea, made concrete. When a verifier sees a credential
from `did:web:lab.example.com`, it asks the registry:

> "Is this lab accredited to issue battery conformity certificates under the EU ESPR framework?"

And gets back a yes/no that is itself signed and verifiable.

The protocol for asking is called **TRQP** (Trust Registry Query Protocol) — just a standard way
of phrasing that question so any verifier can ask any registry.

### And the fourth piece: ledger anchoring

PrimeCredence uses a permissioned blockchain, but **only to store hashes** — tiny fingerprints
that prove "this governance rule existed, unchanged, at this point in time."

> **Nothing personal or commercial ever goes on the ledger.** Not credentials, not invoices, not
> names. Just fingerprints and audit digests.

Why bother with a ledger at all? Because tamper-evidence only works if no single party can quietly
rewrite history. If PrimeCredence stored the audit log in its own database, PrimeCredence could
alter it. On a ledger, it can't — and neither can you, which is the point.

---

## The lifecycle: four verbs

Everything in the platform is one of four things:

```
   ISSUE  ──▶   HOLD   ──▶  VERIFY
     ▲                         │
     └────────  GOVERN  ◀──────┘
```

**Issue** — an organisation creates a credential and signs it.
*"This lab certifies that battery model X passed safety test Y."*

**Hold** — someone keeps it. Two kinds of holders:

| Holder | For whom | How it works |
|---|---|---|
| **PrimeCredence Wallet** | People | A phone app. Keys live in the phone's secure chip, protected by Face ID. |
| **Entity Wallet** | Organisations | Lives on the platform, one per tenant. Receives credentials from partners and can answer requests automatically under rules you set. |

**Verify** — someone checks it. This runs the 5-Check engine (next section).

**Govern** — the rules that define who may issue what, which the verify step consults.

The loop matters: governance isn't a document somebody wrote once. It feeds directly into every
single issuance and verification, at runtime.

---

## The 5-Check Verification Engine

This is the heart of the product. When a credential is presented, **five separate checks run, and
all five must pass**. There is no "mostly valid."

Walk through them with the customs officer example — scanning a battery's conformity credential:

### Check 1 — Identity Authentication
> *"Is this credential genuine, and is the person presenting it actually its holder?"*

Three things happen: the issuer's signature is verified against a key fetched from their `did:web`
domain; every disclosed claim is checked against the signed fingerprint; and the holder proves
they control the credential by answering a fresh random challenge from *this* verifier.

That last part matters — it stops someone from replaying a screenshot of someone else's credential.

**What this proves:** the credential is real and this person holds it.
**What it does NOT prove:** that the issuer had any right to issue it.

### Check 2 — Issuer Authorisation
> *"Was this lab actually accredited to certify batteries?"*

This is where the trust registry gets queried. This is the check that most systems skip entirely.

**A valid signature from an unaccredited issuer fails here.** That's the whole point.

### Check 3 — Schema Acceptance
> *"Is this the right kind of document, in the right shape?"*

The credential says it's a `BatteryConformityCredential`. Does that type resolve to an active,
published schema? And does *this verifier's policy* accept that schema as valid input?

This stops someone presenting a genuine, properly-issued credential that simply isn't the right
document for this situation — like showing a library card at airport security.

### Check 4 — Predicate Satisfaction
> *"Do the actual values meet my requirements?"*

Business rules: `recycled_content > 16%`, `age > 18`, `invoice_amount > threshold`.

Here's a neat trick: when the real value must stay secret, the **issuer attests the conclusion
instead of the raw fact**. Rather than disclosing your birth date, the issuer signs a claim that
simply says `over_18: true`. You reveal the answer, never the data behind it.

### Check 5 — Revocation Freshness
> *"Is it still valid right now — not just when it was issued?"*

Credentials get revoked. A lab loses accreditation; an invoice gets paid. The credential points to
a signed status list, and the verifier checks the current entry within a defined freshness window.

**Critical design decision: if the status list can't be reached, verification FAILS.**

That's called **failing closed**, and it's the opposite of how most systems behave. Most systems
shrug and let things through when a check is unavailable. Here, "I couldn't check" is treated as
"no" — because an attacker who can make your status server unreachable shouldn't thereby get a
free pass.

---

## Two kinds of credentials, two formats (and why)

PrimeCredence deliberately refuses to force everything into one format. There are two layers, and
they have genuinely different requirements.

|  | **Governance credentials** | **Business credentials** |
|---|---|---|
| **What they are** | The rulebook: who's accredited, what policies apply | The actual facts: invoices, certificates, diplomas |
| **Format** | W3C VCDM 2.0 | IETF SD-JWT VC |
| **Who reads them** | Verifier engines, auditors, regulators | Wallets, verifiers, consumers |
| **Priority** | Auditability and long-term provability | Holder privacy |
| **Revoked via** | Bitstring Status List | Token Status List |

**Why two?** Because the priorities are opposite.

*Governance credentials* are public rules. Everyone should see them, forever, and be able to
re-verify what the rules were on any past date. Privacy is irrelevant — transparency is the goal.

*Business credentials* are about you. If your diploma reveals your exact graduation date, employer
history and student ID every time you prove you have a degree, that's a privacy disaster. So
business credentials use **SD-JWT VC**, which has a genuinely elegant property:

### How selective disclosure actually works

Each claim in the credential is individually salted and hashed. The issuer signs the *list of
hashes*, not the values.

When you present the credential, you choose which values to reveal. The verifier hashes what you
gave them, confirms those hashes appear in the signed list — **and the signature still checks out
even though you withheld everything else.**

```
Issuer signs:    [ hash(name), hash(dob), hash(degree), hash(gpa) ]  ← signed once

You present:     degree = "BSc Computer Science"   ✓ revealed
                 name, dob, gpa                     ✗ withheld

Verifier:        hashes the degree → finds it in the signed list → signature valid ✓
                 learns nothing about the other three
```

No re-issuance needed. One credential, infinitely many partial views of it.

### And the third rail: BBS+

Selective disclosure has one remaining leak. If you show the same credential to two different
verifiers, they can compare notes and tell it was the same credential — the signature is identical
each time. That's a correlation risk.

**BBS+** solves it: it generates a *mathematically fresh proof every single time*. Two
presentations of the same credential look completely unrelated. Nobody can link them.

It's used where unlinkability genuinely matters — for example, a supplier presenting receivables
to multiple competing financiers who shouldn't be able to build a picture of that supplier's full
book.

---

## Governance as code, not PDFs

Here's an idea that sounds boring and is actually the most important architectural choice in the
product.

**The trust framework is not a document. It is a set of signed credentials that the verification
engine reads at runtime.**

Normally, a governance framework is a 200-page PDF. Someone reads it, interprets it, and writes
code that approximately implements it. The PDF and the code drift apart. Nobody can prove what the
rules were in March.

PrimeCredence encodes the rules as six credential types:

| Credential type | Answers the question |
|---|---|
| **Issuer Authorisation** | Who may issue what? |
| **Schema Governance** | What fields must a valid credential carry? |
| **Verification Policy** | What must a verifier check before accepting? |
| **Assurance Levels** | How rigorous is this issuer? (KYC tier, audit cadence) |
| **Status Management** | How is revocation handled, and how fresh must the check be? |
| **Interoperability Profile** | How do other frameworks recognise ours? |

Three consequences fall out of this, and they're each significant:

1. **The rules are machine-executable.** No interpretation gap between policy and implementation —
   the policy *is* the implementation.
2. **Every change is versioned, signed, and timestamped.** A regulator can ask "what were the
   rules on 3 March 2027, and did this verification comply with them?" — and get a provable answer.
3. **Policy changes take minutes, not release cycles.** Update the credential, publish, and every
   verifier picks it up on its next query.

**Assurance levels** deserve a mention: trust is *graded*, not binary. Opening a basic savings
account might accept a level-2 issuer; a mortgage might demand level-4. Same engine, different
threshold.

---

## Where things run: the sovereignty question

Banks and governments ask the same question first: *"Does our data leave our building?"*

PrimeCredence's answer splits the system into two halves along a deliberate line.

### Your side (sovereign — runs in your environment)
- The platform runtime
- The database with all credential and business data — **"we never see it"**
- Your API gateway
- Evidence storage (test reports, certifications, audit docs)
- Your KMS/HSM — your signing keys
- A metering agent that counts usage locally and reports signed numbers

### Shared side (so that anyone, anywhere, can verify)
- The trust registry (TRQP)
- The schema catalogue and status lists
- The identity resolver
- PrimeCredence Studio
- The audit ledger

### Why can't the shared half also be private?

This is the part worth understanding properly, because it's not a licensing trick — it's a logical
necessity.

Go back to the customs officer. They scan a QR code on a battery. For that scan to mean anything,
they need **one known place** to ask "is this issuer accredited?" If every company ran its own
private registry, the officer wouldn't know which one to ask — and a registry the verifier can't
reach is the same as no registry at all.

Same for schemas: a credential pointing at a schema that only resolves inside your corporate VPN
is, to the outside world, unverifiable. Same for the audit ledger: tamper-evidence means nothing
if the party being audited can rewrite the log.

**Verifiability requires reachability.** That's the constraint, and it's unavoidable.

> Only four things cross the boundary: **signed governance bundles in**; **registry queries,
> schema lookups, and signed usage counts out**. No credential payloads. No private keys. No
> personal data.

### Key custody — three options

| Mode | What happens | Trade-off |
|---|---|---|
| **Platform custody** (default) | Platform manages your signing keys in a KMS/HSM | Zero setup, fully unattended issuance |
| **Tenant-supplied key** | You supply the key that encrypts your signing keys | *Stored-wrapped* = still unattended; *session-supplied* = platform literally cannot decrypt at rest, but you must unlock before signing |
| **External KMS** | Keys stay in your AWS KMS or Azure Key Vault | Maximum control; platform never holds your key material |

Custody also fails closed: lose the key or lose contact with your KMS, and **signing stops**. It
never silently downgrades to a weaker key, which would be the dangerous default.

---

## The three things you actually touch

### 1. PrimeCredence Wallet — for people
A phone app, live on iOS and Android, free, no account needed to receive your first credential.

Keys live in the phone's Secure Enclave (iPhone) or Android Keystore — the same hardware that
protects your payment cards. They never leave the chip and are never in the cloud. Unlock is
biometric, and high-value credentials can demand biometrics individually.

It works offline (verification data caches locally), refreshes revocation status in the
background, and holds credentials from completely different issuers and trust frameworks side by
side.

**The whole flow:** install → scan a QR → receive the credential. That's it.

### 2. PrimeCredence Studio — for policy teams
A no-code workspace for the people who write the rules, so they don't need an engineering ticket
to change policy.

Organised into four groups:

- **Build** — trust frameworks, schemas, issuers & authorisations, identity authorities
- **Govern** — verifier policies, an approval inbox, the publish console
- **Operate** — credentials, verification, evidence, the registry, the audit trail
- **Insights & Org** — analytics, participants, users & access, integrations

The workflow is: **Author → Sign → Publish → Verify.** Design the framework visually, sign it with
HSM-backed keys through a guided flow, publish it to the registry, and verifiers pick it up at
runtime.

### 3. The SDK — for developers

```
npm install @primethoughts/primecredence-sdk
```

TypeScript, Node 20+, Apache-2.0, **zero runtime dependencies** (Node built-ins only). Published
with npm provenance via Sigstore + GitHub OIDC, so `npm audit signatures` can verify the package
was genuinely built from their source by their CI — a real supply-chain guarantee, not a claim.

The entire flow in about fifteen lines:

```ts
import { PrimeCredenceClient } from '@primethoughts/primecredence-sdk';

const pc = new PrimeCredenceClient({
  baseUrl: 'https://gateway.example.com',
  auth: { type: 'accessKey', keyId: 'ak_live_…', secret: process.env.PRIMECREDENCE_SECRET! },
});

// 1. Issue — the wallet scans the resulting QR to claim it
const offer = await pc.oid4vci.createOffer({
  credentialType: 'InvoiceCredential',
  claims: { invoiceNumber: 'INV-2041', aboveThreshold: true },
  selectiveDisclosureClaims: ['aboveThreshold'],   // ← only this is revealable
});

// 2. Request a presentation — ask for the minimum you need
const request = await pc.presentations.createOid4vpRequest({
  vct: 'urn:primecredence:vc:invoice:1',
  requestedClaims: ['aboveThreshold'],             // ← not the amount, just the fact
});

// 3. Verify — runs all 5 checks, fails closed
const result = await pc.presentations.verify(request.presentationId);
```

Notice what's happening in that example: the invoice amount is **never requested and never
revealed**. The financier learns only that the receivable clears their threshold.

*Caveat: there's no hosted API reference yet — the npm README is currently the canonical docs.*

---

## Standards: why "boring" is the feature

PrimeCredence's stated position: *"Nothing proprietary where a standard exists… so your credentials
outlive any vendor, including us."*

That's a strong claim, and the standards list backs it up:

| Standard | What it's for |
|---|---|
| **W3C VCDM 2.0** | Format for governance credentials |
| **W3C Data Integrity** (`eddsa-jcs-2022`) | The proof suite that signs them |
| **W3C Bitstring Status List** | Revoking governance credentials |
| **IETF SD-JWT VC** | Format for business credentials (selective disclosure) |
| **IETF Token Status List** | Revoking business credentials |
| **OID4VCI** | The protocol for issuing |
| **OID4VP + DCQL** | The protocol for presenting |
| **BBS+** (IRTF CFRG) | Unlinkable presentations |
| **`did:web` / `did:key`** | The only two identifier methods |
| **TRQP** | Querying the trust registry |
| **ToIP four-layer model** | The overall architecture frame |
| **UNTP** | UN model for product passports |
| **GS1 Digital Link / EPCIS 2.0** | QR resolution and traceability events — *demoable, not shipped* |

Note that last row: the site publishes a **shipped vs demoable** column rather than listing
everything as done. That's unusually honest for this category.

### The EUDI claim, stated precisely

The EU's digital identity wallet framework (eIDAS 2.0) specifies exactly three things: OID4VCI for
issuance, OID4VP for presentation, SD-JWT VC for format.

PrimeCredence business credentials use **exactly that stack — not an adaptation of it**. So a
PrimeCredence credential can be issued to, and verified from, an EUDI-conformant wallet with no
translation layer.

> **Their own caveat, stated openly:** this is compatibility *by construction*, not certification.
> PrimeCredence does not hold an eIDAS certification, and wallet conformance programmes are run by
> individual member states.

### On post-quantum — refreshingly candid

Their current crypto is all elliptic-curve: Ed25519, ES256, BLS12-381. **None of it is
post-quantum**, and they say so plainly.

Their position is **crypto-agility**: algorithms sit behind narrow signing interfaces as
configuration, rather than being baked as assumptions throughout the stack. The documented plan is
dual-signing (classical + ML-DSA/SLH-DSA side by side) once the underlying credential formats
standardise PQ suites.

Their words: *"We will not claim 'quantum-safe' before that is true."*

---

## Six places this gets used

### Supply chain finance
**Problem:** Financing reaches tier-1 suppliers. Tier-2, 3 and 4 businesses are invisible to
lenders — they can't prove the receivables they hold, and the data that would unlock credit
(invoice values, buyer names) is commercially sensitive. The result is a multi-trillion-dollar
trade finance gap.

**Solution:** Each receivable becomes a credential. The issuer attests `above_threshold: true`, so
a financier can confirm the receivable qualifies **without seeing the amount or the counterparty**.
BBS+ stops competing financiers correlating a supplier's presentations.

**Explicit non-claim:** no tokenisation. No ERC-721. A receivable is a credential; the ledger holds
only anchors. This dodges the legal ambiguity of on-chain asset tokenisation entirely.

### Banking
**Problem:** Every bank runs its own KYC. Customers fill the same forms repeatedly. Income proofs
arrive as scanned PDFs — slow, expensive, forgeable.

**Solution:** Reusable KYC credentials and issuer-attested `salary ≥ threshold` claims. It doesn't
replace your KYC stack; your existing stack emits its conclusions as credentials. Cross-bank
recognition becomes a governance question rather than N² bilateral integrations.

### Digital Product Passports
**Driver:** The EU's ESPR regulation makes product passports mandatory — **batteries from February
2027**, **textiles from 2028**.

**Solution:** UNTP-aligned passports assembled from upstream supplier credentials. The nice design
touches:

- A **consumer needs no wallet and no app** — the QR resolves to the passport's public tier,
  rendered in a plain browser
- **Tiered access** — regulators, recyclers and customs unlock deeper layers by presenting their
  own role credential. *The passport checks the checker.*
- **Incomplete is visible** — if an upstream supplier credential is missing, expired or revoked,
  the passport is flagged incomplete rather than quietly rendering a gap

### Food traceability (FDA FSMA 204)
**Driver:** FSMA 204 requires firms handling listed foods to track key data at each critical event
and hand the FDA a **sortable spreadsheet within 24 hours** of a request. Compliance date: July
2028.

**Solution:** Each tracking event (harvest, cool, pack, ship, receive, transform) becomes a signed
credential carrying the required data, issued by the party that actually performed it. Lot codes
travel as claims inside the credential, so they can't silently drift between hops.

The 24-hour export stops being an archaeology project and becomes a query.

**Their own honesty box:** nothing in FSMA 204 *requires* credentials. You could comply with
spreadsheets. What credentials add is tamper-evidence, signatures from the performing party, lot
codes that can't drift, and one supplier onboarding that serves every downstream buyer.

### Talent & skills
**Problem:** Résumés are self-asserted, certificates are forgeable, and proving your background
means over-sharing — exact dates, employers, identifiers — in ways that invite bias.

**Solution:** Attestations instead of histories. `more_than_3_years_experience: true` rather than a
full employment record. A verifier trusts the credential because of **governance, not brand
recognition** — they don't need to have heard of your university, only to check it's accredited
under a framework they recognise.

### Government & citizen wallets
Citizen-held credentials in a hardware-backed wallet, presented over the same OID4VP the EUDI
Wallet uses, against a regulator-operated trust registry with graded assurance levels per service.

The canonical example: an issuer-attested **`over_18: true`** claim, with no date of birth and no
document number revealed.

### (Adjacent) KnowMyAgents
A *second* PrimeThoughts product built on the same engine, giving AI agents verifiable identity and
guarded access. The logic: the same credentials and registries that make organisations provable
make their agents provable too.

---

## Honest assessment

### What's genuinely strong

**The core distinction is real and well-defended.** "Cryptography proves identity, governance
proves authority" isn't a slogan bolted onto a homepage — it's carried consistently through every
architectural decision on every page.

**The claim discipline is unusual for this category.** Explicit "we have no eIDAS certification."
Explicit "none of our crypto is post-quantum." A `shipped` vs `demoable` column. Case study metrics
labelled "modelled." A dedicated honesty box separating what the FDA rule requires from what the
product adds. Vendors in this space almost never do this.

**Standards depth is specific, not decorative.** Naming `eddsa-jcs-2022`, DCQL, BLS12-381 and an
ML-DSA/SLH-DSA transition path is the opposite of logo-wall standards-washing.

**Clear anti-hype stance on blockchain.** No tokens, no tokenisation, ledger for anchors only,
payloads off-chain. This is the correct engineering position and it's rarer than it should be.

**The sovereignty boundary is argued from first principles** — explaining *why* the registry must
be shared is the hard part of the sovereign-deployment pitch, and they do the work.

### What to probe

**No named customers. No audited results. No logos.** Every case study is explicitly labelled
*modelled* or *illustrative*. That's honest labelling of a real gap, but it's still a real gap.

**No published pricing at any granularity.** No tier table, no allowance units, no overage rates.
Everything requires a sales conversation.

**No certifications cited anywhere** — no SOC 2, no ISO 27001, no third-party audit — despite
having a page aimed squarely at enterprise security reviewers. For banks and government buyers
this is usually a hard gate.

**No company substance disclosed.** No founders, no team, no locations, no funding, no founding
date.

**Developer docs are immature.** The SDK is v1.0.0 and the npm README is still the canonical
reference; the hosted portal is "on its way."

**Maturity signals are thin overall.** "Demoable" on GS1, illustrative case studies, a demo portal
behind a sign-in wall. The architecture appears to be meaningfully further along than the
deployment base.

**Minor, but telling:** the site is a client-rendered SPA with only `/` prerendered. Every other
route serves the homepage shell to crawlers, which almost certainly hurts discovery of the
solution and standards pages — the very pages that carry their strongest differentiators.

---

## The thirty-second version

PrimeCredence makes digital documents **provable** instead of **trusted**.

It does this by separating two questions that everyone else collapses into one: *is this signature
real?* (cryptography) and *was this signer allowed to sign it?* (governance). Both are checked by
machines, at runtime, against rules that are themselves signed credentials rather than PDFs.

Five checks run on every presentation and all five must pass — and if any check can't be completed,
it fails rather than shrugging. Holders reveal only the specific claims a verifier asks for, so
privacy is a structural property rather than a policy promise. Your data and keys can stay entirely
in your environment; only the registry, schemas and anchors are shared, because verification by
strangers requires something strangers can reach.

It's built on open standards throughout — specifically the same stack the EU Digital Identity
Wallet mandates — so credentials remain verifiable even if PrimeCredence disappears.

**The architecture is genuinely strong and the honesty about its limits is unusual. The open
questions are all about commercial maturity — customers, pricing, certifications — not about the
engineering.**
