# PACT Identity Management Addendum — §5 Node Discoverability (Draft)

*Draft v0.1 — 16 September 2026. First draft of normative text for the Node Discoverability section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Builds on §3 (Identity Model) and §4 (Registration), and implements the decision of the Technology Working Group session of 2 September 2026 that PACT runs a central directory of solution-provider discovery endpoints while buyer and supplier data stays with each provider (W1; see the addendum outline, §12.1). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review. **Two design decisions are deliberately left open and presented side by side** for the WG: how the query fans out across Discovery Services (Option A, §5.5), and who may send discovery queries (Option B, §5.6). Normative text that depends on the choice is marked with the option it belongs to.

---

## 5. Node Discoverability

### 5.1 Introduction

Discovery answers one question for a party that wants to exchange PCF data with an organisation it has not exchanged with before:

> *Given one of this organisation's `companyIds`, which Node(s) can I send a connection request to, who vouches for that answer, and how strongly?*

The target case is **cross-platform**: the requester and the target are served by different Operators and do not already know each other's endpoints. Where two parties already know each other's endpoints, discovery is not needed and nothing in this section applies.

Discovery never grants access. A discovery answer lets a party *request* a connection. Whether the connection is made is decided by the target (§6).

### 5.2 Why a directory of Discovery Services rather than DNS

*Non-normative rationale, recorded as a design decision at the request of the WG.*

A WG participant asked whether DNS could do the job: supplier domains and email addresses are often known already, and a DNS record could point at a PACT endpoint. The 2 September session chose a directory of Discovery Services instead, for these reasons:

1. **PCF exchange is keyed on `companyIds`, not domains.** Footprints carry `companyIds`, and the domain a buyer knows (for example from a contact's email address) often says nothing about where the supplier's PCF tooling lives.
2. **Most suppliers are hosted by a solution provider.** A DNS-based design needs every supplier's IT team to publish records, and to change them whenever the supplier changes provider. For thousands of suppliers that is exactly the registration burden the WG wants to avoid. With a directory of Discovery Services, the provider publishes once for all its customers.
3. **DNS offers no opt-in short of "public", and no gate.** A DNS record is visible to anyone and cannot be withheld from non-conformant or unknown callers. Conformance-gated listing (W2) and suspension (§4.4.2) have no equivalent in DNS.
4. **DNS would publish each supplier's choice of provider to the world.** That feeds the solution-provider concern about competitors and disintermediation.

A domain-keyed lookup, for example a DNS record under the supplier's domain that points at its Discovery Service, remains a possible *complement* in a later version. It would add a lookup key without changing the model.

### 5.3 Architecture

Discovery involves three roles and no central index of parties:

```
             ┌───────────────────────────────┐
             │ PACT Directory                │  lists Discovery Services only (§4)
             └──────────────┬────────────────┘
                            │ 1. list active Discovery Services
                            ▼
┌──────────────┐  2. query by companyId   ┌──────────────────────────────┐
│ Requester    │ ───────────────────────► │ Discovery Service (per SP)   │
│ (Node / SP)  │ ◄─────────────────────── │ answers for its own customers│
└──────┬───────┘  3. Node(s) + assurance  └──────────────────────────────┘
       │ 4. GET $base-url$/.well-known/openid-configuration
       ▼
┌──────────────┐
│ Target Node  │  token_endpoint, registration_endpoint (§4.5)
└──────────────┘
```

Under Option A2 (§5.5), step 2 is performed by the PACT Directory on the requester's behalf.

### 5.4 Resolution procedure

To resolve a `companyIds` value, a requester MUST:

1. Obtain the current list of Discovery Services with `status` `active` from the PACT Directory (§10.2), and MUST NOT query a Service whose listing is `suspended` or withdrawn.
2. Query **every** active Discovery Service for the `companyIds` value, as defined by the chosen fan-out option (§5.5), using the interface of §10.3.
3. Treat a `404` from a Service as "this Service does not publish that identifier", without distinguishing unknown from not visible (§5.8).
4. Collect all positive answers. An Entity MAY legitimately be served by more than one Operator (§3.2), so a requester MUST NOT assume that at most one Service answers.
5. For each Node it intends to contact, read `token_endpoint` and `registration_endpoint` from the Node's `$base-url$/.well-known/openid-configuration`, and not from the discovery answer (§4.5).

Where answers conflict, for example two unrelated Operators both claim to serve the same `companyIds` value with different Nodes:

6. A requester MUST NOT silently discard or merge answers. It SHOULD present them with their attesting Service and assurance (§5.7), and MAY apply a policy that prefers the answer with higher assurance.
7. A requester SHOULD report a suspected false answer to PACT under the procedure of §4.4.2.

*Editor's note.* The requester queries by one identifier at a time. A requester that holds several `companyIds` for the same target MAY resolve each. Batch queries are left out of this version to keep the interface to one simple method.

### 5.5 Open decision A — who performs the fan-out

#### Option A1 — Client-side fan-out

The PACT Directory only publishes the list of Discovery Services. The requester (its Node or its Operator on its behalf) sends the query to every active Service in parallel and aggregates the answers.

- A1-1. The requester MUST perform steps 2–4 of §5.4 itself.
- A1-2. The requester SHOULD query Services in parallel and SHOULD apply a per-Service timeout. A Service that does not answer within the timeout is treated as having returned `404`.

#### Option A2 — Directory-brokered fan-out

The requester sends one query to the PACT Directory, which forwards it to every active Discovery Service and returns the aggregated answers.

- A2-1. The Directory MUST expose a resolve interface (§10.2, `GET /v1/resolve`) and MUST return every positive answer unmodified, each labelled with its attesting `serviceId`.
- A2-2. The Directory MUST NOT store, cache, or log discovery answers, or the identifiers queried, beyond what is needed to complete the request and to operate the service securely. Otherwise the Directory would become the central index of parties that W1 rules out.
- A2-3. The Directory MUST NOT forward the requester's identity to Discovery Services.
- A2-4. The Directory is not in the credential or data path. It is in the **discovery** path only.

#### Trade-offs

| | A1 Client-side | A2 Directory-brokered |
|---|---|---|
| **Who learns about a lookup** | Every listed Discovery Service learns that *this requester* (under B1) or *someone* (under B2) looked up the identifier. Competing SPs learn which suppliers another SP's customers are looking for. | Only PACT learns who looked up what. Discovery Services see the Directory as the caller. |
| **PACT's role** | Publishes a list. Not in any request path. | In the discovery request path. Must be operated and trusted not to retain query data (A2-2). |
| **Availability** | Discovery works while the listing is cached, even if the Directory is down. | Discovery stops when the Directory is down. Established connections are unaffected either way (§4.6). |
| **Requester implementation** | Fan-out, timeouts, and aggregation in every requester. | One HTTP call. |
| **Discovery Service implementation** | One method (§10.3), called by many requesters. | One method (§10.3), called only by the Directory. Easier rate limiting and authentication (only one caller). |
| **Scale** | N parallel calls per lookup. With about 50 SPs and lookups mostly when a new supplier is onboarded, this is modest; it grows linearly with the number of SPs. | One call for the requester; N calls for the Directory. PACT carries the load. |
| **Conformance testing** | Aggregation behaviour of requesters is hard to test. | Aggregation is PACT's own code; only the Discovery Service interface needs testing. |
| **New PACT-side methods** | `GET /v1/discovery-services` | `GET /v1/discovery-services`, `GET /v1/resolve` |
| **New SP-side methods** | 1 | 1 |

### 5.6 Open decision B — who may query a Discovery Service

#### Option B1 — Listed Operators only, with signed requests

- B1-1. A discovery query MUST be signed with HTTP Message Signatures [RFC9421], covering at least `@method`, `@target-uri`, and a `created` timestamp, using a key published at the caller's `jwksUri` as listed in the PACT Directory (§4.3.2).
- B1-2. The request MUST identify the caller's `serviceId`. A Discovery Service MUST verify the signature against that listing's `jwksUri`, MUST check that the listing is `active`, and MUST reject the request with `401` otherwise.
- B1-3. Under Option A2, the caller is the PACT Directory, which is listed with its own `jwksUri` for this purpose.

#### Option B2 — Public queries with a minimal answer

- B2-1. A Discovery Service MUST accept unauthenticated discovery queries.
- B2-2. The answer MUST be limited to the fields marked *minimal* in §5.7.
- B2-3. A Discovery Service SHOULD rate-limit queries per client and MAY return `429`.

#### Trade-offs

| | B1 Listed & signed | B2 Public |
|---|---|---|
| **Harvesting risk** | Only accountable, listed Operators can query, and each query is attributable. Bulk harvesting is detectable and can lead to suspension. | Anyone can take the openly downloadable list of LEIs (several million) and map which SP serves which company. That exposes each SP's customer base to competitors. Rate limiting slows this but cannot stop it. |
| **Who can discover** | Only parties served by a listed Operator, including SPs-of-one. A self-hosted Node that is not listed cannot discover others. | Anyone, including parties that are not yet listed and testers. |
| **Implementation cost** | Key pair, JWKS publication, and signature verification for every Operator. Libraries exist, but it is new work for most SPs. | None beyond the endpoint. |
| **Reuse** | The same keys serve Option C2 of §6.4 (signed connection requests) and prepare the ground for credential-based trust (W3). | No reuse. |
| **Answer detail** | Full answer (§5.7) can be returned, because the caller is known. | Minimal answer only. |
| **Relationship to parked item P3** | Discovery restricted to conformant, listed Operators. | Discovery public. |

*Editor's note.* The **Directory listing itself** (which SPs exist and where their Discovery Services are) is low-sensitivity and largely public already through PACT's list of conformant solutions. The draft therefore assumes the listing is publicly readable under both options. Parked item P3 is mainly about *discovery queries*, which is Option B.

### 5.7 Discovery answer

A positive discovery answer describes one Entity and the Node(s) through which it can be contacted. Fields marked **minimal** are the only ones returned under Option B2.

| Field | Minimal | Requirement | Notes |
|---|---|---|---|
| `companyIds` | ✓ | REQUIRED | The Entity's identifiers the Service publishes. MUST include the value queried. |
| `legalName` |  | OPTIONAL | As known to the Operator. |
| `attestedBy` | ✓ | REQUIRED | `serviceId` of the answering Discovery Service. |
| `assurance.level` | ✓ | REQUIRED | One of the levels of §3.5.1. |
| `assurance.evidence[]` |  | REQUIRED | Per method: `method` (`email`, `domain`, `lei-status`, `vc`), `verifiedAt`, and for `domain` and `email` the verified `domain`; for `lei-status` the LEI; for `vc` the credential `profile`. |
| `nodes[].nodeId` | ✓ | REQUIRED | Stable within the Service. |
| `nodes[].baseUrl` | ✓ | REQUIRED | HTTPS `$base-url$` of the host system. |
| `nodes[].connectionEndpoint` | ✓ | CONDITIONAL | REQUIRED where the Node is Credential-Exchange-capable (§6). |
| `nodes[].conformantVersions` |  | REQUIRED | Base-specification versions, sourced from the PACT Conformance Service. MAY be empty. |
| `nodes[].provisional` | ✓ | REQUIRED | `true` for a demo on-ramp Node (§4.5). |

1. A Discovery Service MUST NOT include `token_endpoint`, `registration_endpoint`, contact details, or information about the Entity's commercial relationships.
2. The published assurance MUST satisfy §3.5.2.

*Example (non-normative, full answer):*

```json
{
  "companyIds": ["urn:lei:894500ABCDEF12345678", "urn:pfi:sp-example.com:customer:10442"],
  "legalName": "Example Supplier Ltd",
  "attestedBy": "urn:pact:discovery-service:7d0c6b8e-2f1a-4b8e-9a51-0c3e1d5a9f10",
  "assurance": {
    "level": "registry-verified",
    "evidence": [
      { "method": "domain", "domain": "example-supplier.co.uk", "verifiedAt": "2026-07-02T09:14:00Z" },
      { "method": "lei-status", "lei": "894500ABCDEF12345678", "verifiedAt": "2026-09-01T03:00:00Z" }
    ]
  },
  "nodes": [
    {
      "nodeId": "node-10442-eu",
      "baseUrl": "https://pact.sp-example.com/tenants/10442",
      "connectionEndpoint": "https://pact.sp-example.com/tenants/10442/3/connections",
      "conformantVersions": ["3.0"],
      "provisional": false
    }
  ]
}
```

### 5.8 Visibility

1. A Discovery Service MUST publish an Entity only where that Entity has opted in to being discoverable. How the Operator collects the opt-in is Operator-internal (§4.5).
2. A Discovery Service MUST return the same `404` response for an identifier it does not serve and for one it serves but does not publish, so that absence reveals nothing.
3. An Entity MUST be able to withdraw its opt-in through its Operator. The Service MUST stop publishing the Entity once the withdrawal is processed.
4. Search or browse by name ("listability" in the 18 June design) is not part of this version. Discovery is by identifier only.
5. Being discoverable never means being connected. Every connection requires the target's approval (§6).

*Editor's note.* The 18 June design (D12) had resolvability default **on** for authenticated members. This draft makes discoverability an explicit **opt-in**, in line with the WG's stated outcome of *opt-in visibility, promoted but never mandated*, and because under Option B2 there are no "authenticated members" to scope a default to. An Operator may still make opting in the easy default in its own onboarding flow. The WG should confirm.

### 5.9 Caching and freshness

1. A requester, or the Directory under Option A2, SHOULD cache the Directory listing according to its HTTP caching headers, and MUST refresh it at least every 24 hours so that suspensions (§4.4.2) take effect within that bound.
2. A requester MAY cache discovery answers according to their HTTP caching headers, and SHOULD resolve again immediately before sending a connection request (§6).
3. How Operators and parties are told about Directory changes and new versions of the discovery interface is parked item P1 and is not specified in this version.

---

## Open items carried from this section

- **Option A** — fan-out: client-side or Directory-brokered (§5.5).
- **Option B** — query authorisation: listed and signed, or public with minimal answer (§5.6). Largely settles parked item P3.
- Visibility default changed from D12 to explicit opt-in (§5.8), for WG confirmation.
- Whether a domain-keyed lookup should complement `companyIds` in a later version (§5.2).
- Batch discovery queries (§5.4 editor's note).
- Notification of Directory changes and versioning (parked item P1).

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC9110] — HTTP Semantics.
- [RFC9111] — HTTP Caching.
- [RFC9421] — HTTP Message Signatures.
