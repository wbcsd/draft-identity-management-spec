# PACT Identity Management — Orientation Note

*Working background document for drafting the PACT Identity Management technical specification.*

---

## 1. Purpose of this note

This note pulls together the context needed to start drafting a *PACT Identity Management* (IM) specification — framed by PACT as an **addendum to the existing PACT Technical Specifications V3**. It synthesises four sources: the existing protocol in GitHub (`wbcsd/data-exchange-protocol` and `wbcsd/pact-directory`), the PACT Notion workspace (history, working-group notes, current epics), the **PACT Tech Master Deck** (dated 2 June 2026), and a first scan of relevant third-party identity standards (notably GLEIF's LEI/vLEI).

It is deliberately written as orientation, not specification: the goal is a shared, accurate picture of what exists, what has been tried, what is being decided right now, and which external standards are worth borrowing from — so that the brainstorming and drafting that follow start from solid ground.

---

## 2. What exists today: the PACT Technical Specifications

The PACT Network (formerly the Pathfinder Network) is an open data exchange for product carbon footprint (PCF) data, developed by the PACT initiative within WBCSD. The technical specifications define two things: a **PCF data model** and a **PCF exchange API** (a REST API). They live at `github.com/wbcsd/data-exchange-protocol`, are authored in Bikeshed, and are published at `docs.carbon-transparency.org/tr/data-exchange-protocol/latest/`. The current published version is **3.0.3 (18 November 2025)**, editor Gertjan Schuurmans (WBCSD); v2.3.0 and v1.0.1 remain documented for older implementers. The published spec states plainly that *"the scope of the HTTP API is minimal by design. Additional features will be added in future versions"* — which is exactly the hook Identity Management extends.

PACT itself describes two pillars: the **Methodology** (how to calculate a PCF) and the **Data Exchange Protocol** (how to exchange one). The Tech Master Deck states the protocol uses "standard OAuth authentication" and explicitly notes "support for future PACT Directory Service" — confirming that discovery/identity is a planned addition, not part of today's exchange.

For scale context (Tech Master Deck, June 2026): PACT was founded in 2020 within WBCSD; the Methodology is on its third version (V1 2021, V2 2023, V3 2025). The network now has roughly **50 PACT-conformant software solutions** (including SAP, Siemens, Microsoft), about **180 corporates** and **5,000+ suppliers** exchanging data, and **4M+ PCFs** calculated in 2025 — described as the largest PCF-exchange network in the world, yet still "a drop in the ocean." National networks in Singapore and China connect through PACT, and GS1 has adopted PACT for Digital Product Passports. This scale is the reason manual, bilateral trust no longer suffices and IM is now a priority.

What matters for Identity Management is **how the current protocol handles identity, authentication, and discovery** — because that is precisely where IM has to fit:

- **A conforming host system MUST implement exactly four actions** (§5.2): **Authenticate**, **ListFootprints**, **GetFootprint**, and **Events** (the last for asynchronous request/response). Any host can act as both *data owner* and *data recipient*, mirroring real supply chains — i.e. a genuinely peer-to-peer exchange.
- **Authentication** (§5.5) uses the **OAuth 2.0 client-credentials flow** (RFC 6749 §4.4), over HTTPS only, with **Bearer** tokens (RFC 6750). The client discovers the token endpoint dynamically via **OpenID Connect Discovery** at `$auth-base-url$/.well-known/openid-configuration`, falling back to `$auth-base-url$/auth/token` if none is published; it then POSTs `grant_type=client_credentials` with `client_id`/`client_secret` to obtain a (SHOULD-expire) access token. The data actions live under `$base-url$/3/...`. Critically, the flow assumes the client *already holds* a `client_id`/`client_secret` — the spec says nothing about how those credentials were provisioned or trusted in the first place.
- **Credential and trust bootstrap is explicitly OUT of scope.** §5.3 lists, as functionality beyond this document, "authentication and access management" (deciding which recipient may access which PCF) and "credentials management — the overall functionality to generate access credentials for data recipients, to exchange these credentials with data recipients, to rotate or revoke such credentials." That declared gap is **precisely** what IM's Automated Credential Exchange fills, and the addendum can reference §5.3 directly.
- **Trust between two parties is established bilaterally and largely manually.** Because the above is out of scope, in practice someone exchanges and configures the `client_id`/`client_secret` by hand — a manual "credential handshake." This works at pilot scale but becomes a bottleneck as the network grows.
- **Identity today is self-asserted, not registered or verified.** In the data model, a footprint's owner is identified by `companyIds` — a free set of URNs chosen by the data owner (e.g. `urn:company:example:company1`); products use `productIds` URNs (e.g. GTIN). There is no central registry and no cryptographic proof that a `companyId` belongs to the entity claiming it. This is a direct hook for IM: a standard, verifiable organisational identifier (e.g. an LEI-based URN) could slot into the existing `companyIds` scheme.
- **There is no native discovery.** The protocol assumes you already know *who* your counterparty is and *where* their endpoint lives. There is no built-in way to look up "which node represents supplier X" — the gap the Node Discoverability work fills.

These three points are the gap Identity Management is meant to close: a way to **find** the right party, **identify** them with integrity, and **establish trust** with them at scale, layered on top of (not replacing) the existing OAuth2 + REST exchange.

### The PACT Network Services platform (`wbcsd/pact-directory`)

A second repository, `wbcsd/pact-directory`, now hosts the **PACT Network Services platform** (`services.carbon-transparency.org`) — a hub of services on a shared portal + single backend API. It currently offers:

- **Conformance Service** — runs the conformance test suite against a provider's implementation so they can become PACT-conformant (a prerequisite for joining the network). Integrated via a proxy to the external `pact-conformance-test-service`.
- **Data Exchange Sandbox** — lets an organisation spin up virtual PACT-conformant **nodes**, create/import/publish footprints, establish **connections** with other nodes (internal or external), send/fulfil **PCF requests**, and inspect activity logs. Each internal node exposes a full PACT v3 API surface (OAuth2 client-credentials, `/3/footprints`, `/3/events`), so internal and external nodes are interchangeable to a client.

Crucially, this repo **grew directly out of the Identity Management initiative**, and two future services are explicitly anticipated:

- a **Directory Service** — "for discovering and identifying parties on the PACT Network and managing trust between them"; and
- a **Data Model Extension registry**.

So IM is not a green-field addition: there is an MVP lineage, a live platform to host it, and "connection management" primitives already present in the Sandbox that IM should build on.

---

## 3. What "PACT Identity Management" is

From the PACT Notion (Implementation Hub → Solution Providers → Identity Management) the service is framed as a **foundational service of the PACT Network** that "enables organizations to securely and efficiently identify and establish trust connections with each other at scale, facilitating PCF exchange between solutions."

In its initial form it is intended to provide organisations with three capabilities:

1. **Register** to the PACT Network.
2. **Find** others on the PACT Network.
3. **Connect** — establish trust connections with them.
4. **Manage credentials** of customers and their suppliers — i.e. *authentication as a service*.

(The Tech Master Deck frames these as Register / Find / Connect / Manage.) The intended outcome is "a robust, high-integrity registry of organizations on the PACT Network, ready to connect and exchange PCF information." The early technical design was captured as an *authentication-as-a-service* design in the `pact-directory` repo.

### Types of actors (Tech Master Deck)

The deck introduces a useful actor model that the identity design must accommodate — entities relate to nodes in more than one way:

- **Single-Party Entity** — an organisation running one or more of its own nodes, acting as supplier or buyer in a value chain (e.g. a buyer org hosting multiple nodes).
- **Multi-Party Entity** — an organisation that creates nodes *on behalf of other organisations*, under its own org profile as a sub-network (e.g. "Solution Provider X creates separate nodes for Unilever and Diageo"). This is the key Solution-Provider case.
- **Reference Entity (read-only)** — an organisation exposing the exchange as a data repository: a reference node from which any org in the network can *pull* PCF data.
- **Router Entity** — a platform that built the exchange and acts purely as a connector between nodes (e.g. a pure data-exchange platform, often as a lead-gen / adoption driver).

This directly informs the identity model: an *entity* (legal org), the *operator* acting on its behalf (possibly a solution provider), and the *node(s)* are distinct objects, and a single operator may represent many entities.

---

## 4. History and lineage

| When | Milestone |
|------|-----------|
| Sep 2024 | Identity Management working group formed to co-create the service |
| Oct–Dec 2024 | IM **MVP** developed |
| Jan 2025 | **Alpha** MVP testing phase launched |
| Mar 2025 | **Beta** MVP testing phase launched; **PACT Identity Management Vision Paper** published |
| Mar–Apr 2025 | MVP testing target (organisations asked to complete testing by 30 Apr 2025) |
| Oct 2025 | MVP learnings fed into the broader **PACT Network Services platform**; Conformance Service shipped first |
| Jan 2026 | Data Exchange Sandbox preview / evaluation |
| Apr 2026 | Data Exchange Sandbox released; Directory / IM service positioned as a planned future addition |

The throughline: an initial IM MVP proved out concepts (registration, finding parties, authentication-as-a-service), then was deliberately **paused** to let other foundational services mature and to gather pilot learnings. V1 of Identity Management is now being planned on the back of those learnings — which is what this specification work serves.

**Design principles** established for the service (from the Notion "Identity Management" page): (1) easy & simple to use, (2) secure, (3) standards-based, (4) global, low-cost and fair, (5) open and federated, (6) iterative.

---

## 5. Where it stands now (mid-2026)

Two committed epics under "Product and Technology" are driving the current architecture work, both targeting **release by end of September 2026**:

- **Identity Management: Node Discoverability** — make nodes discoverable by other participants, with **configurable, opt-in visibility levels** set by the node owner; a structured, searchable directory of participating nodes to replace manual/bilateral discovery. A graph database (Neo4j is the named candidate) is being considered for traversing network relationships and trust chains.
- **Identity Management: Automated Credential Exchange** — automate the credential handshake between nodes, removing manual intervention from trust establishment and onboarding.

The two epics are **deliberately coupled**: the decision log (12 Mar 2026) records that they share an identity architecture and should be designed together. The relevant milestone for both — *"Identity management architecture finalized (covering both Node Discoverability and Credential Exchange)"* — is scheduled for **20 May → 15 June 2026**. In other words, the architecture is being locked down right now, which is exactly the window this specification draft needs to inform.

Shared dependencies and risks already flagged: both require **Sandbox v1.0**; key risks are privacy / data-sovereignty concerns around publishing node information (mitigation: opt-in with granular visibility), security complexity of automated credential exchange (mitigation: security-first design + a dedicated security review before release), and interoperability across different node implementations.

PACT is also running **feedback sessions with implementers** (e.g. Ecochain, CO2 AI, SAP, Siemens) to pressure-test the IM direction — a useful source of requirements as drafting proceeds.

### Delivery plan (Tech Master Deck "Next Steps")

The deck sets out a delivery plan built around an **addendum to Tech Spec V3** and a reopened working group:

| When | Step |
|------|------|
| Jun/Jul 2026 | PACT team creates the **initial draft addendum** to the Technical Specifications V3 |
| Aug 2026 | **Restart the Technology Working Group** (open to all organisations) to co-develop the draft |
| Oct 2026 | Open call for **initial testers** of the draft addendum |
| Feb 2027 | **Final release** of the addendum |

> **Timeline note / to reconcile:** the Notion epics (Node Discoverability, Automated Credential Exchange) carry an internal target of *end-September 2026*, while the more recent Tech Master Deck (2 June 2026) shows a public path ending in *final release Feb 2027*. These likely reflect "build/feature-complete in the Sandbox" vs. "ratified spec addendum" — worth confirming which dates govern the specification work.

### Broader tech roadmap (2026-H2 → 2027)

IM sits within a wider roadmap the deck lays out, several items of which touch identity:

- **Identity Management — Discoverability**: a directory service for discovering nodes across sub-networks.
- **Identity Management — Credential Exchange**: secure, automated connection management, delivered as the addendum to the data-exchange spec.
- **Data Model Extensions (DMX) Registry**: a central directory for discovering and reusing data-model extensions.
- **Reference Data**: letting emission-factor and reference-data providers join the network alongside companies (relates to the Reference Entity actor type).
- **Verification**: aligned with initiatives like the Digital Product Passport, **using verifiable credentials** — a direct signal that the VC/vLEI direction is on PACT's radar.

---

## 6. Key open questions and tensions

These recur across the Notion material and should be treated as first-class design questions in the spec:

- **Organisational hierarchy and identity granularity.** Learnings from CDP (captured in the Vision Paper review) highlight that corporate hierarchies are messy: e.g. Mondelez owns both Cadbury and Milka; sometimes one is a supplier to the other, for different products. Do such entities share one identity or hold distinct identities? Where does PCF responsibility sit — at the global/parent entity or the operating entity? The identity model must represent legal entities, operating units, and the node(s) that act on their behalf without forcing an unrealistic 1:1 mapping.
- **Federated vs. central.** The design principles call for "open and federated," yet a "high-integrity registry" implies some central or root authority. The spec needs a clear stance on what is central (e.g. a root of trust / directory) versus what is held and managed by participants.
- **Discoverability vs. privacy / data sovereignty.** Publishing who is on the network is valuable for matchmaking but sensitive commercially. Opt-in, granular visibility is the agreed mitigation; the spec should define the visibility model precisely.
- **Identity proofing / onboarding integrity.** From the Siemens session: who vouches that a member is who they claim to be? What checks happen at onboarding ("know your business partner")? This is where third-party identity standards become relevant.
- **Interoperability with heterogeneous node implementations.** Whatever credential and discovery mechanisms are chosen must work across independently built PACT-conformant solutions.

---

## 7. Third-party standards landscape

The "standards-based" principle and the implementer feedback (Siemens in particular) point to several external frameworks worth evaluating as building blocks. The most directly relevant is **GLEIF**.

### GLEIF — LEI and vLEI

- **LEI (Legal Entity Identifier)** — a 20-character ISO 17442 code uniquely identifying a legal entity globally, backed by the Global LEI Index. It is a strong candidate as the **canonical organisational identifier** in the PACT registry (stable, global, already adopted in finance/regulatory contexts), directly addressing the "which entity is this really?" problem.
- **vLEI (verifiable LEI)** — the cryptographic, verifiable form of the LEI. It is a decentralised digital identity that lets counterparties **computationally verify** an organisation's identity *and* the authority/role of people acting on its behalf. vLEI credentials are issued by **Qualified vLEI Issuers (QVIs)** under GLEIF governance, with **GLEIF as the root of trust**: every vLEI is traceable through a cryptographically protected chain back to the entity's LEI record. vLEIs are designed to live in identity wallets and verifiable-credential registries, and to integrate with broader ecosystems including the **EU Digital Identity (EUDI) Wallet**.

Worth noting: **GLEIF already appears among PACT's stakeholders** (it features in the standard-setters/data-providers logos in the Tech Master Deck), so there is an existing relationship to build on. PACT's own roadmap also names **verifiable credentials** explicitly (under Verification, aligned with the Digital Product Passport), which points in the same direction as vLEI.

Why this matters for PACT: vLEI offers a ready-made model for exactly the two problems IM is solving — **high-integrity organisational identity** (the registry) and **verifiable, automated trust establishment** (credential exchange), with an existing governance framework and root of trust rather than PACT having to invent one. The trade-offs to weigh: cost and effort of obtaining LEIs/vLEIs (vs. the "low-cost and fair" principle), maturity and adoption of the QVI ecosystem, and the underlying KERI/ACDC technology stack vs. more mainstream W3C Verifiable Credentials / DIDs.

### Other reference points (from Notion / implementer sessions)

- **EUDI Wallet Architecture and Reference Framework (ARF)** — referenced by Siemens; relevant for wallet-based credential presentation and for terminology alignment. vLEI explicitly targets integration with EUDI.
- **W3C Verifiable Credentials (VC) and Decentralized Identifiers (DID)** — the general standards vocabulary for issuing, holding, and verifying credentials in a federated way; useful as the abstract model even if vLEI is chosen as the concrete instantiation.
- **Dun & Bradstreet (DUNS)** — alternative/complementary business-identity reference ("know your business partner").
- **Industrial data-space initiatives** (e.g. Estainium, IETA context, trust service providers) — raised by Siemens as adjacent ecosystems and possible models for neutral identity verification.

A useful framing for the spec: treat **identifier** (who the entity is — likely LEI), **credential** (verifiable proof of identity/role — candidate vLEI / VC), and **discovery** (how to find the entity's node — the directory) as three separable layers, each of which can reference an external standard.

---

## 8. Implications for the specification

A first cut of how IM layers onto the existing protocol, to be refined during drafting:

- **It is an addendum, not a replacement.** PACT's framing is explicit: IM will be codified as an **addendum to Technical Specifications V3**, embedded in the Sandbox. The existing PCF data model and the OAuth 2.0 client-credentials + `/3/...` actions (Authenticate, ListFootprints, GetFootprint, Events) stay intact. IM sits *in front of* and *around* that exchange: it tells a party who to talk to and automates the credential bootstrap that §5.3 of the current spec deliberately leaves out of scope — rather than changing the exchange itself.

- **Federated, DNS-like discovery (Tech Master Deck design sketch).** The deck's working design is to "start as a simple directory but build a federated approach in from the start." PACT would maintain a **registry of discovery services** and route discovery queries to the appropriate solution-provider APIs — "similar to DNS for the internet." Solution providers implementing the discovery service would add only **1–2 extra API methods**. A concrete discovery flow: a customer searches for a supplier; if not found, the platform offers to invite them; if the supplier already uses a different solution provider, the system checks whether that provider is PACT-conformant; the platform then discovers the supplier's API endpoint and enables the connection.

- **"Solution-provider node" concept.** Rather than creating an individual node per customer, a solution provider could run one **SP node** acting as an umbrella for many customers: customers opt in via a "discoverable on PACT network" checkbox, and the SP exposes an endpoint that returns customer endpoints in response to company-identifier queries. This avoids synchronising every customer into PACT's node database and maps directly onto the Multi-Party Entity actor type.
- **Three conceptual components** map cleanly onto the existing work: a **Directory / discovery** service (the Node Discoverability epic), a **credential / trust establishment** mechanism (the Automated Credential Exchange epic), and an **identity model** (identifiers + verifiable credentials) underpinning both — the shared architecture the epics insist on designing jointly.
- **Anchor on a standard identifier.** Adopting LEI as the entity identifier (and evaluating vLEI for verifiable credentials) would satisfy the "standards-based" principle and give the registry its "high integrity."
- **Design for the hierarchy problem from day one** — entities, sub-entities, and nodes as distinct but linked objects.
- **Make discovery opt-in with a defined visibility model**, and put automated credential exchange through a security review before release (both already mandated by the epics).

---

## 9. Key references

- **Existing specs (published):** `docs.carbon-transparency.org/tr/data-exchange-protocol/latest/` — current version **3.0.3** (18 Nov 2025). Source and OpenAPI on GitHub: `github.com/wbcsd/data-exchange-protocol` (`spec/v3/`).
- **PACT Network Services platform (GitHub):** `github.com/wbcsd/pact-directory` — Conformance Service, Data Exchange Sandbox; design notes and IM integration guide in `docs/`. Live: `services.carbon-transparency.org`.
- **PACT Identity Management Vision Paper:** `carbon-transparency.org/resources/pact-identity-management-vision-paper`.
- **GLEIF LEI/vLEI:** `gleif.org/en/organizational-identity/lei-vlei/the-verifiable-lei-vlei`.
- **EUDI Wallet ARF:** `eu-digital-identity-wallet.github.io/eudi-doc-architecture-and-reference-framework`.
- **PACT Notion:** Identity Management page; Node Discoverability and Automated Credential Exchange epics; Vision Paper review; Siemens / SAP / Ecochain / CO2 AI feedback sessions.

---

## 10. Open items for this note

- **Tech Master Deck** — read and reconciled into sections 1, 2, 3, 5, 7 and 8 (deck dated 2 June 2026). The deck and this note are well aligned; the main thing to confirm is the **timeline discrepancy** flagged in section 5 (Notion epics' end-Sept-2026 target vs. the deck's Feb-2027 final release).
- **v3 spec detail** — §5.2 (Host System), §5.3 (Out of Scope) and §5.5 (Authentication Flow) have now been read in full from the spec source and incorporated (see section 2). Still to review when drafting: §5.6–§5.8 action details (ListFootprints/GetFootprint/Events), §6 Exchanging Footprints, and §8 Product Identification (URN schemes — relevant to how an identity identifier would be encoded).
- **MVP design doc** — `authentication-as-a-service-design` and the integration guide in `pact-directory/docs/` still to be read in full for concrete MVP lessons.
- **MVP design doc** — the `authentication-as-a-service-design` and integration guide in `pact-directory/docs/` should be read in full to capture concrete lessons from the MVP.
