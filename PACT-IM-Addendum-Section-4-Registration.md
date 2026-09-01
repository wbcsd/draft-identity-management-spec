# PACT Identity Management Addendum — §4 Registration (Draft)

*Draft v0.1 — 1 September 2026. Draft normative text for the Registration section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Builds directly on §3 (Identity Model) and reflects the design decisions resolved on 18 June 2026 (see the addendum outline, §12). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review. Two assumptions are taken as settled for the purpose of this draft and are flagged where they bind: (1) the **LEI [ISO17442] is the primary organisational identifier** of the PACT Network; (2) **OAuth 2.0 Dynamic Client Registration [RFC7591]** is the mechanism by which exchange credentials are auto-provisioned. Attribute and URN strings are provisional pending WG ratification.

---

## 4. Registration

### 4.1 Introduction

Registration is the act by which an Entity (§3.2) and the Node(s) representing it become **known to a Registry**, and therefore resolvable (§5) and connectable (§6). A registration record binds three things that are otherwise unrelated:

1. an **organisational identity** — one or more identifiers in `companyIds`, primarily an LEI (§3.4);
2. one or more **network endpoints** — the Nodes through which that Entity exchanges PCF data; and
3. the **assurance** attached to that binding — how the network came to believe that the identity and the endpoints belong together (§3.5).

Registration is deliberately narrow. It is **not** an authorisation step, **not** a conformance gate, and **not** the point at which any credential is issued:

- Registration MUST NOT, by itself, grant any party access to any other party's footprints. Access is established only through the connection and authorisation mechanisms of §6.
- Registration MUST NOT be conditional on the registrant holding a PACT conformance status. A Node's conformance is recorded as an **attribute** of its record (§4.5.3), visible to counterparties, not as an admission criterion.
- A Registry MUST NOT issue, hold, escrow, or relay the OAuth client credentials used by the base specification's §5.5 token flow. Those credentials are provisioned peer-to-peer between two Nodes (§6, §4.8).

*Editor's note.* This narrowness is the point. The registry is a **thin index**: identifier → endpoint (or → a delegated discovery service), plus a small set of attributes needed to decide whether to attempt a connection. Everything with commercial or security weight happens between the two Nodes afterwards. This is what keeps PACT out of the credential and data path.

### 4.2 Where registration happens

Consistent with the hybrid-root discovery architecture (§5.1), registration occurs at a **Registry**, of which there are three kinds:

- the **Root Registry** operated for the PACT Network as a whole;
- a **Regional Registry** — a root operated for a national or sectoral network, tied to a parent root; and
- a **Solution Provider Directory** — a discovery service operated by a Multi-Party Entity (§3.3) that registers, and answers on behalf of, the Entities it fronts.

The following requirements apply:

1. An Entity MUST be registered in exactly one **authoritative** Registry per identifier. That Registry is the one whose record the root index resolves to.
2. A Root Registry MUST hold, for each registered identifier, either (a) the Node metadata directly, or (b) a **delegation** to the Registry that holds it. It MUST NOT hold both for the same identifier.
3. A Regional Registry or Solution Provider Directory MUST declare a parent Registry, and MUST publish the identifier scope for which it is authoritative.
4. A Registry MUST NOT replicate LEI reference data beyond what §4.4.2 permits.

Registration of the Registries themselves is defined in §4.7.

### 4.3 Who may register

A registration request is made by a **Registrant**, which is either the Entity itself or an Operator (§3.2) acting on its behalf.

1. A Registry MUST record which party submitted each registration and under what claim of authority.
2. Where an Operator registers an Entity it does not itself constitute, the Operator MUST assert delegated authority for that Entity. In this version, that assertion MAY be self-declared; where a delegated-role verifiable credential is presented instead, the record is treated as credential-verified for the delegation (§4.6.3).
3. An Entity MUST be able to take over, correct, or withdraw a registration made on its behalf, upon demonstrating control of its own identifier at an assurance level at least equal to that of the existing record. A Registry MUST provide such a procedure.

*Editor's note.* Requirement 3 is the practical protection against an Operator holding an Entity's network identity hostage — an SP-portability concern raised repeatedly by members. The mechanism (what "demonstrating control" means at each assurance level) is intentionally left to §4.6 and to Registry policy for now.

### 4.4 Registering an Entity

#### 4.4.1 Entity registration attributes

A registration request for an Entity MUST carry at least:

| Attribute | Requirement | Notes |
|---|---|---|
| `companyIds` | REQUIRED | Array of URNs per §3.4. SHOULD contain an `urn:lei:…` value. |
| `legalName` | REQUIRED | The Entity's legal name as the registrant asserts it. |
| `actorType` | REQUIRED | One of the actor types of §3.3. |
| `contact` | REQUIRED | An operational contact for network and security matters. Not published to anonymous requesters (§8). |
| `jurisdiction` | OPTIONAL | ISO 3166 country of legal registration; MAY be omitted where derivable from the LEI. |
| `parentRef` | OPTIONAL | Permitted only for LEI-identified Entities (§3.6). |

A Registry MUST NOT require attributes beyond those listed as REQUIRED here as a condition of registration.

#### 4.4.2 Handling of the LEI

Where the registrant supplies an LEI:

1. The Registry MUST verify that the LEI resolves in the Global LEI Index and that its registration status is `ISSUED` before recording the Entity's assurance level as *Registry-verified* (§3.5).
2. The Registry MUST store the LEI value itself, and MAY cache only the minimum reference data needed to render a human-readable result — legal name, jurisdiction, LEI registration status, and the timestamp of the last check. It MUST NOT replicate the Global LEI Index or present cached reference data as authoritative.
3. The Registry MUST re-check the LEI registration status periodically. A recommended interval is **not less than once every 30 days**.
4. Where an LEI transitions out of `ISSUED` (for example to `LAPSED`, `MERGED`, `RETIRED`, or `ANNULLED`), the Registry MUST downgrade the recorded assurance level and reflect the new status (§4.9.2). It MUST NOT continue to present the Entity as *Registry-verified*.
5. Where an LEI is `MERGED` into a successor, the Registry SHOULD record a pointer to the successor LEI and SHOULD NOT silently delete the record (§4.9.3).

Where the registrant supplies no LEI, the Entity is registered at the *Self-asserted* assurance level (§3.5) and MUST NOT declare hierarchy relationships (§3.6, requirement 2). Registration MUST NOT be refused on this basis alone.

*Editor's note.* Requirements 1 and 3 are the whole of PACT's "know your business partner" position: the network **inherits** the KYB performed by the LEI issuer and re-checks that it still holds, rather than performing any verification of its own. Where a Registry operator nonetheless performs manual checks during the transition period, §4.5.2 applies.

#### 4.4.3 Identifier collision and duplicate detection

1. A Registry MUST treat `companyIds` as a set. If a submitted identifier value is already present on another Entity record in that Registry, the Registry MUST NOT create a second record; it MUST either reject the request or route it to the takeover procedure of §4.3(3).
2. A Registry MUST NOT infer that two records are the same Entity from name similarity, endpoint similarity, or shared contact details.
3. Self-asserted identifiers MUST NOT be assumed globally unique. A Registry MAY namespace them internally, and MUST NOT surface a self-asserted identifier in discovery responses as though it were authoritative.

### 4.5 Registering a Node

#### 4.5.1 Node registration attributes

Each Node registered against an Entity MUST carry at least:

| Attribute | Requirement | Notes |
|---|---|---|
| `nodeId` | REQUIRED | Stable identifier of the Node within the Registry. |
| `entityRef` | REQUIRED | The `companyIds` value of the Entity the Node represents (§3.2, requirement 4). |
| `operatorRef` | REQUIRED | Identifier of the Operator. Equal to `entityRef` where self-operated. |
| `baseUrl` | REQUIRED | The base-specification `$base-url$` of the host system realising the Node. MUST use `https`. |
| `actorType` | REQUIRED | Per §3.3; MUST be consistent with the Entity's declared type. |
| `conformantVersions` | REQUIRED | Array; MAY be empty (§4.5.3). |
| `visibility` | REQUIRED | The two controls of §4.5.4. |
| `connectionEndpoint` | REQUIRED for Credential-Exchange-capable Nodes | Where connection requests are received (§6). |

A Registry MUST NOT store the Node's token endpoint or client registration endpoint as authoritative values. Both are discovered from the Node itself (§4.5.2).

#### 4.5.2 Endpoint metadata and proof of control

The base specification already requires a host system to publish OAuth 2.0 authorisation server metadata at `$base-url$/.well-known/openid-configuration`. This addendum reuses that document rather than duplicating its contents in the Registry:

1. A Node's `token_endpoint` MUST be obtained from that well-known document, not from the Registry record.
2. A Node that supports automated credential provisioning MUST advertise a `registration_endpoint` in that same document, as defined by [RFC8414]/[RFC7591]. Its presence is the machine-readable signal that the Node implements the *Credential-Exchange-capable Node* conformance class (§9).
3. A Registry MAY cache these values for display, and MUST mark cached values as non-authoritative.

Before a Node record becomes resolvable to any other participant, the Registry MUST verify that the Registrant controls the declared `baseUrl`. A conforming Registry MUST implement at least one **proof-of-control** challenge, and MUST record which was used and when. RECOMMENDED challenges are:

- **Endpoint challenge** — the Registry issues a nonce and requires it to be served from a Registry-specified path under `$base-url$`; or
- **Signed challenge** — the Registrant returns the nonce signed by a key bound to a verifiable credential presented under §4.6.

A Registry MUST re-verify control at least on every change to `baseUrl`, and SHOULD re-verify periodically thereafter.

*Editor's note.* Proof of endpoint control is the single most important anti-impersonation control at registration time: without it, the identifier-to-endpoint binding — the only thing the directory actually asserts — is unverified, and an attacker can point a legitimate LEI at an endpoint they own. It is the direct analogue of domain validation in the DNS/PKI world the architecture is modelled on. Whether the well-known nonce path is newly minted or reuses an existing PACT path is for §10.

#### 4.5.3 Conformance as an attribute

1. A Node's `conformantVersions` MUST list the version(s) of the base specification for which the Node currently holds a PACT conformance status, and MUST be empty where it holds none.
2. A Registry MUST source `conformantVersions` from the PACT Conformance Service and MUST NOT accept the value as a self-assertion by the Registrant.
3. An empty `conformantVersions` MUST NOT prevent registration, resolution, or the receipt of connection requests. It MAY be used by a counterparty as a basis for declining a connection.
4. A Registry MAY offer a **provisional demo Node** as an on-ramp for Entities that hold no exchange tooling. Such a Node MUST be marked as provisional in every discovery response and MUST NOT report any conformant version.

#### 4.5.4 Visibility at registration

Every Node record carries two orthogonal visibility controls, set by the Node owner (§5.3, §8):

- **Resolvability** — whether a party that already knows the Entity's identifier can resolve it to this Node and submit a connection request. The default MUST be *on for authenticated network members* and *off for anonymous requesters*.
- **Listability** — whether the Node appears in search or browse results. The default MUST be *off*. A Node owner MAY opt in to *members-only* or *public* listing.

A Registry MUST apply these defaults where the Registrant does not state a preference, and MUST allow the Node owner to change them at any time. Neither control affects §6: a resolvable or listed Node still approves every connection individually.

#### 4.5.5 Example registry record

*Non-normative.* An Entity registered with an LEI, self-operating one Node, credential-exchange capable, default visibility:

```json
{
  "entity": {
    "companyIds": [
      "urn:lei:5493001KJTIIGC8Y1R12",
      "urn:pfi:www.example.com:org-id:401765"
    ],
    "legalName": "Example Manufacturing B.V.",
    "actorType": "single-party-entity",
    "assurance": {
      "level": "registry-verified",
      "source": "gleif-lei-index",
      "leiRegistrationStatus": "ISSUED",
      "lastCheckedAt": "2026-08-28T04:00:00Z"
    }
  },
  "nodes": [
    {
      "nodeId": "urn:pact:node:5493001KJTIIGC8Y1R12:eu-1",
      "entityRef": "urn:lei:5493001KJTIIGC8Y1R12",
      "operatorRef": "urn:lei:5493001KJTIIGC8Y1R12",
      "baseUrl": "https://pact.example.com",
      "actorType": "single-party-entity",
      "conformantVersions": ["2.3", "3.0"],
      "connectionEndpoint": "https://pact.example.com/3/connections",
      "visibility": { "resolvability": "members", "listability": "none" },
      "endpointControl": {
        "method": "endpoint-challenge",
        "verifiedAt": "2026-08-14T11:07:52Z"
      }
    }
  ]
}
```

*Non-normative.* A Solution Provider Directory record for an Entity it fronts — the root index holds a **delegation**, not an endpoint:

```json
{
  "entity": {
    "companyIds": ["urn:lei:894500ABCDEF12345678"],
    "legalName": "Example Supplier Ltd",
    "assurance": { "level": "registry-verified", "source": "gleif-lei-index" }
  },
  "delegation": {
    "discoveryService": "urn:lei:213800SPPROVIDER00001",
    "discoveryEndpoint": "https://directory.sp-example.com/3/discovery",
    "discoverable": true
  }
}
```

### 4.6 Identity proofing and assurance

#### 4.6.1 Principle

The PACT Network does not proof organisational identity itself. Assurance recorded at registration is **derived** — from the LEI issuance process, or from a verifiable credential issued within the vLEI ecosystem — and is recorded as an attribute of the Entity identity (§3.5), never as a precondition for holding one.

1. A Registry MUST record, for every Entity, the assurance level, the source from which it was derived, and the time it was last confirmed.
2. A Registry MUST NOT record an assurance level it did not itself derive from a check it performed or a credential it verified.
3. A Registry MUST make the assurance level and its timestamp available in discovery responses, subject to §4.5.4 visibility.

#### 4.6.2 Interim manual verification

Where a Registry operator performs a manual check of a registrant during the transition period before broad LEI coverage:

1. The result MUST be recorded as a distinct, clearly labelled interim status and MUST NOT be presented as *Registry-verified*.
2. The interim status MUST carry an expiry and MUST be re-confirmed or downgraded at expiry.

*Editor's note.* This exists only to avoid a hard dependency on LEI coverage (~40–60% of non-financial corporates today) blocking early adopters. It is explicitly a stopgap; the WG should decide whether it belongs in the normative text at all or in an operational annex.

#### 4.6.3 Verifiable credentials (optional path)

An Entity, or an Operator acting for it, MAY bind its registration to a **verifiable LEI (vLEI)** [VLEI-EGF]. This path is OPTIONAL in this version of the addendum.

1. Where a Registry accepts verifiable credentials, it MUST verify the credential's issuance chain to the GLEIF root of trust and MUST verify that the credential is not revoked at the time of the check.
2. On successful verification, the Entity's assurance level MUST be recorded as *Credential-verified*, together with the credential identifier, issuer, and verification timestamp.
3. A Registry MUST re-check revocation status at an interval no longer than that used for LEI status (§4.4.2, requirement 3), and MUST downgrade the assurance level where the credential is revoked or expired.
4. **Entity-level** credentials are in scope for this version. A **delegated-role** credential — by which an Operator proves it is the authorised exchange agent for an Entity — is defined as a named extension and, where presented, MUST be recorded against the delegation asserted under §4.3(2). Natural-person credentials are out of scope.
5. Where a Node has bound a credential, its public key material MAY be used for the signed proof-of-control challenge of §4.5.2 and for identity verification at connection time (§6.1).

*Editor's note.* The credential presentation and verification mechanics (the vLEI ecosystem's own key-event and credential-chaining infrastructure) are referenced, not restated, here. The addendum should state *what must be true* — chain verified to the GLEIF root, revocation checked, outcome recorded and time-stamped — and defer the *how* to [VLEI-EGF]. Whether generic W3C VC/DID credentials are accepted alongside vLEI is still open.

### 4.7 Registration of Discovery Services

A Registry other than the Root Registry is itself registered, so that queries can be routed to it and its answers trusted.

1. The operator of a Regional Registry or Solution Provider Directory MUST itself be a registered Entity, and SHOULD be *Credential-verified* (§4.6.3).
2. A discovery service registration MUST declare: the operating Entity's identifier, the discovery endpoint URL, the parent Registry, and the identifier scope for which the service is authoritative.
3. A parent Registry MUST verify control of the declared discovery endpoint using the mechanism of §4.5.2 before routing any query to it.
4. A parent Registry MUST be able to suspend a child discovery service, and MUST cease routing queries to a suspended service.
5. A discovery service MUST NOT answer authoritatively for identifiers outside its declared scope. A Registry receiving such an answer MUST discard it.

*Editor's note.* Requirements 4 and 5 are the containment story for the "rogue discovery service" threat named in §7 — a compromised or misbehaving sub-directory can lie only about the identifiers it was delegated, and can be cut off by its parent.

### 4.8 Relationship to credential provisioning

Registration produces exactly the metadata that automated credential exchange (§6) consumes, and nothing more:

1. The Registry resolves an identifier to a Node's `baseUrl` and `connectionEndpoint`.
2. The Node's own well-known document supplies the `token_endpoint` and, where supported, the `registration_endpoint` [RFC7591] (§4.5.2).
3. The two Nodes then establish trust and provision credentials **directly** (§6.1, §6.2). The Registry is not a party to that exchange and MUST NOT be required to be online for an established connection to continue functioning.
4. The credentials so provisioned are the `client_id` and `client_secret` that the base specification's §5.5 client-credentials flow already consumes. This addendum introduces no change to that token flow.

### 4.9 Maintaining a registration

#### 4.9.1 Update

1. A Registrant MUST keep `baseUrl`, `connectionEndpoint`, and `contact` current.
2. A change to `baseUrl` MUST trigger re-verification of endpoint control (§4.5.2), and the Node MUST NOT be resolvable at the new URL until that verification succeeds.
3. A Registry SHOULD perform a liveness check against registered Nodes and SHOULD expose the outcome as an attribute; it MUST NOT remove a record solely because a Node is temporarily unreachable.

#### 4.9.2 Suspension and downgrade

A Registry MUST suspend a Node record, or downgrade an Entity's assurance level, where:

- the underlying LEI leaves `ISSUED` status (§4.4.2, requirement 4) — assurance downgraded to *Self-asserted*;
- a bound verifiable credential is revoked or expires (§4.6.3, requirement 3) — assurance downgraded to the next level still supported by evidence;
- endpoint control can no longer be demonstrated (§4.5.2) — the Node record suspended from resolution; or
- the parent Registry suspends the discovery service holding the record (§4.7, requirement 4).

A suspended or downgraded record MUST reflect the change in every discovery response. A Registry MUST NOT present stale assurance.

#### 4.9.3 Deregistration

1. A Node owner MAY deregister a Node at any time. Deregistration removes the Node from resolution and listing.
2. Deregistration MUST NOT be treated as, or relied upon as, a means of revoking exchange credentials. Credentials already provisioned remain valid at the counterparty Node until revoked through the connection lifecycle of §6.3. A Registry SHOULD warn a deregistering owner of any outstanding connections it is aware of.
3. A Registry MUST NOT reassign a `nodeId` or an identifier binding that has been deregistered. It SHOULD retain a tombstone record sufficient to prevent reuse and to answer "this identifier was registered and has been withdrawn".
4. Deregistration MUST NOT delete the audit record of the registration events themselves (§7).

### 4.10 Error conditions

*Provisional — the full error catalogue and its alignment with the base specification's `Error` enum belong to §10.* A conforming Registry is expected to distinguish at least:

| Condition | Meaning |
|---|---|
| `IdentifierNotResolvable` | A supplied LEI does not resolve, or is not in `ISSUED` status. |
| `IdentifierAlreadyRegistered` | The identifier is bound to an existing record; use the takeover procedure (§4.3). |
| `EndpointControlNotProven` | The proof-of-control challenge (§4.5.2) was not completed. |
| `CredentialVerificationFailed` | A presented credential could not be chained to the root of trust, or is revoked. |
| `OutOfScopeForRegistry` | The identifier lies outside this Registry's declared authoritative scope (§4.7). |
| `AuthorityNotDemonstrated` | The Registrant did not demonstrate authority to act for the Entity (§4.3). |

---

## Open items carried from this section

- Whether the interim manual-verification status (§4.6.2) belongs in normative text or an operational annex.
- The concrete proof-of-control challenge: newly minted well-known path vs. reuse of an existing PACT path (§4.5.2) — to be fixed in §10.
- Whether generic W3C VC/DID credentials are accepted alongside vLEI (§4.6.3).
- Whether registration and lifecycle events MUST be audit-logged; the base specification leaves logging out of scope (§5.3(d)). Carried to §7.
- Re-check intervals (§4.4.2, §4.6.3) are stated as recommendations; the WG should decide whether any are normative.
- The "invite a non-member supplier to join" journey remains informative/product, not normative.

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3 (§5.3 out of scope; §5.5 authentication).
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC7591] — OAuth 2.0 Dynamic Client Registration Protocol.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC8141] — Uniform Resource Names (URNs).
- [ISO17442] — Legal Entity Identifier (LEI).
- [VLEI-EGF] — vLEI Ecosystem Governance Framework, GLEIF.
