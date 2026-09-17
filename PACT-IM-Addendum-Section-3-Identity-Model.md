# PACT Identity Management Addendum — §3 Identity Model (Draft)

*Draft v0.2 — 16 September 2026. Draft normative text for the Identity Model section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Reflects the design decisions resolved on 18 June 2026 as amended by the Technology Working Group session of 2 September 2026 (see the addendum outline, §12.1). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review. Open items are called out inline as *Editor's notes* and consolidated in the addendum outline §12. Where this section defines a URN namespace or attribute name, the exact string is provisional pending WG ratification.
>
> **Changes in v0.2 (following the 2 September WG session).** The LEI is no longer the primary or canonical identifier; it is one optional layer (W3). The assurance ladder is extended with *Email-verified* and *Domain-verified* levels (W4). Assurance for buyers and suppliers is attested by the Operator of their Discovery Service (§3.5). A new §3.7 sets the identity-verification options side by side.

---

## 3. Identity Model

### 3.1 Introduction

This section defines the identity model that underpins both Node Discoverability (§5) and Automated Credential Exchange (§6). It introduces no change to the [=data model=] or the four core actions of the base specification; it defines the *participants* of the PACT Network, *how they are identified*, and *how strongly that identity is assured*, so that those participants can be discovered and can establish trust.

The model is deliberately layered so that a participant can take part with minimal friction (a self-asserted identifier, or a verified email address) and later strengthen its identity (a verified domain, an LEI, a verifiable credential) without changing how it is represented. **Assurance is an attribute of an identity, not a precondition for holding one.**

This addendum reuses the base-specification terms [=host system=], [=data owner=], [=data recipient=], [=solution=], and [=solution provider=] without modification, and adds the terms defined below.

### 3.2 Entities, Operators, Nodes, and Discovery Services

The identity model distinguishes four concepts:

- **Entity** — a legal organisation that owns or is responsible for PCF data (a [=data owner=] and/or [=data recipient=] in the sense of the base specification). An Entity is the subject of an organisational identity (§3.4).
- **Operator** — the party that technically operates one or more [=Nodes=]. An Operator MAY be the Entity itself, or a [=solution provider=] acting on the Entity's behalf.
- **Node** — an addressable PACT Network participant endpoint, realised by a base-specification [=host system=] that implements the four core actions.
- **Discovery Service** — an endpoint, run by an Operator, that answers discovery queries (§5) for the Entities whose Nodes that Operator runs. Discovery Services, not Entities or Nodes, are what the central **PACT Directory** lists (§4).

The following relationships hold:

1. An Entity MUST be represented on the network by at least one Node.
2. An Entity MAY be represented by more than one Node (for example, distinct business units or regions), and MAY be served by more than one Operator.
3. An Operator MAY operate Nodes on behalf of many Entities.
4. A Node MUST be associated with exactly one Entity as the [=data owner=] whose footprints it serves; this does not restrict the Node from also acting as a [=data recipient=].
5. An Operator that wishes the Entities it serves to be discoverable MUST run a Discovery Service. An Entity that operates its own Node(s) and wishes to be discoverable runs its own Discovery Service, answering only for itself (the *SP-of-one* case). No separate mechanism exists for self-hosted Entities.

*Editor's note.* The Node↔host-system mapping is intentionally 1:1 at the protocol level: a Node *is* a conformant host system that is reachable through a Discovery Service. The Operator and Discovery Service concepts express the common case where one [=solution provider=] fronts many Entities, and keep all Entity data with that provider rather than in a central index (W1).

### 3.3 Actor types

A participant relates to the network in one of the following ways. A conforming implementation SHOULD classify each Node as exactly one actor type; the type informs discovery (§5) and visibility behaviour.

- **Single-Party Entity** — an Entity operating its own Node(s), acting as [=data owner=] and/or [=data recipient=] in a value chain. Where it is discoverable, it runs its own Discovery Service (§3.2, requirement 5). *Example: a buyer organisation hosting its own Node.*
- **Multi-Party Entity** — an Operator (typically a [=solution provider=]) operating Nodes on behalf of multiple Entities, answering discovery queries for them through a single Discovery Service. *Example: "Solution Provider X" operating distinct Nodes for two of its customers.*
- **Reference Entity** — an Entity exposing a Node from which other participants MAY retrieve reference or PCF data on a read-only basis (for example, an emission-factor or reference-data provider). A Reference Entity's Node MUST implement the retrieval actions but MAY decline connection requests that imply write or event obligations.
- **Router Entity** — a party operating a Node purely as a connector between other Nodes, which is neither the [=data owner=] nor the [=data recipient=] of the footprints it conveys. A Router Entity MUST NOT present itself as the [=data owner=] of footprints it merely routes.

### 3.4 Organisational identifiers

Every Entity MUST be identified by at least one organisational identifier expressed as a Uniform Resource Name [RFC8141], carried in the base-specification `companyIds` array. This addendum does not introduce a new identifier attribute, and it does not designate a single canonical identifier scheme. Discovery queries are keyed on `companyIds` values (§5).

1. An Entity's `companyIds` MUST contain at least one identifier and MAY contain several (for example, a self-issued identifier together with a registry-issued one).
2. An Entity SHOULD include identifiers issued by an authoritative registry where it holds them. Where the Entity holds a **Legal Entity Identifier (LEI)** [ISO17442], it SHOULD include it: the LEI is globally unique, openly resolvable, and is the identifier that enables the *Registry-verified* assurance level (§3.5) and hierarchy (§3.6).
3. An Entity that does not hold an LEI MAY participate using any other URN identifier (for example a DUNS-based or self-issued URN, or a Decentralized Identifier). Holding an LEI MUST NOT be a condition of participation.
4. Implementations MUST treat `companyIds` as a set: two records sharing at least one identical identifier value refer to the same Entity. Implementations MUST NOT assume global uniqueness of self-issued identifiers.

The RECOMMENDED URN form for an LEI is:

```
urn:lei:<20-character-LEI>
```

*Example `companyIds` for an LEI-holding Entity that also exposes a self-issued identifier:*

```json
"companyIds": [
  "urn:lei:5493001KJTIIGC8Y1R12",
  "urn:pfi:www.example.com:org-id:401765"
]
```

*Editor's note.* The exact LEI URN namespace (`urn:lei:…` vs. an LEI sub-type within the existing `urn:pfi:…` convention) is to be ratified by the WG. Two related questions are open: whether a verified Internet domain should itself be expressible as a `companyIds` value (for example a `did:web` identifier), and whether a VAT or Chamber-of-Commerce number can serve as a PACT identifier (parked item P2, §12.1).

### 3.5 Identity assurance

An organisational identity carries an **assurance level**, which is an attribute of the identity and does not affect how the Entity is represented (§3.4) or whether it may take part (§4).

#### 3.5.1 Assurance levels

This addendum defines the following assurance levels, in ascending order:

| Level | What has been established | Evidence |
|---|---|---|
| **Self-asserted** | Nothing beyond the Entity's (or its Operator's) own declaration. The default; no onboarding friction. | None. |
| **Email-verified** | A representative of the Entity controls a mailbox at an Internet domain the Entity claims as its own. | Completed email challenge (§3.7.1). |
| **Domain-verified** | The Entity controls an Internet domain. Includes the evidence of *Email-verified* for that domain, or supersedes it. | Completed domain challenge (§3.7.2). |
| **Registry-verified** | The Entity is *Domain-verified* **and** holds an LEI whose registration status is `ISSUED`; its legal existence has been validated by an LEI issuer as part of issuance. | Domain challenge plus LEI status check (§3.7.3). |
| **Credential-verified** | The Entity, or a delegated role acting for it, has presented a verifiable credential that cryptographically proves the organisational identity. | Verified credential presentation (§3.7.4). |

1. The levels from *Email-verified* to *Registry-verified* are cumulative: each includes the evidence of the level below it. *Credential-verified* MAY be reached directly, since the credential itself binds the holder to the identifier.
2. An assurance level applies to an Entity identity as a whole, but the evidence supporting it MUST be recorded per method, with the time it was last confirmed (§3.5.2).
3. An email address at a domain operated by a public mailbox provider MUST NOT support *Email-verified*.

*Editor's note.* *Registry-verified* is defined to require *Domain-verified* because an LEI status check on its own shows only that the identifier is valid, not that the party presenting it is that Entity. Requiring a verified domain as well narrows that gap without closing it: an LEI record carries no domain, so the link between the verified domain and the LEI still rests on the attesting Operator's judgement. Only *Credential-verified* removes this gap cryptographically. The WG should confirm this position.

#### 3.5.2 Who attests

The PACT Network does not verify buyer or supplier identities itself.

1. For *Email-verified*, *Domain-verified*, and *Registry-verified*, the assurance level is **attested by the Operator of the Discovery Service** through which the Entity is discoverable. That Operator performs the check when it onboards the Entity. How it does so is not specified; what it publishes is (§5.7).
2. A Discovery Service MUST NOT publish an assurance level for which it has not itself performed the check, or verified a credential.
3. Every published assurance level MUST identify the attesting Discovery Service and carry, per method, the time the evidence was last confirmed.
4. For *Credential-verified*, a counterparty MAY verify the credential itself instead of relying on the Operator's attestation (§3.7.4, §6).
5. A counterparty relies on an Operator's attestation to the extent it trusts that Operator. The Operator's own identity is established by domain ownership as part of the conformance-gated listing in the PACT Directory (§4.3).

#### 3.5.3 Assurance and connection policy

1. The network MUST NOT require any assurance level above *Self-asserted* as a condition of being listed, discovered, or sending a connection request.
2. Each party decides, through its own connection policy (§6), which minimum assurance level it accepts from a counterparty. *Email-verified* or *Domain-verified* is a sufficient basis for a connection **where both parties agree** to accept it; this is the working position for this version of the addendum (W4).
3. Discovery and connection policies MAY treat a higher assurance level as preferable.

*Editor's note.* Assurance above *Self-asserted* is either attested by an Operator that is itself conformance-verified, or derived from the LEI/vLEI ecosystem. Verifiable credentials, with vLEI as one profile, are OPTIONAL in this version and are designed to be switched on as the stakes of fraud rise, for example once carbon figures carry financial value (W3).

### 3.6 Organisational hierarchy

Entities MAY be related in a parent/subsidiary hierarchy. The PACT Network does not maintain its own hierarchy model; it references the relationship data of the underlying identifier.

1. Where an Entity is identified by an LEI, parent and ultimate-parent relationships SHOULD be derived from **LEI Level-2 relationship data** rather than asserted separately within PACT.
2. An Entity that does not hold an LEI MUST NOT declare a hierarchy relationship; hierarchy is available only once an Entity is identified by an LEI.
3. Hierarchy relationships are informational. A parent/subsidiary relationship MUST NOT, by itself, grant a Node access to another Entity's footprints; access is governed solely by the connection and authorisation mechanisms of §6.

This treatment resolves the common case where two related operating entities (for example, two subsidiaries of the same group) act as supplier and customer to each other for different products: each operating Entity holds its own identity, and the supplier/customer direction is a property of the *connection* between their Nodes (§6), not of their identities.

*Editor's note.* With the LEI now optional, hierarchy is an optional feature that becomes available to Entities that hold one. PACT still does not take on hierarchy maintenance.

### 3.7 Identity verification options

*This subsection sets out the verification methods side by side. They are **layers, not alternatives**: an Entity can hold evidence from several methods at once, and moving up a layer never requires re-registering or changing identifiers.*

| | Email ownership | Domain ownership | LEI status check | Verifiable credential (vLEI profile) |
|---|---|---|---|---|
| **Proves** | A representative controls a mailbox at the Entity's domain | The Entity controls the domain (DNS or web root) | The identifier exists, is `ISSUED`, and passed issuer KYB | Cryptographic proof of organisational identity, optionally of a delegated role |
| **Does not prove** | That the person may act for the Entity; that the domain belongs to the legal Entity named | That the domain belongs to the legal Entity named | That the presenting party is the LEI holder | — (subject to issuer trust and revocation) |
| **Who checks** | Entity's Operator, at onboarding | Entity's Operator, at onboarding; for Operators themselves, the PACT Conformance Service (§4.3) | Entity's Operator | Operator at onboarding, and/or the counterparty at connection time |
| **Cost / friction for the Entity** | Very low: click a link or enter a code | Low: IT adds a DNS record or file | LEI fees; renewal each year | LEI plus vLEI issuance through a QVI; key management |
| **Supports level** | Email-verified | Domain-verified | Registry-verified (with domain) | Credential-verified |
| **Status in this version** | In scope | In scope; REQUIRED for listed Operators | In scope, OPTIONAL | In scope, OPTIONAL; generic VC architecture, vLEI as one profile |

How an Operator performs the checks for its own customers is **not specified** by this addendum. Only the published result is: level, method, attesting Discovery Service, and timestamp (§3.5.2, §5.7). The descriptions in §3.7.1–§3.7.3 are *non-normative* and describe typical practice. The domain challenge that the PACT Conformance Service applies to Operators themselves is normative, and is defined in §4.3.3.

#### 3.7.1 Email ownership

*Non-normative.* The Operator sends a single-use, time-limited challenge (a link or code) to an address at a domain the Entity claims, and records a successful response. The verified *domain*, not the email address, is what is published (§5.7), so no personal data leaves the Operator.

#### 3.7.2 Domain ownership

*Non-normative.* The Operator issues a random token and confirms that it is published under the domain, typically as a DNS `TXT` record or as a file served over HTTPS from the domain. Operators are encouraged to use the same challenge format the PACT Conformance Service uses for listing (§4.3.3), so that one tool covers both.

#### 3.7.3 LEI status check

*Non-normative.* The Operator resolves the LEI in the Global LEI Index and confirms status `ISSUED`, and re-checks periodically (the Global LEI System renews registrations every year; a monthly re-check is common practice). Only the LEI value and the check outcome are published. LEI reference data is not republished as authoritative.

*Normative.* A Discovery Service MUST NOT continue to publish *Registry-verified* for an Entity whose LEI it knows to have left `ISSUED` status (for example `LAPSED`, `MERGED`, `RETIRED`, `ANNULLED`).

#### 3.7.4 Verifiable credentials

The Entity, or an Operator acting for it, presents a verifiable credential binding the organisational identifier to a key it controls. Verification checks the issuance chain to the trust anchor of the credential profile and confirms the credential is not revoked.

1. This addendum defines the **generic** requirements (chain verified, revocation checked, outcome and timestamp recorded) and treats each ecosystem as a **profile**. The **vLEI profile** [VLEI-EGF] is defined first, with the GLEIF root of trust as its anchor.
2. **Entity-level** credentials are in scope. A **delegated-role** credential, by which an Operator proves it is the authorised exchange agent for an Entity, is a named extension. Natural-person credentials are out of scope.
3. Where a credential is bound, its key MAY also be used to sign connection requests (§6, Option C2).

*Editor's note.* Whether generic W3C VC/DID credentials beyond the vLEI profile are accepted in this version is still open. The profile structure is there so they can be added without changing §3.5.

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC8141] — Uniform Resource Names (URNs).
- [RFC8615] — Well-Known Uniform Resource Identifiers.
- [ISO17442] — Legal Entity Identifier (LEI).
- [VLEI-EGF] — vLEI Ecosystem Governance Framework, GLEIF.
- [VC-DATA-MODEL] — W3C Verifiable Credentials Data Model.
