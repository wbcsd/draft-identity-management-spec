# PACT Identity Management — Addendum Structure (Working Outline)

*Draft v0.2 — 18 June 2026. A structural sketch for the Identity Management addendum to the PACT Technical Specifications for PCF Data Exchange (V3). For brainstorming in this project; intended to later become a spec in `wbcsd/data-exchange-protocol`. **v0.2: design decisions resolved via review session — see §12.***

---

## How to read this outline

This is a **skeleton, not a draft spec.** Each section below lists: *Purpose* (why the section exists), *Content* (what goes in it), and where useful *Decisions* (open design choices to resolve before writing normative text). Decisions are the things worth arguing about in the Tech Working Group; everything else is largely mechanical once they're settled.

The addendum mirrors the parent spec's structure (Introduction → Scope → Terminology → Conformance → Data Model → HTTP REST API → Examples → Changelog) so it can slot in as a companion document. It anchors explicitly on **§5.3 "Out of scope"** of the V3 spec, which already names "authentication and access management" and "credentials management (generate / exchange / rotate / revoke credentials)" as out of scope — i.e. this addendum fills a gap the base spec itself declared.

Scope of the addendum (per the Tech Master Deck) is the two coupled topics, designed against one shared identity model:
- **A. Automated Credential Exchange** — make the trust/credential bootstrap automatic.
- **B. Node Discoverability** — let participants find each other, with visibility controls.

---

## 0. Front matter

*Purpose:* situate the addendum relative to the base spec.

*Content:* title ("Technical Specification for PACT Identity Management — Addendum to PCF Data Exchange V3"), version/status block, editors, relationship statement ("this document extends, and does not modify, [DATA-EXCHANGE-PROTOCOL] V3"), link to base spec §5.3.

---

## 1. Introduction

*Purpose:* explain the problem and the fit with the existing network.

*Content:*
- The scaling problem: manual, bilateral credential handshakes and manual discovery don't scale to a network of ~50 solutions / ~180 corporates / thousands of suppliers.
- What IM adds as a *foundational network service*: **Register, Find, Connect, Manage** (authentication-as-a-service).
- Explicit relationship to base spec: IM sits *around* the existing exchange (OAuth 2.0 client-credentials + `/3/` actions), automating the credential bootstrap that §5.3 leaves out of scope; it does **not** change the data model or the four core actions.

*Decisions:*
- **D1 — Single addendum vs. two.** One document covering both topics (recommended; the deck and the epics both stress a shared architecture) vs. separate Discovery and Credential-Exchange documents.

### 1.1 Status of this document
### 1.2 Scope
*Content:* in scope = identity model, registration, discovery, automated credential exchange, the supporting API additions. Out of scope = footprint calculation, the PCF data model itself, identity proofing of natural persons, payment/commercial terms.
### 1.3 Intended audience
*Content:* solution providers (node operators), implementers of discovery/credential endpoints, the PACT registry operator.
### 1.4 Relationship to the PACT Network Services platform
*Content:* how the normative spec relates to the reference implementation (Sandbox / Directory Service in `pact-directory`).

---

## 2. Terminology

*Purpose:* define the vocabulary precisely; reuse base-spec terms (`host system`, `data owner`, `data recipient`, `solution`, `solution provider`) and add IM-specific ones.

*Content (new terms to define):* Entity (legal organisation), Operator, Node, Connection, Credential, Organisational Identifier, Registry, Directory / Discovery Service, Discovery Endpoint, Visibility Level, Verifiable Credential, Root of Trust.

*Decisions:*
- **D2 — Node vs. host system.** Is a "node" exactly a base-spec `host system`, or a logical participant that may map to many host systems? (The Sandbox treats a node as a virtual host; the SP-node concept implies one operator fronting many entities.)

---

## 3. Identity Model (the shared core)

*Purpose:* the conceptual foundation both topics build on. This is the section most worth getting right first.

### 3.1 Actor types
*Content:* the four actor types from the deck, normatively defined:
- **Single-Party Entity** — org running its own node(s) as supplier/buyer.
- **Multi-Party Entity** — operator (typically a solution provider) running nodes on behalf of many entities under a sub-network.
- **Reference Entity (read-only)** — a node others pull reference/PCF data from (relates to the "Reference Data" roadmap item).
- **Router Entity** — a platform acting purely as a connector between nodes.

### 3.2 Entities, operators and nodes
*Content:* the relationship model — an *entity* (legal org) is represented by one or more *nodes*, which may be operated by the entity itself or by an *operator* on its behalf. Define cardinalities (entity 1—* nodes; operator 1—* entities).

### 3.3 Organisational identifiers
*Content:* how an entity is identified network-wide. Maps onto the base spec's existing `companyIds` (a set of URNs on every footprint). Proposes a standard, resolvable identifier namespace.

*Decisions:*
- **D3 — Identifier scheme.** Adopt **LEI** (ISO 17442) as the canonical identifier via a `urn:lei:...`-style namespace? Allow multiple (LEI, DUNS, existing free URNs) with LEI recommended? Stay scheme-agnostic and only require *a* resolvable URN? Trade-off: integrity/global-uniqueness vs. the "low-cost and fair" principle (LEIs cost money and not all suppliers have one).
- **D4 — Identifier ≠ verification.** Decide whether holding an identifier implies any verification, or whether verification is a separate, optional layer (see §4.2).

### 3.4 Organisational hierarchy
*Content:* how to represent parent/subsidiary and the case where related entities are sometimes mutual suppliers (the Mondelez / Cadbury / Milka problem from the CDP learnings). Distinct identities per operating entity, with optional parent links, rather than one identity per corporate group.

*Decisions:*
- **D5 — Hierarchy representation.** Flat identifiers with optional parent references, vs. a structured group/sub-entity model. How much hierarchy does discovery actually need?

---

## 4. Registration

*Purpose:* how an entity/node joins the network and becomes findable and connectable.

### 4.1 Registering an entity and its node(s)
*Content:* the registration request, required attributes (identifier, name, endpoint base URL, contact), and the resulting registry record.

### 4.2 Identity proofing and assurance
*Content:* what, if anything, is checked at onboarding ("know your business partner" — raised in the Siemens session). Define assurance levels (e.g. self-asserted vs. third-party-verified vs. cryptographically verifiable).

*Decisions:*
- **D6 — Proofing model.** Self-service registration (low friction) vs. vetted onboarding vs. tiered assurance. Who vouches? (PACT, a QVI, a solution provider?)
- **D7 — Conformance gate.** Must a registrant already be PACT-conformant (link to the Conformance Service) before becoming discoverable/connectable?

### 4.3 Verifiable credentials (optional path)
*Content:* an optional binding of the registry identity to a **verifiable credential** (e.g. GLEIF **vLEI**), giving cryptographic proof of identity and of the authority of the acting party. Aligns with the deck's "Verification … using verifiable credentials" roadmap item and the EUDI direction.

*Decisions:*
- **D8 — vLEI / VC adoption.** Native VC/vLEI support now, design-for-later, or out of scope for V1? Which trust framework (vLEI ecosystem vs. generic W3C VC/DID)?

---

## 5. Node Discoverability (Topic B)

*Purpose:* let a participant find the right counterparty's node, with the owner in control of visibility.

### 5.1 Discovery architecture
*Content:* the federated model — "start as a simple directory but build a federated approach in from the start." PACT maintains a **registry of discovery services** and routes queries to the appropriate solution-provider discovery endpoints — described in the deck as **"like DNS for the internet."** Define the central registry vs. the federated endpoints split.

*Decisions:*
- **D9 — Centralisation boundary.** What is central (a root registry / index) vs. federated (per-SP discovery endpoints)? How are discovery services themselves registered and trusted?
- **D10 — Technology stance.** Keep the spec implementation-agnostic while the reference implementation explores graph storage (Neo4j named as a candidate for traversing trust chains).

### 5.2 Discovery API
*Content:* the new endpoint(s). Per the deck, solution providers implementing discovery add only **1–2 extra API methods**. Likely: a *query* method (resolve a company identifier → node/endpoint) and a *list* method (an SP node returning the customer endpoints it fronts).

*Decisions:*
- **D11 — SP-node concept.** Standardise the "one SP node as umbrella for many customers, each with a discoverable opt-in flag" pattern, so providers needn't sync every customer into a central node database.

### 5.3 Visibility and privacy controls
*Content:* opt-in, granular visibility levels (e.g. invisible / discoverable-to-authenticated / public), set by the node owner. This is the agreed mitigation for the data-sovereignty risk.

*Decisions:*
- **D12 — Visibility model.** What levels exist, what's the default (recommend default-private/opt-in), and at what granularity (per node, per relationship, per attribute)?

### 5.4 Discovery flow
*Content:* the end-to-end flow from the deck: customer searches for a supplier → if not found, platform offers to invite them → if the supplier uses another solution provider, system checks whether that provider is PACT-conformant → platform resolves the supplier's API endpoint and enables the connection.

---

## 6. Automated Credential Exchange (Topic A)

*Purpose:* automate the trust/credential bootstrap that §5.3 of the base spec leaves out of scope, so two nodes can go from "discovered" to "exchanging" without manual handshaking.

### 6.1 Trust establishment
*Content:* how two nodes establish mutual trust once discovered — the connection request/approval handshake (mirrors the Sandbox "connection invite → establish connection" flow).

### 6.2 Automated provisioning of exchange credentials
*Content:* how the `client_id` / `client_secret` used by the base spec's §5.5 OAuth flow get generated and delivered to the counterparty automatically and securely, replacing manual configuration. This is the precise fill for §5.3(b)/(c).

*Decisions:*
- **D13 — Credential mechanism.** Continue with provisioned client-credentials (minimal change to §5.5), or move toward credential-less / VC-based or mutual-TLS trust? Keep backward compatibility with the existing OAuth flow as a baseline.

### 6.3 Connection lifecycle
*Content:* request, approve, **rotate**, and **revoke** — covering the full "credentials management" list named in §5.3(c). Define states and the events that drive them.

### 6.4 Relationship to base-spec authentication (§5.5)
*Content:* explicit mapping showing this layer *produces* the credentials that §5.5 then *consumes* — no change to the token flow itself.

---

## 7. Security Considerations

*Purpose:* required before release (the Credential-Exchange epic mandates a security review).

*Content:* threat model (impersonation, credential leakage/replay, rogue discovery services, enumeration of the directory), mitigations, HTTPS/transport requirements, credential rotation/expiry, audit logging (the base spec puts logging out of scope in §5.3(d) — decide if IM normatively requires it).

---

## 8. Privacy and Data Sovereignty

*Purpose:* address the top risk flagged for discoverability.

*Content:* what registry data is published, lawful-basis/consent considerations, the opt-in visibility model (cross-ref §5.3), minimisation (expose endpoints, not commercial relationships), and protection against competitive intelligence harvesting.

---

## 9. Conformance

*Purpose:* define what it means to conform to the addendum, layered on the base spec's conformance model (RFC 2119 keywords).

*Content:* conformance classes — e.g. a **Discoverable Node**, a **Discovery Service provider**, a **Credential-Exchange-capable Node** — so implementers can adopt incrementally rather than all-or-nothing.

*Decisions:*
- **D14 — Conformance granularity.** One IM conformance level, or separable classes per capability (recommended, given the two topics may ship on slightly different timelines)?

---

## 10. API Specification

*Purpose:* the normative interface additions, in the base spec's style (Action definitions + OpenAPI).

*Content:* new actions for discovery (query/list) and credential exchange (connect/approve/rotate/revoke), request/response schemas, error codes (extend the base spec's `Error` enum), and OpenAPI fragments. Keep additions minimal and under the existing `$base-url$/3/...` convention where they belong to a node, and define where registry-level calls live.

*Decisions:*
- **D15 — Versioning.** Does this ship as part of a V3 minor release, or as an independently versioned companion spec referenced from V3?

---

## 11. Examples (non-normative)

*Content:* worked end-to-end walkthroughs — (a) discover an unknown supplier and connect; (b) an SP node fronting two customers (e.g. the deck's Unilever/Diageo example) responding to a discovery query; (c) credential rotation; (d) revoking a connection.

---

## 12. Design Decisions — resolved

Resolved in a design review on 18 June 2026 (Gertjan + Claude), grounded in the GLEIF meeting notes (4 & 17 June 2026), the existing `companyIds` ADR, and the Tech Master Deck. These are the working positions to take into the Tech Working Group.

| # | Decision | Resolution |
|---|----------|------------|
| **Ambition** | Integrity bar for V1 | **(a) low-friction directory now → (c) tiered/verified soon.** Assurance is an *attribute* of an identity, never a precondition for having one. |
| **D1** | One addendum vs. two | **One addendum** covering both topics on one shared identity model. |
| **D2** | Node vs. host system | **Node** = addressable PACT participant endpoint in the directory, realised by a base-spec *host system*; an **operator** may run many nodes; an **SP-node** also exposes discovery for many fronted entities. "Verified" is an attribute of the entity, not a node type. |
| **D3** | Identifier scheme | **LEI is the recommended canonical identifier, carried inside the existing `companyIds` URN array** (which stays open/multi-value, DUNS/DID allowed). No new PACT-specific identifier. LEI *recommended, not required* in V1 (coverage ~40–60% of non-financial corporates; cost being negotiated down with GLEIF). |
| **D4** | Identifier ≠ verification | Holding an identifier implies **no** verification; verification is the separate, optional tier (D8). |
| **D5** | Org hierarchy | **(b) per operating-entity identity; hierarchy referenced, not owned** — parent/ultimate-parent resolved from **LEI Level-2 data**. **No hierarchy links until an entity has an LEI.** Mutual supplier/buyer relationships are a property of the connection, not of identity. |
| **D6/D7** | Proofing & conformance gate | **Registration ≠ conformance gate.** Conformance is a visible **attribute, exposed as conformant-version(s)**. Exchange needs a conformant host on one side; demo node is the no-tooling on-ramp. Proofing **inherited from LEI/vLEI KYB**, never built by PACT; manual staff KYB is an interim stopgap. |
| **D8** | vLEI / VC adoption | **vLEI is in-scope-optional in the V1 document** as the verified tier; not mandated. |
| **D9** | Centralisation boundary | **(c) hybrid root** — thin central index keyed by identifier (LEI); each record either carries the endpoint directly (self-hosted) or delegates to an SP discovery endpoint (umbrella). PACT never replicates LEI reference data. **Regional roots allowed; every root can be tied to a parent** (DNS-like root hierarchy). |
| **D10** | Discovery technology | Spec stays **implementation-agnostic** (reference implementation may use a graph DB, e.g. Neo4j). |
| **D11** | SP-node concept | **Standardised** — one SP-node fronts many customers via a per-customer discoverable opt-in; no central sync of customer databases. |
| **D12** | Visibility model | **Two orthogonal controls.** *Resolvability* (know-the-LEI → resolve endpoint + request connection): **default ON for authenticated members**, off for anonymous. *Listability* (appear in search/browse): **default OFF, opt-in**, tiers members-only vs. public. **Connection always requires owner approval** — visibility ≠ access. |
| **D13** | Credential mechanism / auth | **(b) vLEI for identity/trust establishment, OAuth 2.0 for runtime API auth** (§5.5 unchanged). Credential exchange is **peer-to-peer after discovery** (PACT never in the credential/data path). Abstract connection/credential lifecycle is normative; **RFC 7591 (OAuth 2.0 Dynamic Client Registration) is the RECOMMENDED binding**; vLEI-native / mTLS bindings left open. **Rotate** and **revoke** defined. Approval manual in V1, policy-based auto-approval later. |
| **vLEI granularity** | Verified-tier credential scope | **(a) entity-level for V1**; **(b) entity + delegated role** as the named extension (for SP-node delegated authority); **person-level out of scope** (no natural-person data in PCF exchange). |
| **D14** | Conformance granularity | **(a) separable capability classes** — *Discoverable Node*, *Discovery Service Provider*, *Credential-Exchange-capable Node* — with verification as a cross-cutting attribute. |
| **D15** | Versioning | Companion document **referenced from V3** (provisional; revisit at §10). |

**Still open / deferred:** policy-based auto-approval rules (post-V1); the "invite a non-member supplier to join" flow (treat as informative/product, not normative); precise schema of the discovery API methods; whether IM normatively *requires* audit logging (§5.3 leaves logging out of scope). 

### One-line architecture summary

A federated, DNS-like directory of **LEI-identified** entities with hierarchical (global/regional/SP) roots; discovery is opt-in and privacy-controlled; once two nodes find each other they establish trust **peer-to-peer**, verifying identity via **vLEI** (optional) and auto-provisioning the existing **OAuth** exchange credentials via **RFC 7591** — leaving the base-spec data model and token flow untouched.

---

## 13. Changelog & References

*Content:* changelog; normative references (base spec [DATA-EXCHANGE-PROTOCOL] V3, RFC 6749, RFC 6750, OpenID Connect Discovery, ISO 17442/LEI, W3C VC/DID, vLEI Ecosystem Governance Framework); informative references (EUDI ARF, PACT IM Vision Paper).

---

## Suggested drafting order

1. **§3 Identity Model** (and resolve D3, D5) — the shared foundation.
2. **§6 Automated Credential Exchange** — fills the clearest, best-defined §5.3 gap.
3. **§5 Node Discoverability** — depends on the identity model and the SP-node concept.
4. **§4 Registration**, then **§7–§8 Security/Privacy**, **§9 Conformance**, **§10 API**, **§11 Examples**.
