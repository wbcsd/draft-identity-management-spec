# PACT Identity Management Addendum — §3 Identity Model (Draft)

*Draft v0.1 — 24 June 2026. Draft normative text for the Identity Model section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Reflects the design decisions resolved on 18 June 2026 (see the addendum outline, §12). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review. Open items are called out inline as *Editor's notes* and consolidated in the addendum outline §12. Where this section defines a URN namespace or attribute name, the exact string is provisional pending WG ratification.

---

## 3. Identity Model

### 3.1 Introduction

This section defines the identity model that underpins both Node Discoverability (§5) and Automated Credential Exchange (§6). It introduces no change to the [=data model=] or the four core actions of the base specification; it defines the *participants* of the PACT Network, *how they are identified*, and *how strongly that identity is assured*, so that those participants can be discovered and can establish trust.

The model is deliberately layered so that a participant can take part with minimal friction (a self-asserted identifier) and later strengthen its identity (a verified, cryptographically provable identifier) without changing how it is represented. **Assurance is an attribute of an identity, not a precondition for holding one.**

This addendum reuses the base-specification terms [=host system=], [=data owner=], [=data recipient=], [=solution=], and [=solution provider=] without modification, and adds the terms defined below.

### 3.2 Entities, Operators, and Nodes

The identity model distinguishes three concepts:

- **Entity** — a legal organisation that owns or is responsible for PCF data (a [=data owner=] and/or [=data recipient=] in the sense of the base specification). An Entity is the subject of an organisational identity (§3.4).
- **Operator** — the party that technically operates one or more [=Nodes=]. An Operator MAY be the Entity itself, or a [=solution provider=] acting on the Entity's behalf.
- **Node** — an addressable PACT Network participant endpoint, realised by a base-specification [=host system=] that implements the four core actions. A Node is the unit that is registered in the directory (§4) and discovered (§5).

The following relationships hold:

1. An Entity MUST be represented on the network by at least one Node.
2. An Entity MAY be represented by more than one Node (for example, distinct business units or regions).
3. An Operator MAY operate Nodes on behalf of many Entities.
4. A Node MUST be associated with exactly one Entity as the [=data owner=] whose footprints it serves; this does not restrict the Node from also acting as a [=data recipient=].

*Editor's note.* The Node↔host-system mapping is intentionally 1:1 at the protocol level: a Node *is* a conformant host system plus its directory registration. The Operator concept exists to express the common case where one [=solution provider=] fronts many Entities (see the SP-node in §3.3 and §5).

### 3.3 Actor types

A participant relates to the network in one of the following ways. A conforming implementation SHOULD classify each registered Node as exactly one actor type; the type informs discovery (§5) and visibility (§5, §8) behaviour.

- **Single-Party Entity** — an Entity operating its own Node(s), acting as [=data owner=] and/or [=data recipient=] in a value chain. *Example: a buyer organisation hosting its own Node.*
- **Multi-Party Entity** — an Operator (typically a [=solution provider=]) operating Nodes on behalf of multiple Entities under its own profile, as a sub-network. Such an Operator MAY expose a single **SP-node** that fronts discovery for the Entities it represents (§5). *Example: "Solution Provider X" operating distinct Nodes for two of its customers.*
- **Reference Entity** — an Entity exposing a Node from which other participants MAY retrieve reference or PCF data on a read-only basis (for example, an emission-factor or reference-data provider). A Reference Entity's Node MUST implement the retrieval actions but MAY decline connection requests that imply write or event obligations.
- **Router Entity** — a party operating a Node purely as a connector between other Nodes, which is neither the [=data owner=] nor the [=data recipient=] of the footprints it conveys. A Router Entity MUST NOT present itself as the [=data owner=] of footprints it merely routes.

### 3.4 Organisational identifiers

Every Entity MUST be identified by at least one organisational identifier expressed as a Uniform Resource Name [RFC8141], carried in the base-specification `companyIds` array. This addendum does not introduce a new identifier attribute; it constrains and recommends the *values* of the existing one.

1. An Entity's `companyIds` MUST contain at least one identifier and MAY contain several (for example, a self-issued identifier together with a registry-issued one).
2. An Entity SHOULD include a **Legal Entity Identifier (LEI)** [ISO17442] among its `companyIds` where one is held. The LEI is the RECOMMENDED canonical organisational identifier for the PACT Network because it is global, openly resolvable via the Global LEI Index, and maintained against an authoritative source.
3. An Entity that does not hold an LEI MAY participate using any other resolvable URN identifier (for example a DUNS-based or self-issued URN, or a Decentralized Identifier). Such an identifier is treated as **self-asserted** (§3.5).
4. Implementations MUST treat `companyIds` as a set: two records sharing at least one identical identifier value refer to the same Entity. Implementations MUST NOT assume global uniqueness of self-asserted identifiers.

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

*Editor's note.* The exact LEI URN namespace (`urn:lei:…` vs. an LEI sub-type within the existing `urn:pfi:…` convention from the legacy `companyIds` proposal) is to be ratified by the WG and, if newly minted, registered. The intent — LEI as a first-class, recommended value of `companyIds` — is fixed.

### 3.5 Identity assurance

An organisational identity carries an **assurance level**, which is an attribute of the identity and does not affect how the Entity is represented (§3.4) or whether it may register (§4).

This addendum defines the following assurance levels:

- **Self-asserted** — the identifier is declared by the Entity (or its Operator) and has not been independently verified by the network. This is the default and imposes no onboarding friction.
- **Registry-verified** — the identifier is an LEI [ISO17442]; the Entity's existence and reference data have been validated by an LEI issuer against the authoritative source as part of LEI issuance (the network relies on this Know-Your-Business process rather than performing its own).
- **Credential-verified** — the Entity, or a delegated role acting for it, presents a verifiable credential (a **verifiable LEI (vLEI)** [VLEI-EGF]) that cryptographically proves the organisational identity. See §6 for how a Credential-verified identity is presented during trust establishment.

A conforming directory (§4) MUST record and expose the assurance level of each Entity identity. The network MUST NOT require any assurance level above *Self-asserted* as a condition of registration. Discovery and connection policies (§5, §6) MAY treat a higher assurance level as preferable.

*Editor's note.* The PACT Network does not issue or verify organisational identities itself; assurance above *Self-asserted* is derived entirely from the LEI/vLEI ecosystem. vLEI support is OPTIONAL in this version of the addendum; entity-level vLEI is in scope, delegated-role vLEI is defined as an extension (§6), and natural-person credentials are out of scope.

### 3.6 Organisational hierarchy

Entities MAY be related in a parent/subsidiary hierarchy. The PACT Network does not maintain its own hierarchy model; it references the relationship data of the underlying identifier.

1. Where an Entity is identified by an LEI, parent and ultimate-parent relationships SHOULD be derived from **LEI Level-2 relationship data** rather than asserted separately within PACT.
2. An Entity that does not hold an LEI MUST NOT declare a hierarchy relationship; hierarchy is available only once an Entity is identified by an LEI.
3. Hierarchy relationships are informational. A parent/subsidiary relationship MUST NOT, by itself, grant a Node access to another Entity's footprints; access is governed solely by the connection and authorisation mechanisms of §6.

This treatment resolves the common case where two related operating entities (for example, two subsidiaries of the same group) act as supplier and customer to each other for different products: each operating Entity holds its own identity, and the supplier/customer direction is a property of the *connection* between their Nodes (§6), not of their identities.

*Editor's note.* Restricting hierarchy to LEI-identified Entities both avoids PACT taking on hierarchy maintenance (repeatedly cited by members as a major pain) and creates a concrete incentive to obtain an LEI.

---

### References used in this section (to be consolidated into the addendum References)

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC8141] — Uniform Resource Names (URNs).
- [ISO17442] — Legal Entity Identifier (LEI).
- [VLEI-EGF] — vLEI Ecosystem Governance Framework, GLEIF.
