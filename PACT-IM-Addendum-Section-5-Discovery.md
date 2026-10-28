# PACT Identity Management Addendum — §5 Node Discoverability (Draft)

*Draft v0.3 — 28 October 2026. Normative text for the Node Discoverability section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Builds on §3 (Identity Model) and §4 (Registration). Implements the decisions of the Technology Working Group session of **30 September 2026**: PACT operates a **central discovery service** that fans a query out to the discovery endpoints of registered Solution Providers, and the service is **private** — only registered Solution Providers may query it (see the addendum outline, §12.4). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Working draft for Technology Working Group review.
>
> **Changes in v0.3 (versioning proposal, 28 October 2026 — for WG confirmation).**
> - §5.9 defines **IM interface versions**: `MAJOR.MINOR`, the major in the path (`/im/{major}/…`), the full version in a `PACT-IM-Version` header; providers declare the majors they support and the service calls each at the highest common one.
> - §5.10 defines **how providers are told** about changes: a service-info document, `Deprecation`/`Sunset` headers, and mail to the registered contact.
> - §5.11 settles **changes to discovery data**: not versioned, not pushed — re-resolve before connecting.
> - The figures for notice period and number of supported majors are **left to the WG**.
>
> **Changes in v0.2 (following the 30 September WG session).**
> - The two options of v0.1 are gone. **Central fan-out is the design** (W5) and the service is **private** (W6). Both options and the reasoning behind the choice are recorded in the decision log (§12.4), not here.
> - A Solution Provider implements **one** discovery endpoint, called by the PACT discovery service rather than by every other provider.
> - Discoverability is **opt-in, held and managed by the provider** (W7); by registering, a provider **vouches** for the customers that opted in (W8).
> - New §5.9 on **version changes to the service**, the item the WG asked for on 30 September (W10).
> - **Two decisions remain open** and are presented side by side: how a provider authenticates **to** the service (§5.5), and how a provider's endpoint authenticates calls **from** it (§5.6).

---

## 5. Node Discoverability

### 5.1 Introduction

Discovery answers one question for a party that wants to exchange PCF data with an organisation it has not exchanged with before:

> *Given one of this organisation's `companyIds`, which Node(s) can I send a connection request to, and who vouches for that answer?*

The target case is **cross-platform**: the two parties are served by different Solution Providers and do not know each other's endpoints. **Discovery within a provider's own platform is out of scope** — a provider already knows its own customers and handles that itself.

Discovery never grants access. A discovery answer lets a party *request* a connection; whether the connection is made is decided by the target (§6).

### 5.2 Why a directory rather than DNS

*Non-normative rationale, recorded at the request of the WG.*

DNS was considered: supplier domains are often known already, and a DNS record could point at a PACT endpoint. The directory was chosen because:

1. **PCF exchange is keyed on `companyIds`, not domains.** The domain a buyer knows, typically from a contact's email address, often says nothing about where the supplier's PCF tooling lives.
2. **Most suppliers are hosted by a Solution Provider.** A DNS design needs every supplier's IT team to publish and maintain records. With a provider-level directory, the provider publishes once for all its customers.
3. **DNS offers no opt-in short of "public", and no gate.** A record is visible to everyone and cannot be withheld from unknown callers; conformance-gated listing (§4.3) and suspension (§4.4.2) have no DNS equivalent.
4. **DNS would publish each supplier's choice of provider to the world**, which is precisely the exposure the private model (§5.5) avoids.

A domain-keyed lookup remains possible as a *complement* in a later version.

### 5.3 Architecture

```
┌──────────────────────┐   1. resolve(companyId)    ┌──────────────────────────────┐
│ Requesting Solution  │ ─────────────────────────► │ PACT Discovery Service        │
│ Provider (on behalf  │ ◄───────────────────────── │ · registry of SP endpoints §4 │
│ of its customer)     │   4. aggregated answers    │ · fan-out, retains nothing    │
└──────────────────────┘                            └───────┬───────┬───────┬──────┘
                                                            │ 2. query each registered
                                                            ▼       ▼       ▼
                                                   ┌────────────┐ ┌─────┐ ┌─────┐
                                                   │ SP-S disc. │ │SP-X │ │SP-Y │
                                                   │ endpoint   │ │ 404 │ │ 404 │
                                                   └─────┬──────┘ └─────┘ └─────┘
                                                         │ 3. match: node metadata
                                                         ▼
                                              ┌──────────────────────┐
┌──────────────────────┐  5. GET $base-url$/.well-known/openid-configuration
│ Requester  ─────────────────────────────────►│ Target Node          │
└──────────────────────┘                       └──────────────────────┘
```

1. PACT MUST operate a single **PACT Discovery Service**, which holds the registry of Solution Provider discovery endpoints (§4) and performs the fan-out.
2. A Solution Provider MUST implement exactly one **discovery endpoint**, which answers for the Entities it serves (§5.7).
3. The Discovery Service MUST NOT hold, index or build a register of the Entities that providers answer for, and MUST NOT store, cache or log query identifiers or answers beyond what is needed to serve the request and to operate the service securely.
4. Neither the Discovery Service nor any provider's discovery endpoint is in the credential or data path. Once a connection exists (§6), exchange MUST NOT depend on either being available.

### 5.4 Resolution procedure

1. A requester resolves an identifier by sending **one** query to the Discovery Service (§10.2), carrying a single `companyIds` value in canonical form (§3.4).
2. The Discovery Service MUST query every registered discovery endpoint whose listing status is `active`, and MUST NOT query a suspended or withdrawn listing.
3. The Discovery Service SHOULD query endpoints in parallel and SHOULD apply a per-endpoint timeout; an endpoint that does not answer in time is treated as returning no match, and the response MUST say so rather than omitting it silently.
4. The Discovery Service MUST return every positive answer it received, each labelled with the provider that gave it. An Entity MAY be served by more than one provider, so a requester MUST NOT assume at most one answer.
5. Where answers conflict — two unrelated providers claiming the same identifier with different Nodes — the requester MUST NOT silently discard or merge them, SHOULD present them with the vouching provider, and SHOULD report a suspected false answer to PACT (§4.4.2).
6. For each Node it intends to contact, the requester reads `token_endpoint` and `registration_endpoint` from that Node's own `$base-url$/.well-known/openid-configuration`, never from the discovery answer (§4.5).

*Editor's note.* One identifier per query. Batch queries are left out of this version to keep the interface small; the WG can add them once traffic patterns are known.

### 5.5 Access to the Discovery Service — private (W6)

1. The Discovery Service MUST accept queries **only from registered Solution Providers** (including an SP-of-one, §4.3.1). It MUST NOT offer anonymous or public lookup in this version.
2. A provider queries on behalf of its own customers; the Discovery Service MUST NOT require the identity of the customer on whose behalf a query is made, and a provider SHOULD NOT supply it.
3. Opening the service to a wider set of callers is a future change, subject to §5.9: a public read tier would return a reduced answer and is not defined here.

#### Open decision 1 — how a provider authenticates to the service

*The WG has not settled this. Both options are drafted; the choice affects §10 only.*

| | **1a — Credentials issued at registration** | **1b — Signed requests** |
|---|---|---|
| Mechanism | The service issues OAuth 2.0 client credentials when the provider is registered; the provider presents a bearer token on each query | The provider signs each query (HTTP Message Signatures [RFC9421]) with a key published at a `jwks_uri` under a domain verified in §4.3.3 |
| Provider effort | Store a secret, fetch a token — the same pattern providers already implement for the base specification | Key pair, JWKS publication, signing |
| Consistency with what exists | Matches the token route the PACT identity management service already serves (§10.5) | New infrastructure, but no shared secret to leak and reusable for §6 |
| Revocation | Revoke the client at suspension | Remove the listing; keys stay valid until the listing is checked |

### 5.6 Calls to a provider's discovery endpoint

1. A provider's discovery endpoint MUST answer queries from the PACT Discovery Service, and MUST authenticate that caller.
2. It MUST NOT accept unauthenticated queries from any source.

#### Open decision 2 — how the endpoint authenticates the service, and whether direct calls are allowed

| | **2a — Service only** | **2b — Service, plus direct provider-to-provider as fallback** |
|---|---|---|
| Who may call | The PACT Discovery Service and nobody else | The service, and any registered provider calling directly |
| Provider effort | One caller to authenticate; simplest to build and to conformance-test | Must authenticate many callers and check each is registered |
| Availability | Discovery is unavailable while the service is down; established connections are unaffected (§5.3, requirement 4) | A provider can fall back to querying endpoints directly from the registry listing |
| Exposure | Every queried provider sees the identifier; central fan-out can hide the requesting provider's identity | Every queried provider sees both the identifier and the authenticated requesting provider |

*Editor's note.* The two open decisions interact: 1b and 2b share one signing mechanism, while 1a and 2a together are the smallest thing a provider has to build. The WG may find them easier to decide as a pair.

### 5.7 Discovery answer

A positive answer describes one Entity and the Node(s) through which it can be contacted.

| Field | Requirement | Notes |
|---|---|---|
| `companyIds` | REQUIRED | The identifiers the provider publishes for the Entity, in canonical form (§3.4). MUST include the value queried. |
| `legalName` | OPTIONAL | As known to the provider. |
| `vouchedBy` | REQUIRED | The registered provider answering, and therefore vouching (§3.5, §4.5). |
| `assurance.level` | OPTIONAL | One of the levels of §3.5.1, where the provider publishes one. Absence means the provider's vouching alone, and MUST NOT be read as a negative statement. |
| `assurance.evidence[]` | CONDITIONAL | REQUIRED where a level is published: per method (`email`, `domain`, `lei-status`, `vc`), the time it was last confirmed, and the verified domain, LEI or credential profile (§3.5.2). |
| `nodes[].nodeId` | REQUIRED | Stable within the provider. |
| `nodes[].baseUrl` | REQUIRED | HTTPS `$base-url$` of the host system. |
| `nodes[].interfaceVersions` | CONDITIONAL | Major versions of the IM connection interface the Node supports (§5.9.1). REQUIRED where the Node is Credential-Exchange-capable (§6); its connection endpoint is then `{baseUrl}/im/{major}/connections`. |
| `nodes[].conformantVersions` | REQUIRED | Base-specification versions, from the PACT Conformance Service. MAY be empty. |
| `nodes[].provisional` | REQUIRED | `true` for a demo on-ramp Node (§4.5). |

1. A discovery answer MUST NOT include `token_endpoint`, `registration_endpoint`, contact details, or anything about the Entity's commercial relationships.
2. A provider MUST answer only for Entities it serves and that have opted in (§5.8).

*Example (non-normative):*

```json
{
  "companyIds": ["urn:lei:894500ABCDEF12345678", "urn:pfi:sp-example.com:customer:10442"],
  "legalName": "Example Supplier Ltd",
  "vouchedBy": "urn:pact:discovery-service:7d0c6b8e-2f1a-4b8e-9a51-0c3e1d5a9f10",
  "assurance": {
    "level": "domain-verified",
    "evidence": [
      { "method": "domain", "domain": "example-supplier.co.uk", "verifiedAt": "2026-07-02T09:14:00Z" }
    ]
  },
  "nodes": [
    {
      "nodeId": "node-10442-eu",
      "baseUrl": "https://pact.sp-example.com/tenants/10442",
      "interfaceVersions": ["1"],
      "conformantVersions": ["3.0"],
      "provisional": false
    }
  ]
}
```

### 5.8 Visibility (W7)

1. An Entity is discoverable only where it has **opted in**. The opt-in is held and managed by the Solution Provider that serves it; how the provider collects and records it is not specified.
2. A provider MUST return the same "no match" result for an identifier it does not serve and one it serves but that has not opted in, so that absence reveals nothing.
3. An Entity MUST be able to withdraw its opt-in through its provider, and the provider MUST stop publishing it once the withdrawal is processed.
4. Search or browse by name is not part of this version; discovery is by identifier only.
5. Being discoverable is never being connected: every connection requires the target's approval (§6).

### 5.9 Interface versions (W10)

1. The interfaces defined by this addendum — the Discovery Service interface (§10.2), the provider discovery endpoint (§10.3) and the Node connection interface (§10.4) — are versioned together as the **IM interface**, with a version of the form `MAJOR.MINOR`.
2. A **minor** version adds only backwards-compatible changes: new optional fields, new optional query parameters, new error codes. A **major** version is any change that can break a conforming implementation of the previous major version.
3. The major version is carried in the **path**: every IM interface path has the form `/im/{major}/…` (§10). The full version is carried in the `PACT-IM-Version` response header on every response.
4. Implementations MUST ignore fields they do not recognise, so that a minor version can be adopted without coordination.
5. The IM interface is versioned **independently of the base specification**. A change to the base specification's data model or actions does not by itself change the IM interface version, and a change to the IM interface does not change the base specification's.

#### 5.9.1 Coexistence of versions

1. A provider MUST declare in its registry listing the major versions its discovery endpoint supports (`interfaceVersions`, §4.3.2). The declaration is the provider's own; it is not separately tested per version.
2. The Discovery Service MUST call each provider at the highest major version that both the service and the provider support, and MUST NOT call a provider at a major version it has not declared.
3. A requester MAY call the Discovery Service at any major version the service supports. The service MUST translate between versions when fanning out, so that a requester and a target provider need not support the same major version.
4. A discovery answer lists, per Node, the major versions of the connection interface that Node supports (`nodes[].interfaceVersions`, §5.7). A Requester MUST send a connection request at a major version the Target declares (§6.3).

#### 5.9.2 Breaking changes

1. PACT MUST announce a new major version at least **[notice period — WG to decide]** before the previous major version is withdrawn.
2. The Discovery Service MUST support at least **[number of major versions, e.g. current and previous — WG to decide]** at any time.
3. A provider MUST NOT be removed from the registry for not supporting a new major version before the previous one is withdrawn (§4.3.4).
4. Once a major version is withdrawn, the service stops calling providers that declare only that version. Such a provider is not suspended, but its customers are not discoverable until it declares a supported version.

### 5.10 Notification of changes

1. The Discovery Service MUST publish a machine-readable **service-info** document (§10.2.3) listing the IM interface versions it supports and, for each deprecated version, the date it was deprecated and the date it will be withdrawn, with a link to human-readable release notes.
2. On every response served under a deprecated version, the service MUST include a `Deprecation` header [RFC9745] and, once a withdrawal date is fixed, a `Sunset` header [RFC8594], each with a `Link` to the release notes.
3. PACT MUST notify each listed provider by email, at its registered `contact` (§4.3.2), of:
   - changes to **its own listing**: suspension, reinstatement, a domain re-verification coming due, and repeated failed liveness checks (§4.4);
   - a new **IM interface version**, a deprecation, and an approaching withdrawal date (§5.9.2);
   - any change to what it needs in order to **call the service**, such as credential or key rotation (§5.5).
4. Changes to other providers' listings are not notified. Under central fan-out a provider never calls another provider directly, so it has no use for them.
5. A provider MUST keep its `contact` current (§4.4.1).

*Editor's note.* No change feed or subscription is defined in this version. The service-info document and the standard headers make changes visible to software; the mail makes them visible to people. A feed can be added later as a minor version.

### 5.11 Changes to discovery data

1. Discovery answers are **not versioned and changes to them are not pushed**. What a provider publishes — which Nodes serve an Entity, which identifiers it carries, whether it has opted in — is read fresh each time it is resolved.
2. A provider SHOULD send HTTP caching headers [RFC9111] with each answer, and the Discovery Service MUST pass them on unchanged.
3. A requester MAY reuse an answer within its caching lifetime, but MUST resolve again immediately before sending a connection request (§6), and SHOULD resolve again when a connection or token request fails in a way that suggests the Node has moved.
4. When an Entity withdraws its opt-in or leaves its provider, later resolutions return no match (§5.8). Connections already established are not affected (§4.4.3); ending them belongs to the connection lifecycle (§6.7).

### 5.12 Summary figure

*Non-normative. The detailed sequence, including registration and credential exchange, is in the flow document.*

```mermaid
sequenceDiagram
    participant BSP as Buyer's solution provider
    participant DIR as PACT Discovery Service
    participant SSP as Supplier's solution provider
    SSP->>DIR: 1. Register discovery endpoint (domain-verified)
    BSP->>DIR: 2. Resolve supplier companyId
    DIR->>SSP: 3. Query discovery endpoint (and every other active endpoint)
    SSP-->>DIR: Discovery answer, vouched by supplier's provider
    DIR-->>BSP: Aggregated answers
    BSP->>SSP: 4. Request connection
    SSP-->>BSP: 5. Approve, credentials issued (RFC 7591)
    BSP->>SSP: 6. Exchange PCFs over the V3 API
    Note over BSP,SSP: PACT is never in the credential or data path
```

---

## Open items carried from this section

- **Open decision 1** — how a provider authenticates to the Discovery Service (§5.5).
- **Open decision 2** — how a provider's endpoint authenticates the service, and whether direct provider-to-provider queries are allowed as a fallback (§5.6).
- Accepted identifier schemes (§3.4) — **open pending the Taiwan UBN proof of concept on 28 October**; the mechanics of naming and matching are specified, the list is not.
- Notice period before a major version is withdrawn, and the number of major versions supported at once (§5.9.2) — **figures for the WG**.
- Whether a change feed or subscriptions are wanted on top of service-info, headers and mail (§5.10).
- Batch queries (§5.4).

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC6749] — The OAuth 2.0 Authorization Framework.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC9110] — HTTP Semantics.
- [RFC9111] — HTTP Caching.
- [RFC9421] — HTTP Message Signatures.
- [RFC8594] — The Sunset HTTP Header Field.
- [RFC9745] — The Deprecation HTTP Response Header Field.
