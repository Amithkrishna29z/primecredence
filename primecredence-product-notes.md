# PrimeCredence Trust OS — Product Notes

> Source: https://primecredence.com/ (all routes, extracted 2026-10-13)
> Vendor: **PrimeThoughts** · Tagline: **"Where interactions are provable"**

---

## 1. What it is

PrimeCredence is a **Trust Operating System (Trust OS)** — a platform that lets organisations
**issue, hold, verify, and govern digital credentials across organisational boundaries**.

It combines four things into one product:

1. **Decentralised identifiers (DIDs)** — who controls an identity
2. **Verifiable credentials (VCs)** — tamper-evident, machine-checkable claims
3. **Trust registries** — who is *authorised* to do what, under which framework
4. **Ledger anchoring** — tamper-evidence and audit, without storing payloads on-chain

### The core thesis (the product's central argument)

> **Cryptography proves identity. Governance proves authority.**

A valid signature only proves that *some key controls some identifier*. It says nothing about
whether that identifier is *allowed* to issue this credential, attest this fact, or sign for this
organisation. PrimeCredence treats that second question as a separate, governed, and itself-verifiable
decision — handled by the trust registry and policy engine.

This framing is the product's main differentiator versus "blockchain identity wallets": it is not a
wallet or a token, it is a **governed platform**, and the ledger is used *only* to anchor governance
hashes and audit digests — never to store personal data.

### Headline stats used on the site

| Figure | Meaning |
|---|---|
| **5-Check** | Verification engine stages |
| **4-Layer** | ToIP (Trust over IP) trust model |
| **2** | Credential layers — W3C VCDM 2.0 & SD-JWT VC |
| **9+** | Open standards implemented |

---

## 2. The credential lifecycle — Issue / Hold / Verify / Govern

The product is organised around a four-stage, policy-driven loop. Governance policy feeds back into
every issuance and verification.

### Issue
- **Governance credentials** → W3C VCDM 2.0 with **Data Integrity Proofs**
- **Business credentials** → **IETF SD-JWT VC** issued over **OID4VCI**
- **BBS+ rail** where unlinkability is required

### Hold
Two holder types on the same rail:

| Holder | Who it's for | Notes |
|---|---|---|
| **PrimeCredence Wallet** | People | Mobile (iOS/Android), hardware-backed keys, biometric protection |
| **Entity Wallet** | Organisations | Platform-side holder per tenant; receives partner-issued credentials and answers OID4VP requests automatically under rules you set, or holds for review. Holder keys are per-tenant and never leave the platform key vault. One Entity Wallet can serve several trust networks. |

### Verify
Every presentation runs the **5-Check engine** (see §4).

### Govern
Trust frameworks defined through governance VC types, lifecycle state machines, and policy-driven
verification rules (see §5).

---

## 3. Platform capabilities (8 core modules/capabilities)

1. **Dual-Layer Credential Architecture** — each layer uses the standard built for it: W3C VCDM 2.0
   + Data Integrity Proofs for governance; IETF SD-JWT VC (OID4VCI in, OID4VP out) for business
   credentials; plus BBS+ for unlinkable presentations.
2. **Trust Registry & Governance** — who is authorised to do what under which framework. Governance
   VC types, accreditation chains, and the **Trust Registry Query Protocol (TRQP)**.
3. **PrimeCredence Studio** — no-code workspace for authoring trust frameworks, signing governance
   credentials, publishing schemas and status lists, configuring verification policy.
4. **5-Check Verification Engine** — policy-driven five-stage pipeline.
5. **Credential Status & Schemas** — IETF Token Status List hosting + registry-served JSON Schema
   catalogue. Tenant-scoped, signed, **fail-closed in production** so a verifier never mistakes an
   unreachable list for a valid credential.
6. **Ledger Anchoring** — a permissioned ledger anchors governance-credential hashes and audit
   digests; identity resolves via `did:web` / `did:key`. **Proofs and anchors only — never personal
   or commercial data.**
7. **Privacy by Construction** — SD-JWT selective disclosure + BBS+ unlinkable presentations. Data
   minimisation and consent built into every flow.
8. **Multi-Tenant Enterprise Security** — strict tenant isolation, BYOK custody with external
   KMS/HSM, crypto-agile design with a documented post-quantum transition strategy.

### 3.1 The dual-layer credential model (detail)

PrimeCredence deliberately does **not** force every credential into a single format.

| | **Governance VCs** | **Business VCs** |
|---|---|---|
| **Format** | W3C VCDM 2.0 | SD-JWT VC (+ BBS+) |
| **DID method** | `did:web` | `did:web` / `did:key` |
| **Proof** | Data Integrity Proof | JWS + key-binding JWT / BBS+ derived proof |
| **Exchange** | Trust registry (TRQP) | OID4VCI / OID4VP |
| **Status** | Bitstring Status List | Token Status List |

- **Governance credentials** are the accreditations and authorisations that define who can do what.
  Signed with Data Integrity Proofs under a `did:web` steward, revoked via W3C Bitstring Status List.
- **Business credentials** are where holder privacy matters most. Every claim is individually salted
  and hashed into the issuer's signature, so the holder reveals only the claims a verifier asks for
  and the signature still checks. Revoked via the signed **IETF Token Status List** — the rail the
  **EU Digital Identity Wallet (EUDI)** framework specifies.
- Where two presentations must not be linkable, a **BBS+ rail** on the same protocols derives a
  fresh proof every time.

### 3.2 Anchors & status — what's on-ledger vs on the web

| **Permissioned ledger** (anchors only, never content) | **Resolved over HTTPS** (public, signed, verifiable by anyone) |
|---|---|
| Governance-credential hashes | `did:web` documents (issuer keys) |
| Audit & billing Merkle roots | JSON Schema catalogue (registry) |
| Governance state transitions | Token Status List & Bitstring Status List |

> Personal and commercial data live **off-ledger, tenant-scoped** — on neither side.

---

## 4. The 5-Check Verification Engine

Every presentation is evaluated by a policy-driven, five-stage pipeline. **All five must pass.**
There are no partial-trust outcomes — it **fails closed**.

| # | Check | What it actually does |
|---|---|---|
| 1 | **Identity Authentication** | Confirms the credential is genuine and holder-bound: issuer signature verifies against a key resolved from its `did:web`, every disclosed claim matches the signed digest, and the holder's key-binding proof answers *this* verifier's nonce. Proves possession — not yet authority. |
| 2 | **Issuer Authorisation** | Checks the issuer against the trust registry: is this issuer accredited to issue this credential type under the governing framework? *Signature validity is necessary but never sufficient.* |
| 3 | **Schema Acceptance** | Verifies the credential type resolves to an active schema in the registry catalogue, and that the verifier's policy accepts it. Policy packs declare exactly which schemas are valid inputs. |
| 4 | **Predicate Satisfaction** | Evaluates required claims/predicates (e.g. `amount > threshold`, `over 18`) over the claims the holder chose to disclose. Where the underlying value must stay private, the **issuer attests the threshold fact as its own claim** and the holder discloses only that. |
| 5 | **Revocation Freshness** | Reads the credential's entry in a signed IETF Token Status List (Bitstring Status List for governance credentials) within the policy's freshness window. **In production, an unverifiable list fails closed.** |

---

## 5. Trust frameworks — governance encoded as verifiable credentials

The trust framework is **not a PDF** — it is a set of credentials the verification engine can read.
Six governance credential types:

| Type | Role |
|---|---|
| **Issuer Authorisation** | Declares who is accredited to issue specific credential types under a framework. Checked first (after identity authentication) by the 5-Check engine. |
| **Schema Governance** | Binds a schema definition to a trust framework — what attributes a credential must carry to be valid. |
| **Verification Policy** | Defines exactly what a verifier must check — predicate requirements, identity-anchor gating, schema acceptance, assurance level. |
| **Assurance Levels** | Certifies an issuer's compliance grade (e.g. KYC tier, audit cadence). High-stakes services can demand a higher level — verification is **graded, not all-or-nothing**. |
| **Status Management** | Defines revocation methods and freshness requirements per credential type (the 5th check). |
| **Interoperability Profile** | Publishes TRQP endpoints, registry profiles, and UNTP mappings so credentials stay verifiable across recognising frameworks. |

### Governed lifecycle
**Author** (programme team designs in Studio) → **Sign** (KMS/HSM-backed keys, every signing event
auditable) → **Publish** (to the Trust Registry; verifiers query via TRQP at runtime) → **Verify**.

Because the framework is itself signed credentials, every change is **versioned, signed and
timestamped** — regulators can verify framework state at any point in time, including historic
verifications against historic policy.

---

## 6. Architecture — the ToIP four-layer model

Every design decision maps to one of the four Trust over IP layers.

| Layer | Name | PrimeCredence implementation |
|---|---|---|
| **1** | **Public Utilities** | A permissioned ledger anchors governance-credential hashes, audit digests and state transitions — proofs and anchors only, never payloads. Everything a verifier must resolve (issuer keys, schema catalogue, signed status lists) is served over HTTPS so anyone, anywhere, can check it. |
| **2** | **DID Methods** | `did:key` for bootstrap keys, `did:web` for DNS-anchored organisational identity. Every issuer signs under its own `did:web`, key in its own custody. BYOK custody. |
| **3** | **Credential Exchange** | OID4VCI carries issuance, OID4VP carries presentation, over plain HTTPS. Governance = W3C VCDM 2.0; business = SD-JWT VC with per-claim selective disclosure, or BBS+ where presentations must be unlinkable. |
| **4** | **Governance** | Trust registries publish accreditation and policy: who is authorised to do what, under which framework, at what assurance level. |

---

## 7. Products / surfaces

### 7.1 PrimeCredence Wallet (`/platform/wallet`) — for people
Mobile wallet, **live on iOS and Android**. Free, no account needed to receive a first credential.

- **App Store:** `https://apps.apple.com/in/app/primecredence-wallet/id6792476123`
- **Google Play:** `https://play.google.com/store/apps/details?id=io.primecredence.wallet`

**Core features**
- **Hardware-backed key storage** — keys live inside the device Secure Enclave / Android Keystore;
  never leave the HSM; nothing in the cloud
- **Selective disclosure** — reveal only requested claims (e.g. "qualification met")
- **Standards-native exchange** — OID4VCI issuance, OID4VP presentation, plain HTTPS. No proprietary
  agent, no side channel
- **Biometric authentication** — Face ID / Touch ID / Android biometrics, with per-credential
  biometric requirements available for high-value credentials

**Advanced / long-term features**
- Multiple credentials from different issuers and trust frameworks, side by side
- **Offline presentation** — verification data caches locally
- **Background freshness** — revocation status updates in background, with visual indicators
- **Encrypted backup & recovery** — recovery via biometric re-auth on a new device, no cloud key exposure
- **Cross-ecosystem** — works with any PrimeCredence-powered ecosystem; presents to any
  TRQP-compliant verifier

**Onboarding:** Install → Scan a QR (credential offer or presentation request) → Receive credentials.

**Security posture:** private keys never leave the secure element; no cloud storage of credentials or
keys; every issuance and presentation over HTTPS; tamper-evident credential storage; no vendor lock-in.

### 7.2 PrimeCredence Studio (`/platform/studio`) — no-code governance workspace
For policy/programme teams to run trust-framework work **without engineering tickets**.

**What it does**
- **Author trust frameworks** visually — accredited issuers, schemas, verification policies, assurance levels
- **Sign governance credentials** through a guided flow, KMS/HSM-backed signing in production
- **Configure verification policy** — predicate requirements, identity-anchor gating, schema
  acceptance; "update policy in minutes, not weeks"
- **Manage credential schemas** — author, publish and version schemas in the registry-served JSON
  Schema catalogue; manage the status lists verifiers check
- **Observability & audit** — real-time issuance / verification / revocation events with a
  tamper-evident audit trail

**Module groups (as shipped):**
- *Build* — Trust Frameworks, Schemas, Issuers & Authorisations, DIA Authorities
- *Govern* — Verifier Policies, Approval Inbox, Publish Console, Assets
- *Operate* — Credentials, Verify, Evidence, Trust Registry, Audit Trail
- *Insights & Org* — Analytics, Participants, Users & Access, Identity & Wallet, Integrations, Settings

**Audiences:** programme owners · compliance & policy teams · security teams (KMS/HSM custody, RBAC,
full audit trail) · operators (live monitoring).

### 7.3 SDK & APIs (`/for/developers`)
- **Package:** `@primethoughts/primecredence-sdk` (npm)
- **Version / licence:** v1.0.0 · Apache-2.0
- **Runtime:** TypeScript, **Node 20+**, fully typed, **zero runtime dependencies** (Node built-ins only)
- **Supply-chain security:** published with **npm provenance via Sigstore + GitHub OIDC** —
  `npm audit signatures` verifies the tarball was built from their source by their CI
- **Surface:** twelve typed service modules — identity, OID4VCI issuance, OID4VP presentations,
  credentials & status, trust registry, evidence, wallet, billing, compliance
- **Auth:** JWT (as a user) or access key (backend integration); one client, tenant-scoped by credentials
- **Docs status:** a hosted developer portal with full API reference is "on its way"; until then the
  **npm README is the canonical reference**

```ts
import { PrimeCredenceClient } from '@primethoughts/primecredence-sdk';

const primecredence = new PrimeCredenceClient({
  baseUrl: 'https://gateway.example.com',
  auth: { type: 'accessKey', keyId: 'ak_live_…', secret: process.env.PRIMECREDENCE_SECRET! },
});

// Issue an SD-JWT VC over OID4VCI — the wallet redeems the offer QR
const offer = await primecredence.oid4vci.createOffer({
  credentialType: 'InvoiceCredential',
  claims: { invoiceNumber: 'INV-2041', aboveThreshold: true },
  selectiveDisclosureClaims: ['aboveThreshold'],
});

// Ask for a presentation over OID4VP — only the claims you need
const request = await primecredence.presentations.createOid4vpRequest({
  vct: 'urn:primecredence:vc:invoice:1',
  requestedClaims: ['aboveThreshold'],
});

// After the wallet answers: run the policy-driven 5-Check engine
const result = await primecredence.presentations.verify(request.presentationId);
```

---

## 8. Deployment & sovereignty (`/platform/deployment`)

Two models, one principle: *everything that is yours — runtime, data, keys — can live in your
environment, while shared trust infrastructure keeps your credentials verifiable by anyone.*

| | **Managed PaaS** | **Fully Sovereign** |
|---|---|---|
| Who operates | PrimeThoughts hosts and operates | You host runtime, data and keys |
| Fit | Most deployments; fast onboarding, no ops team | Banks, government, regulated entities |
| Extras | Regionally configurable hosting; optional bring-your-own-HSM for key sovereignty with managed compute | You run a small ops team; PrimeThoughts provides governance tooling, shared registry access, SDK |

You can **move between models** as the programme matures — "the standards don't change, only who
operates the machines."

### The sovereignty boundary

**Sovereign (runs in your environment):** platform runtime · application database (credential and
business data — "we never see it") · API gateway · evidence storage (certifications, test reports,
audit documents in your object storage) · KMS/HSM · metering agent (counts usage locally, submits
signed event counts — no credential content, no personal data).

**Shared (so anyone can verify)** — and *why* it must be shared:
- **Trust Registry (TRQP)** — a customs officer scanning a QR needs ONE place to check issuer
  authorisation; fragmented registries mean verifiers don't know where to ask
- **Schema catalogue & status lists** — a credential referencing a schema that resolves only on a
  private network is unverifiable by third parties
- **Identity resolver** — product QR codes encode public URLs; a consumer in a shop cannot be routed
  into a private corporate network
- **PrimeCredence Studio** — governance is authored centrally and distributed as signed bundles, so
  frameworks cannot silently diverge
- **Audit ledger** — tamper evidence only works on a ledger no single party can rewrite

> Only four kinds of data cross the boundary: **signed governance bundles in**; **registry queries,
> schema resolutions and signed usage counts out**. No credential payloads, no private keys, no
> personal data.

### Key custody — three modes, selectable per tenant

| Mode | Behaviour |
|---|---|
| **Platform custody** (default) | Signing keys protected by platform-managed encryption, KMS/HSM in production. Zero setup, unattended issuance. |
| **Tenant-supplied key** | You supply the key that protects your signing keys. *Stored-wrapped* keeps issuance unattended; *session-supplied* means the platform cannot decrypt at rest — you unlock before signing. |
| **External KMS** | Point at a key in your own AWS KMS or Azure Key Vault. The platform delegates wrap/unwrap to your KMS and never holds your key material. |

Custody **fails closed**: a lost tenant key or unreachable KMS stops signing rather than silently
falling back to a weaker key. Every mode change passes a wrap-and-unwrap self-test before activating.

---

## 9. Security & compliance (`/how-it-works/security`)

**Posture by default:** fail-closed verification · crypto-agile · sovereign-ready.

- **Hardware-backed key custody** — production signing keys live in KMS/HSM and never leave;
  ephemeral keys only for development; wallet uses Secure Enclave / Keystore for holder keys
- **Tenant isolation by default** — enforced on every query and operation: token-based access
  control, isolated credential stores, independent governance config per tenant
- **Privacy by design** — selective disclosure built in, BBS+ unlinkable presentations where
  correlation must be impossible; no new personal-data surfaces created by default
- **On-ledger data minimisation** — proofs and anchors only; payloads off-ledger, tenant-scoped,
  access-controlled
- **End-to-end auditability** — every issuance, verification and revocation is an evented record;
  audit trails verifiable **independently of the platform operator**

**Four operating principles**
1. **No silent escape hatches** — there is no "do something outside the framework" path; every
   operation flows through governance
2. **Crypto-agility** — signing algorithms and key parameters are configurable
3. **Fail closed** — if any of the 5 checks cannot be satisfied, verification fails. No partial trust
4. **Sovereign deployment** — data residency, key custody and operational control stay with you

---

## 10. Standards & interoperability (`/how-it-works/standards`)

Stated position: *"Nothing proprietary where a standard exists… so your credentials outlive any
vendor, including us."* Statuses are published honestly — **shipped** = in the platform today,
**demoable** = working in guided demos.

| Standard | Body | What PrimeCredence does with it | Status |
|---|---|---|---|
| **W3C Verifiable Credentials Data Model 2.0** | W3C | Format of every governance credential | shipped |
| **W3C Data Integrity (`eddsa-jcs-2022`)** | W3C | Proof suite on governance credentials, signed under the steward's `did:web` | shipped |
| **W3C Bitstring Status List** | W3C | Revocation/suspension for governance credentials — one bit per credential, checked without correlating holders | shipped |
| **IETF SD-JWT VC** | IETF | Format of every business credential; each claim individually salted and hashed into the signature | shipped |
| **OID4VCI** | OpenID Foundation | The only way business credentials are issued — offer, holder proof-of-possession, credential. Batch issuance gives wallets single-use copies for unlinkability | shipped |
| **OID4VP + DCQL** | OpenID Foundation | The only way business credentials are presented: signed request names the claims needed (DCQL), wallet answers bound to verifier's nonce, 5-Check evaluates | shipped |
| **IETF Token Status List** | IETF | Revocation for SD-JWT VCs — signed, compressed lists served by the registry; verifiers pin the signing key and fail closed in production | shipped |
| **BBS+ signatures (IRTF CFRG draft)** | IRTF | The unlinkability rail — fresh zero-knowledge proof per presentation on the same OID4VCI/OID4VP protocols | shipped |
| **`did:web` and `did:key`** | W3C DID Core | The only DID methods: `did:web` anchors issuers/stewards to domains they control; `did:key` covers bootstrap keys | shipped |
| **TRQP (Trust Registry Query Protocol)** | Trust over IP | The authorisation layer: "may this issuer issue this credential type under this framework?" | shipped |
| **ToIP four-layer model** | Trust over IP | The architecture frame — utilities, DID methods, credential exchange, governance as separate auditable layers | shipped |
| **UNTP (UN Transparency Protocol)** | UN/CEFACT | The Digital Product Passport model — traceability events, facility records, conformity credentials, the passport | shipped |
| **GS1 Digital Link & EPCIS 2.0** | GS1 | QR codes resolve through the identity resolver via GS1 Digital Link; traceability events map to EPCIS 2.0 vocabulary | **demoable** |

Also referenced: **eIDAS 2.0 / EUDI Wallet ARF**, **EU ESPR**.

### EUDI compatibility — the precise claim
eIDAS 2.0 establishes the EU Digital Identity Wallet; its Architecture and Reference Framework
specifies OID4VCI (issuance), OID4VP (presentation), and SD-JWT VC (format). PrimeCredence business
credentials use **exactly this stack — not an adaptation of it** — so a PrimeCredence ecosystem can
issue to and verify from EUDI-conformant wallets without translation layers.

> **Caveat they state openly:** this is protocol and format compatibility *by construction*.
> PrimeCredence does **not** hold an eIDAS certification, and wallet conformance programmes are run
> by member states.

### Post-quantum position (notably candid)
Current suites are elliptic-curve: **Ed25519** Data Integrity proofs on governance credentials,
**ES256** across the OpenID rail, **BLS12-381** for BBS+. **None are post-quantum.** The commitment
is crypto-agility — algorithms are configuration behind narrow signing interfaces rather than
assumptions baked through the stack. Documented transition strategy is **dual-signing** (classical +
NIST-standardised PQ signature, **ML-DSA / SLH-DSA**, side by side) once the credential formats they
build on standardise PQ suites. *"We will not claim 'quantum-safe' before that is true."*

---

## 11. Solutions (six verticals + one adjacent product)

### 11.1 Supply Chain Finance (`/solutions/supply-chain-finance`) — *Trade Finance*
**Pitch:** Turn receivables into verifiable credentials and unlock deep-tier financing, with
selective disclosure that protects commercial terms.

**Problem:** Financing reaches tier-1 suppliers, but tier-2/3/4 businesses are invisible to lenders.
They can't prove the receivables they hold, and the data that would unlock credit (invoice values,
buyer relationships) is commercially sensitive. Result: a multi-trillion-dollar trade finance gap.

**Approach:**
- **Receivables as credentials** — each invoice/receivable issued as a VC bound to buyer, seller and
  amount; portable across financiers and platforms
- **Threshold claims, not invoice data** — issuer-attested threshold claim proves "this receivable
  exceeds the financing threshold" without revealing the amount or counterparty; BBS+ where
  financiers must not correlate presentations
- **Accredited identity anchoring** — suppliers anchored to a governed identity authority (DIA)
- **Deep-tier reach** — financing cascades beyond tier-1

**Capabilities:** threshold claims with selective disclosure (SD-JWT VC) · accredited
identity-anchored suppliers · 5-Check on every receivable presentation · trust framework defining
accredited financiers and issuers · BBS+ unlinkable org-to-org presentations · revocation/status
tracking for settled receivables.

**Explicit non-claim:** *no tokenisation.* No ERC-721 or similar. A receivable is a verifiable
credential; the ledger anchors only governance hashes and audit digests — keeping commercial data
off-chain and avoiding the legal ambiguity of on-chain tokenisation.

### 11.2 Banking (`/solutions/banking`) — *Retail & Corporate Banking*
**Pitch:** Reusable KYC and verifiable credit attestations; privacy-preserving onboarding with
regulatory audit built in. *"Onboarding done once. Verification every time."*

**Problem:** Every bank runs its own KYC; customers refill the same forms; regulators audit paper.
Income/employment proofs arrive as scanned PDFs — slow, expensive, forgeable.

**Approach:** reusable KYC credentials · verifiable income & employment (issuer-attested
`salary ≥ threshold` claims) · privacy-preserving customer profiles · auditable regulatory reporting.

**Positioning:** does **not** replace existing KYC systems — sits alongside, and your existing stack
emits its assertions as verifiable credentials. Cross-institution recognition becomes **governance,
not bilateral integration**.

### 11.3 Digital Product Passports (`/solutions/digital-product-passports`) — *EU Compliance*
**Pitch:** UNTP-aligned DPPs meeting EU ESPR mandates with an immutable, ledger-anchored lifecycle record.

**Regulatory driver:** EU Ecodesign for Sustainable Products Regulation (ESPR) makes DPPs mandatory —
**batteries from 2027** (Regulation 2023/1542, February 2027), **textiles from 2028**, more to follow.

**Approach:** UNTP data objects (DTE, DFR, DCC, DPP) · ledger-anchored lifecycle · multi-party supply
chains via network anchors + participants · verifiable conformity (test results, certifications,
declarations of conformity as VCs checked by the 5-Check engine, not PDFs that must be trusted).

**Product depth notes:**
- **Consumer access needs no wallet and no app** — the QR resolves through the identity resolver to
  the **public tier** of the passport, rendered in the browser
- **Tiered access** — public tier for shoppers; regulators, recyclers and customs unlock deeper tiers
  by presenting their own role credential over OID4VP ("the passport checks the checker")
- **Battery passports** — demoable; assembles the regulation's data points from supplier credentials
  (model/manufacturer identity, technical spec, carbon footprint, recycled-content shares, hazardous
  substances), each field sourced from the DTE, DFR or DCC that attests it
- **Textile passports** — same assembly model (fibre composition, facility records, conformity
  claims); demonstrated in an apparel click-through demo
- **One aggregation, many issuers** — the assembly engine aggregates upstream supplier credentials
  into one passport credential signed by the economic operator, and **flags the passport incomplete**
  when an upstream credential is missing, expired or revoked

**Explicit non-claim:** product data is **not** on the blockchain — only hashes and anchors.

### 11.4 Food Traceability (`/solutions/food-traceability`) — *FDA FSMA 204*
**Pitch:** *"FSMA 204, answered in hours — not weeks."*

**Regulatory driver:** FSMA 204 (21 CFR Part 1, Subpart S) — anyone who manufactures, processes,
packs or holds foods on the Food Traceability List must keep **Key Data Elements** for defined
**Critical Tracking Events**, assign and pass **traceability lot codes**, and hand the FDA a
**sortable electronic spreadsheet within 24 hours** of request. Compliance date **July 2028**
(extended).

**Approach:**
- **Tracking events as credentials** — CTEs (harvesting, cooling, initial packing, first land-based
  receiving, shipping, receiving, transformation) each become a Digital Traceability Event credential
  carrying the rule's KDEs, signed by the party that performed it
- **Lot codes that travel** — the lot code is a claim inside each event credential, so it cannot
  silently change between hops
- **The 24-hour export** — because every event is structured and signed, the export is a query, "not
  an archaeology project"
- **EPCIS-mapped, supplier-friendly** — events map to GS1 EPCIS 2.0; suppliers issue from their own
  wallets

**"Honesty box" framing** (the site separates rule from product deliberately): nothing about FSMA 204
requires credentials — you could comply with spreadsheets. What credentials *add*: tamper-evident and
portable records signed by the performing party and checked by the 5-Check engine; lot codes that
can't drift; exports generated from verified data instead of assembled from inboxes; and one supplier
onboarding that serves every downstream buyer.

**Positioning:** does not replace ERP/WMS; suppliers need no new infrastructure (join as network
participants, issue from their wallet).

### 11.5 Talent & Skills Ecosystems (`/solutions/talent-ecosystems`) — *Education & Skills*
**Pitch:** *"Skills you can prove, privacy you keep."*

**Problem:** Résumés are self-asserted, certificates are forgeable, verification is manual — and
proving your background usually means over-sharing (exact graduation dates, employers, identifiers)
in ways that invite bias and breach privacy.

**Approach:** issued skill credentials (courses, assessments, employers) · **attestations, not
histories** (prove "more than three years of experience" or "assessed above the required level" as a
selectively disclosed issuer-attested claim) · learner-held wallets · governed by frameworks that
define which issuers are accredited for which skills — so a verifier trusts a credential **because of
governance, not brand recognition**.

### 11.6 Government & Citizen Wallet (`/solutions/government`) — *Public Sector*
**Pitch:** Citizen-held credentials plus a regulator-governed trust registry. *"Digitising a PDF is
not enough."*

**Approach:** citizen-held credentials in the PrimeCredence Wallet (hardware-backed, biometric,
presented over the same OID4VP the EUDI Wallet uses) · regulator-grade trust registry publishing
accreditation and policy · **graded assurance levels** per service · auditable by design (evented,
anchored records without centralising citizens' personal data).

Example of minimal disclosure: an issuer-attested **"over 18"** claim, without revealing date of
birth or document number.

### 11.7 AI Agent Governance — **KnowMyAgents** (adjacent product)
Not part of PrimeCredence proper: **KnowMyAgents** is PrimeThoughts' *second* product built on the
same PrimeCredence engine — verifiable identity, **guarded access**, and tamper-evident proof for AI
agents doing real work in enterprises. "The same credentials, registries and verification that make
organisations provable make their agents provable too." Early access at **knowmyagents.com**.

---

## 12. Audience pages

### For Enterprises (`/for/enterprises`)
Six non-negotiables: enterprise-grade key custody (KMS/HSM signing, tenant isolation, no shared
secrets or shadow key material) · privacy by design not bolt-on · built to interoperate · auditable
end to end · governance not just signatures · operates at enterprise scale (multi-tenant isolation,
observability, high-volume policy evaluation).

**Questions enterprise reviewers ask (their answers):**
- *Where does the data live?* On-ledger: proofs and anchors only. Off-ledger: payloads, governance
  metadata, tenant data. Managed or fully sovereign — data can stay in your environment.
- *How do we control issuance and verification?* Trust frameworks declare accredited issuers,
  accepted schemas and verification policy; anything outside the framework is rejected by the 5-Check
  engine.
- *How does it integrate with our identity stack?* Identity-stack-agnostic. Visitors and employees can
  be anchored through existing IdPs; governance credentials handle the trust-framework layer.
- *What about post-quantum?* Crypto-agile by design; HSM-backed key custody; configurable signing
  algorithms allowing migration as PQ primitives mature.

### For Regulators & Public Sector (`/for/regulators`)
Pillars: governance not just identity (the framework is itself verifiable, "not a PDF") · graded
assurance levels · auditable by design · citizen-held and privacy-preserving · interoperable across
borders.

Where it fits: **national identity programmes** (sovereign citizen wallet + regulator-operated trust
registry) · **sectoral oversight** (financial, health, education regulators define the accreditation
chain) · **cross-border compliance** (recognise credentials issued under another framework via
federation, without giving up audit, assurance grading or sovereign data residency).

### For Developers (`/for/developers`)
See §7.3.

---

## 13. Pricing (`/pricing`)

**No public price list.** Pricing is tailored per ecosystem — "talk to us for a proposal that fits."

**Stated model — four principles:**
1. **A predictable platform fee** — one flat fee covers every deployment; no per-seat surprises, no
   hidden tiers
2. **Generous allowances included** — credential issuance, verification, consumer scans and evidence
   storage all carry substantial allowances in the platform fee; "most deployments stay well within them"
3. **Rates that decline with scale** — beyond allowances, usage rates fall as volume rises
4. **Flexible deployment** — managed or sovereign; pricing adapts to the model

**Included in every deployment:** one predictable platform fee · generous usage allowances ·
declining rates as you scale · managed or sovereign hosting · **every platform capability included**.

> **Evaluation note:** there is no published $ figure, no tier table, and no usage-unit definition on
> the site. Allowance sizes, overage rates and the sovereign-vs-managed price delta all require a
> sales conversation.

---

## 14. Company

- **Product:** PrimeCredence (Trust OS) · **Parent:** **PrimeThoughts**
- PrimeThoughts is described as *"a technology leadership firm that helps ambitious organisations turn
  complex technology into business leverage, blending strategy, architecture, and execution across
  AI, Blockchain, and Cloud."*
- **PrimeThoughts' mission:** "To make complex technologies simple, secure, and actionable — bridging
  the gap between strategy and execution, so every solution earns its place in the real world."
- **PrimeCredence mission:** replace *assumed* trust with *provable* trust — every claim
  cryptographically verifiable, every authorisation explicitly governed, people keeping control of
  their own data.
- **Product portfolio named:** PrimeCredence · KnowMyAgents · **PrimeCompass**
- **Principles:** privacy by design · open standards · interoperability · enterprise-grade security
- © 2026 PrimeThoughts

**Not disclosed on the site:** named founders or team, office locations, founding date, legal entity
details, customer names, headcount, funding.

### Contact & links

| | |
|---|---|
| Email | **hello@primethoughts.com** |
| Contact form | `/contact` (name, email, organisation, use case, message, privacy consent) |
| Demo portal | https://sales.primecredence.com (sign-in required) |
| SDK | https://www.npmjs.com/package/@primethoughts/primecredence-sdk |
| GitHub | https://github.com/primethoughts |
| LinkedIn | https://www.linkedin.com/company/primethoughts/ |
| X | https://x.com/PrimeThoughtsAI |
| Privacy / Terms / Cookies | hosted on `primethoughts.com` |
| Sister product | https://knowmyagents.com |

**Contact form "use case" options** (a decent proxy for their target segments): Supply Chain Finance ·
Banking · Digital Product Passports · Food Traceability (FSMA 204) · Talent & Skills Ecosystems ·
Government & Citizen Wallet · PrimeCredence Wallet · PrimeCredence Studio · Enterprise evaluation ·
Public sector / regulator · Pricing conversation · Partnership · Press / analyst · Other.

---

## 15. Resources (`/resources`)

### Whitepapers (5)
1. **The 5-Check Verification Model for Interoperable Trust** — specification of the policy-driven
   runtime engine; how a verifier policy selects and weights each check, short-circuit behaviour on
   required failure, and how one engine evaluates both W3C governance and SD-JWT VC business credentials
2. **Governance VCs: Bridging Policy and Cryptography** — governance credential types, DID-native
   framework identifiers, multi-authority models, and the **six-state governed lifecycle** with audit trail
3. **EU ESPR Compliance: A Trust Operating System Approach** — ESPR requirements mapped to platform
   capabilities; battery and textile passport mandates with UNTP alignment; tiered
   public/restricted/verified passport access; selectively disclosed conformity attestations
4. **Privacy-Preserving Supply-Chain Finance with Selective Disclosure** — issuer-attested threshold
   claims with SD-JWT selective disclosure, BBS+ unlinkable presentations, the Verifiable Receivable
   credential model, identity-anchored issuance, status-list revocation of settled receivables
5. **From AnonCreds to SD-JWT VC: Why PrimeCredence Moved to the EUDI Rail** — why the business rail
   moved from AnonCreds-over-DIDComm to SD-JWT VC over OID4VCI/OID4VP. Gained: wallet
   interoperability, JOSE end-to-end, Token Status List revocation. Traded: zero-knowledge range
   predicates (replaced by issuer-attested claims). BBS+ restores unlinkable presentations.
   *(→ "Start here" papers are #1 and #2.)*

### Case studies (4) — labelled **illustrative / modelled**, not customer references
| Case study | Headline modelled outcomes |
|---|---|
| **Tyre Manufacturer DPP** — privacy-preserving sell-through verification | Dealer participation modelled rising 40% → 95% (voluntary); verification from multi-week manual audit to real-time; zero trade-secret exposure; every attestation signed and status-checked |
| **Deep-Tier Supply-Chain Finance** — verifiable receivables | Financing extended to tier-2/tier-3 suppliers; onboarding modelled weeks → hours; settled receivables revoked on the status list so they can't be re-presented; modelled financing-cost reduction |
| **Peer-Learning Talent Ecosystem** — privacy-preserving skill credentials | Modelled thousands of credentials/month; 5-Check on every presentation; employers see only "qualifies: yes/no"; forgery caught by signature + issuer-authorisation checks |
| **Battery Passport (EU ESPR)** — UNTP-compliant DPP | Passport readiness modelled ahead of the Feb 2027 mandate; traceability events from every supplier tier; QR scan resolves public passport in browser (no wallet/app); only the attested compliance claim disclosed |

> **Important caveat:** every metric is explicitly **"modelled"** or **"illustrative"**. There are no
> named customers or audited results anywhere on the site.

### Concept library / glossary
~60 plain-language A–Z definitions at `/resources/glossary/<slug>`. Notable entries and the
PrimeCredence-specific vocabulary:

- **Network Anchor** — the organisation that creates and governs a trust network (manufacturer,
  brand, bank, agency)
- **Network Participant** — an organisation that joins a Network Anchor's network to transact
  (supplier, subcontractor, logistics provider)
- **Entity Wallet** — the organisational holder; a tenant receives partner-issued credentials into it
- **DIA (Digital Identity Anchor)** — authority-issued credential linking a DID to a verified legal
  identity under a governance framework
- **DTE / DFR / DCC / DPP / IDR** — the UNTP credential objects (Digital Traceability Event, Digital
  Facility Record, Digital Conformity Credential, Digital Product Passport, Identity Resolver)
- **Dual-Layer Architecture**, **Governance VC**, **5-Check Verification**, **Assurance Level**,
  **Verification Policy**, **Status Management**, **Interoperability Profile**, **Trust Registry**,
  **TRQP**, **BYOK**, **Multi-Tenancy**, **Key Binding**, **Predicate**, **Selective Disclosure**,
  **ZKP**, **SSI**

**Marked `retired`** (useful signal about their architectural history): **AnonCreds**, **DIDComm**,
**`did:peer`**, **Accumulator**, **ZKP-Enabled Signature**, **Dual-Format Architecture**. This
corroborates whitepaper #5 — they migrated off the Hyperledger AnonCreds/DIDComm stack onto the
EUDI (SD-JWT VC + OID4VCI/OID4VP) rail.

---

## 16. Site map (all routes)

```
/                                   Home
/platform                           Capabilities
/platform/wallet                    PrimeCredence Wallet
/platform/studio                    PrimeCredence Studio
/platform/deployment                Deployment & Sovereignty
/how-it-works                       ToIP layers, lifecycle, 5-Check
/how-it-works/trust-framework       Governance as verifiable credentials
/how-it-works/security              Security & Compliance
/how-it-works/standards             Standards & Interoperability
/solutions                          Overview
/solutions/supply-chain-finance
/solutions/banking
/solutions/digital-product-passports
/solutions/food-traceability
/solutions/talent-ecosystems
/solutions/government
/for/enterprises
/for/regulators
/for/developers
/resources
/resources/whitepapers              (5 papers)
/resources/case-studies             (4 illustrative studies)
/resources/glossary                 (~60 terms)
/pricing
/company
/contact
```

---

## 17. Assessment — signals and gaps

**Strengths of the positioning**
- The governance-vs-cryptography distinction is a genuinely sharp, defensible differentiator, and it
  is carried consistently through every page rather than being a homepage slogan.
- **Unusually honest claim discipline** for this category: explicit "no eIDAS certification",
  explicit "none of our suites are post-quantum", `shipped` vs `demoable` status column, case-study
  metrics labelled "modelled", and a dedicated "honesty box" separating FDA rule from product.
- Standards depth is real and specific (named proof suite `eddsa-jcs-2022`, DCQL, BLS12-381,
  ML-DSA/SLH-DSA transition plan) rather than logo-wall standards-washing.
- Clear anti-blockchain-hype stance: no tokenisation, ledger for anchors only, payloads off-ledger.
- Sovereignty boundary is argued from first principles (*why* the registry must be shared), which is
  the hard part of the sovereign-deployment pitch.

**Gaps / things to probe**
- **No named customers, no audited results, no logos.** Every case study is modelled.
- **No published pricing** at any granularity — no allowance units, no overage rates.
- **No certifications cited** — no SOC 2, ISO 27001, or any third-party audit mentioned anywhere,
  despite a page aimed at security reviewers.
- **No company substance disclosed** — no team, no founders, no locations, no funding, no founding date.
- **Developer docs are immature** — no hosted API reference yet; the npm README is canonical; SDK is
  at v1.0.0.
- **Maturity signals are thin** — "demoable" status on GS1, "illustrative" scenarios, and a demo
  portal behind sign-in suggest an early-stage platform with the architecture further along than the
  deployment base.
- The site itself is a client-rendered SPA with only `/` prerendered — every other route serves the
  homepage shell to crawlers, which likely hurts their SEO on the solution and standards pages.

---

### How these notes were compiled
`primecredence.com` is a client-rendered single-page app: every URL returns the same prerendered
homepage shell, so only `/` is readable by a plain fetch. The content above was extracted from the
site's JavaScript route bundles (`/assets/*.js`), which carry the full text of every page, plus
`sitemap.xml` for the route list. All quotes and figures are the site's own wording.
