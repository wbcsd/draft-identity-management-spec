# PACT Identity Management Addendum — §4 Registration (Draft)

*Draft v0.2 — 16 September 2026. Draft normative text for the Registration section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Builds on §3 (Identity Model) and reflects the design decisions of 18 June 2026 as amended by the Technology Working Group session of 2 September 2026 (see the addendum outline, §12.1). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review. Attribute names and URN strings are provisional pending WG ratification. **OAuth 2.0 Dynamic Client Registration [RFC7591]** is assumed as the credential-provisioning binding (§6).
>
> **Changes in v0.2 (following the 2 September WG session).** Restructured around W1 and W2:
> - The central index no longer holds Entity or Node records, or delegations keyed by identifier. It is a **PACT Directory of Discovery Services** (§4.2).
> - Listing in the Directory is **gated on passing the discovery conformance tests**, which include a **domain-ownership check** of the Operator (§4.3). This replaces the Registry-run proof of endpoint control of v0.1 §4.5.2. Base-specification exchange conformance remains an attribute and is not a gate.
> - Registration of Entities and Nodes is an Operator-internal matter. This addendum specifies only what a Discovery Service publishes about them (§4.5, §5.7).
> - LEI handling and verifiable credentials move to §3.5 and §3.7. Regional roots and the delegation hierarchy become a non-normative future extension (§4.7).

---

## 4. Registration

### 4.1 Introduction

Registration in the PACT Network happens at two levels, with different owners and different degrees of specification:

1. **Listing a Discovery Service in the PACT Directory.** An Operator applies to have its Discovery Service listed. Listing is conformance-gated and fully specified here (§4.3, §4.4).
2. **Registering an Entity and its Nodes with an Operator.** A buyer or supplier becomes discoverable through the Operator that runs its Node(s). How the Operator onboards it is not specified. What the Operator's Discovery Service then publishes about it is (§4.5, §5.7).

Registration at either level is deliberately narrow. It is **not** an authorisation step and **not** the point at which any credential is issued:

- Listing and registration MUST NOT, by themselves, grant any party access to any other party's footprints. Access is established only through the connection and authorisation mechanisms of §6.
- The PACT Directory MUST NOT hold buyer or supplier data: no Entity records, no Node records, and no index of `companyIds` values (W1).
- The PACT Directory and Discovery Services MUST NOT issue, hold, escrow, or relay OAuth client credentials on behalf of another party. Those credentials are provisioned peer-to-peer between two Nodes (§6).

*Editor's note.* The PACT Directory answers one question: *which Discovery Services exist, and can they be trusted to answer for their customers?* Everything about a specific buyer or supplier stays with the Operator that serves it. This is what keeps PACT out of the credential and data path and addresses the solution-provider concern about being cut out of the relationship with their customers.

### 4.2 The PACT Directory

1. PACT MUST operate a single **PACT Directory** listing the Discovery Services of the PACT Network.
2. For each listed Discovery Service, the Directory MUST hold the attributes of §4.3.2 and nothing about the Entities that Service answers for.
3. The Directory MUST publish its listing through the read interface defined in §10. Who may read it is an open decision tied to Option B of §5.6.
4. The Directory MUST NOT replicate LEI reference data, or any other registry's reference data, about Operators beyond the identifiers they declare.

### 4.3 Listing a Discovery Service

#### 4.3.1 Who may apply

Any Operator that runs a Discovery Service MAY apply for listing. This includes:

- a [=solution provider=] answering for many customers (Multi-Party Entity, §3.3); and
- an Entity that runs its own Node(s) and answers only for itself (the *SP-of-one* case, §3.2 requirement 5). An SP-of-one is listed exactly like any other Operator and passes the same tests.

#### 4.3.2 Listing attributes

A Directory listing MUST carry:

| Attribute | Requirement | Notes |
|---|---|---|
| `serviceId` | REQUIRED | Stable identifier assigned by the Directory. Never reassigned (§4.4.3). |
| `operator.companyIds` | REQUIRED | Array of URNs per §3.4 identifying the Operator as an Entity. |
| `operator.name` | REQUIRED | The Operator's legal or trading name. |
| `domains` | REQUIRED | One or more Internet domains whose control the Operator demonstrated (§4.3.3). |
| `discoveryEndpoint` | REQUIRED | HTTPS URL of the Discovery Service (§10). MUST be under one of `domains`. |
| `discoveryConformance` | REQUIRED | Result of the discovery conformance tests: test-suite version and date passed (§4.3.4). |
| `conformantVersions` | REQUIRED | Base-specification versions for which the Operator's solution holds PACT conformance. MAY be empty (§4.3.5). |
| `domainVerifiedAt` | REQUIRED | Time the domain challenge last succeeded. |
| `status` | REQUIRED | `active` or `suspended` (§4.4.2). |
| `jwksUri` | CONDITIONAL | Required only if the WG adopts request signing (Option B1 of §5.6 or Option C2 of §6.4). MUST be under one of `domains`. |
| `contact` | REQUIRED | Operational and security contact. MUST NOT be published to other participants. |

The Directory MUST NOT require further attributes as a condition of listing.

#### 4.3.3 Domain ownership of the Operator

Before a Discovery Service is listed, the PACT Conformance Service MUST verify that the applicant controls every domain in `domains` (W2). The verification is part of the discovery conformance tests (§4.3.4).

1. The Conformance Service MUST issue a random token, at least 128 bits, for each domain and confirm it is published under that domain by at least one of:
   - a DNS `TXT` record at `_pact-challenge.<domain>` whose value is the token; or
   - an HTTPS resource at `https://<domain>/.well-known/pact-challenge/<token-id>` whose body is the token, served with a valid TLS certificate for `<domain>`.
2. Tokens MUST be single-use and MUST expire, RECOMMENDED within 7 days of issue.
3. The Conformance Service MUST record the method used and the time of success, and the Directory MUST expose the latter as `domainVerifiedAt`.
4. Domain control MUST be re-verified on every change to `domains`, `discoveryEndpoint`, or `jwksUri`, and at every conformance re-test.

*Editor's note.* This is the v0.1 "proof of endpoint control" moved to the level where the WG placed it. Binding `discoveryEndpoint` (and, where adopted, `jwksUri`) to a verified domain means an attacker cannot list a Discovery Service that impersonates an existing Operator. It does not stop a *listed* Operator from publishing false information about its own customers. That risk is contained by accountability and delisting (§4.4.2), not by cryptography. The `_pact-challenge` label and the `.well-known` suffix need registering or confirming; that is for §10.

#### 4.3.4 Conformance-gated listing

1. A Discovery Service MUST NOT be listed until its Operator has passed the **discovery conformance tests** run by the PACT Conformance Service (W2).
2. The discovery conformance tests MUST cover at least: domain ownership (§4.3.3); the discovery query interface and response format (§5.7, §10); correct handling of unknown and non-visible identifiers (§5.8); and, where the Operator declares support for the *Credential-Exchange-capable Node* class, the connection and RFC 7591 interfaces (§6, §10).
3. A home-built solution that has not passed the discovery conformance tests is not eligible for listing, whatever its technical merit. It becomes eligible once it passes.
4. The Directory MUST record the test-suite version passed. Where a new test-suite version is released, the Directory MUST NOT delist an Operator solely for not yet having passed it before a transition period announced with the release has ended.

*Editor's note.* Requirement 4 depends on parked item P1 (how SPs are notified of registry updates and new-version roll-outs). The transition period is deliberately not fixed here.

#### 4.3.5 Exchange conformance as an attribute

1. `conformantVersions` MUST list the base-specification version(s) for which the Operator's solution currently holds PACT conformance, and MUST be empty where it holds none.
2. The Directory MUST source `conformantVersions` from the PACT Conformance Service and MUST NOT accept it as a self-assertion.
3. An empty `conformantVersions` MUST NOT prevent listing. Counterparties MAY use it to decide whether to attempt a connection.

### 4.4 Maintaining a listing

#### 4.4.1 Update

1. An Operator MUST keep `discoveryEndpoint`, `domains`, `contact`, and (where applicable) `jwksUri` current.
2. A change to `domains`, `discoveryEndpoint`, or `jwksUri` MUST trigger re-verification (§4.3.3). The changed value MUST NOT be published until re-verification succeeds; the previous value remains in force meanwhile.
3. The Directory SHOULD run a periodic liveness check against listed Discovery Services and SHOULD expose the result. It MUST NOT delist a Service solely because it is temporarily unreachable.

#### 4.4.2 Suspension

The Directory MUST set a listing to `suspended`, and clients MUST NOT query a suspended Service (§5.4), where:

- domain control can no longer be demonstrated (§4.3.3);
- the Operator fails a conformance re-test, subject to §4.3.4 requirement 4; or
- PACT determines, under a published procedure, that the Service has published materially false information, for example answering for Entities it does not serve.

A suspension MUST be reflected in the Directory listing without delay. The procedure for determining misbehaviour and for reinstatement is operational and is expected in an annex.

#### 4.4.3 Delisting

1. An Operator MAY request delisting at any time. Delisting removes the Service from the Directory.
2. Delisting or suspension MUST NOT be treated as, or relied upon as, a means of revoking exchange credentials. Credentials already provisioned between Nodes remain valid until revoked through the connection lifecycle of §6.7.
3. The Directory MUST NOT reassign a `serviceId` and SHOULD retain a tombstone sufficient to answer "this Service was listed and has been withdrawn".
4. Delisting MUST NOT delete the audit record of listing events (§7).

### 4.5 Entities and Nodes at a Discovery Service

An Operator decides how it onboards the Entities it serves: what it asks for, which checks it runs, and how its customers opt in. This addendum constrains only what the Operator's Discovery Service **publishes** (editorial position E2, §12.2).

1. A Discovery Service MUST answer only for Entities whose Node(s) its Operator runs, or for itself in the SP-of-one case.
2. A Discovery Service MUST publish an Entity only where that Entity has opted in to being discoverable (§5.8).
3. The assurance published for an Entity MUST satisfy §3.5.2: it identifies the attesting Service, reflects checks that Service actually performed or credentials it verified, and carries per-method timestamps.
4. For each Node, a Discovery Service MUST publish at least `baseUrl` (HTTPS) and, where the Node is Credential-Exchange-capable, `connectionEndpoint`. A Node's `token_endpoint` and `registration_endpoint` MUST NOT be published as authoritative. Both are read from the Node's own `$base-url$/.well-known/openid-configuration` (§5.7).
5. A Node that supports automated credential provisioning MUST advertise a `registration_endpoint` in that well-known document, as defined by [RFC8414]/[RFC7591]. Its presence is the machine-readable signal that the Node implements the *Credential-Exchange-capable Node* conformance class (§9).
6. A Discovery Service MAY mark a Node as `provisional` (a demo Node offered as an on-ramp to Entities without exchange tooling). A provisional Node MUST be marked as such in every response.
7. When an Entity ceases to be served, or withdraws its opt-in, the Discovery Service MUST stop publishing it. As in §4.4.3, this is not credential revocation.

*Editor's note — portability.* v0.1 §4.3(3) made it normative that an Entity can take over or withdraw a registration an Operator made on its behalf, to prevent an Operator holding a customer's network identity hostage. Because onboarding is now Operator-internal, that requirement has no normative place in this version. Two things partly replace it: an Entity may be served by more than one Operator (§3.2 requirement 2), and clients aggregate answers from every Service (§5.4). The WG should decide whether SP portability needs a normative hook, for example a conformance test that an Operator honours a withdrawal request within a fixed time.

### 4.6 Relationship to credential provisioning

Listing and registration produce exactly the metadata that automated credential exchange (§6) consumes, and nothing more:

1. The PACT Directory lists Discovery Services. A Discovery Service resolves a `companyIds` value to Node `baseUrl` and `connectionEndpoint` (§5).
2. The Node's own well-known document supplies `token_endpoint` and, where supported, `registration_endpoint` (§4.5).
3. The two Nodes then establish trust and provision credentials **directly** (§6). Neither the Directory nor any Discovery Service is a party to that exchange. An established connection MUST NOT depend on the Directory or any Discovery Service being online.
4. The credentials so provisioned are the `client_id` and `client_secret` that the base specification's §5.5 client-credentials flow already consumes. This addendum introduces no change to that token flow.

### 4.7 Future extension: regional directories

*Non-normative.* The 18 June design (D9) allowed regional roots tied to a parent root, in the manner of DNS. This version has a single PACT Directory. The design keeps room for a later extension in which a **regional directory** (for example for a national network) lists its own Discovery Services, declares the PACT Directory as its parent, can be suspended by it, and is itself listed so that clients can follow it. The containment rules of §4.4.2 would apply to a regional directory as a whole. Nothing in this version depends on that extension.

### 4.8 Error conditions

*Provisional — the full error catalogue and its alignment with the base specification's error response belong to §10.* The listing process is expected to distinguish at least:

| Condition | Meaning |
|---|---|
| `DomainControlNotProven` | The domain challenge (§4.3.3) was not completed for one or more domains. |
| `EndpointNotUnderVerifiedDomain` | `discoveryEndpoint` or `jwksUri` is not under a verified domain. |
| `DiscoveryConformanceNotPassed` | The applicant has not passed the current discovery conformance tests (§4.3.4). |
| `OperatorAlreadyListed` | A listing with an overlapping `operator.companyIds` value already exists. |

---

## Open items carried from this section

- SP portability: whether an Entity's right to withdraw from, or move away from, an Operator needs a normative hook (§4.5 editor's note).
- Transition period for new discovery test-suite versions, tied to parked item P1 on notification and versioning (§4.3.4).
- Who may read the PACT Directory listing, tied to Option B (§5.6) and parked item P3.
- The misbehaviour and reinstatement procedure for suspension (§4.4.2): operational annex.
- Registration of the `_pact-challenge` DNS label and `.well-known/pact-challenge` suffix (§4.3.3), to be fixed in §10.
- Whether listing and lifecycle events MUST be audit-logged. The base specification leaves logging out of scope (§5.3(d)). Carried to §7.
- The "invite a non-member supplier to join" journey remains informative/product, not normative.

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3 (§5.3 out of scope; §5.5 authentication).
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC7591] — OAuth 2.0 Dynamic Client Registration Protocol.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC8615] — Well-Known Uniform Resource Identifiers.
- [RFC8141] — Uniform Resource Names (URNs).
