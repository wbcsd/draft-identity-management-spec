# PACT Identity Management Addendum — §6 Automated Credential Exchange (Draft)

*Draft v0.1 — 16 September 2026. First pass at normative text for the Automated Credential Exchange section of the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3. Builds on §3 (Identity Model), §4 (Registration) and §5 (Node Discoverability). Reflects decision D13 of 18 June 2026 (peer-to-peer after discovery; RFC 7591 as the RECOMMENDED binding) and the 2 September 2026 WG decisions (see the addendum outline, §12.1). Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** First pass for Technology Working Group discussion. **One design decision is deliberately left open and presented side by side:** how a target Node knows a connection request really comes from the party it names, and how the approval reaches that party (Option C, §6.4). Parameter names are provisional.

---

## 6. Automated Credential Exchange

### 6.1 Introduction

This section fills the gap the base specification declares out of scope in §5.3: how two host systems that have never exchanged data come to hold the OAuth 2.0 client credentials that the base specification's §5.5 token flow consumes, without a manual, bilateral handshake.

The exchange is strictly **peer-to-peer**. Neither the PACT Directory nor any Discovery Service takes part in it, or ever holds the resulting credentials (§4.1).

Terminology used in this section:

- **Requester** — the Node that wants to become an OAuth client of another Node, typically acting as [=data recipient=].
- **Target** — the Node that will issue credentials and serve footprints, typically acting as [=data owner=].
- **Connection** — the relationship, owned by the Target, that results from an approved request and to which exactly one set of provisioned credentials belongs.

The flow has four steps: **request** (§6.3), **approve** (§6.5), **provision** (§6.6), then **manage** (rotate, suspend, revoke; §6.7).

### 6.2 Preconditions

1. The Requester has resolved the Target through discovery (§5.4) or already knows the Target's `baseUrl` and `connectionEndpoint`.
2. The Target's `$base-url$/.well-known/openid-configuration` advertises a `registration_endpoint` [RFC8414]. A Target without one is not Credential-Exchange-capable, and the Requester MUST fall back to manual credential exchange.
3. A Node that sends connection requests MUST itself be served by an Operator listed in the PACT Directory. This lets the Target check the Requester against a verified Operator (§6.4).

### 6.3 Connection request

The Requester sends `POST {Target connectionEndpoint}` (§10.4) with:

| Field | Requirement | Notes |
|---|---|---|
| `requester.companyIds` | REQUIRED | Requester's identifiers per §3.4. |
| `requester.name` | REQUIRED | For display to the Target's approver. |
| `requester.nodeId` | REQUIRED | The Requester's Node as published by its Discovery Service. |
| `requester.serviceId` | REQUIRED | The listed Discovery Service of the Requester's Operator. |
| `target.companyId` | REQUIRED | The Target identifier that was resolved. |
| `message` | OPTIONAL | Free text for the approver, for example a purchase-order reference. |

1. The Target MUST respond `202 Accepted` with a `connectionId` and `status` `pending`, or with an error (§10.6).
2. `connectionId` MUST be generated with at least 128 bits of entropy and MUST NOT be guessable.
3. The Target MUST check that `requester.serviceId` refers to an `active` listing in the PACT Directory, and MUST reject the request with `RequesterNotListed` otherwise.
4. The request MUST NOT contain a callback URL, a redirect URL, or any other endpoint that the Target would use to reach the Requester. Endpoints for the Requester are only ever taken from its Operator's Discovery Service (§6.4).
5. A Target SHOULD rate-limit connection requests per `requester.serviceId`.

### 6.4 Open decision C — authenticating the Requester and delivering approval

A connection request arrives before any credentials exist, so it cannot be authenticated with OAuth. Without a further check, anyone could send a request that *names* a real company. The two options below close that gap in different ways.

#### Option C1 — Approval delivered to the Requester's published endpoint

The request itself is unsigned. Authenticity comes from where the approval is sent.

- C1-1. Before approving, the Target MUST resolve `requester.companyIds` through the Discovery Service named in `requester.serviceId` (§5.4, §10.3), and MUST confirm that the answer lists `requester.nodeId` with a `connectionEndpoint`. Otherwise it MUST reject the request with `RequesterNotResolvable`.
- C1-2. On a decision, the Target MUST send `POST {Requester connectionEndpoint}/{connectionId}/decision` (§10.4), using the `connectionEndpoint` obtained in C1-1 and never an endpoint supplied in the request. On approval, the body carries the initial access token (§6.6).
- C1-3. The Requester MUST reject, with `404`, a decision for a `connectionId` it did not itself create. It SHOULD log such events as a possible impersonation attempt.
- C1-4. The Requester MUST NOT use any endpoint contained in a decision message. It reads the Target's `registration_endpoint` from the Target's own well-known document (§6.2).

*Why it holds.* Someone impersonating a Requester can create a pending request, but the initial access token only ever travels to the endpoint the genuine Requester's Operator publishes. A forged decision sent to a Requester fails at C1-3 or when the token is used.

#### Option C2 — Signed request, Requester polls for the decision

- C2-1. The connection request MUST be signed with HTTP Message Signatures [RFC9421], covering at least `@method`, `@target-uri`, `content-digest` [RFC9530], and a `created` timestamp, with a key published at the `jwksUri` of the listing for `requester.serviceId` (§4.3.2).
- C2-2. The Target MUST verify the signature against that `jwksUri` and MUST reject the request with `InvalidSignature` if verification fails or the listing is not `active`.
- C2-3. The Requester obtains the decision by polling `GET {Target connectionEndpoint}/{connectionId}` (§10.4), with each poll signed as in C2-1. The Target MUST return `status` and, once approved, the initial access token until the token has been used or has expired.
- C2-4. The Target SHOULD return `Retry-After` on pending polls, and the Requester MUST respect it.
- C2-5. The Target MAY also resolve the Requester through discovery as in C1-1. It is not required to, because the signature already binds the request to an accountable, listed Operator.

*Why it holds.* Only the listed Operator holds the private key for its domain-verified `jwksUri`, so a request cannot be sent in another Operator's customer's name. The token is returned only to a correctly signed poll.

#### Trade-offs

| | C1 Approval to published endpoint | C2 Signed request + polling |
|---|---|---|
| **Cryptography for SPs** | None beyond TLS. | Key pair, JWKS publication, request signing and verification. |
| **Can someone send requests in another party's name?** | Yes, but they are useless: they cannot get the token. The Target's approver may still see nuisance requests. | No. |
| **Requester must be discoverable** | Yes. The Target has to resolve the Requester's `connectionEndpoint`, so a party that wants to connect but not be found cannot use this option. | No. Being served by a listed Operator is enough. |
| **Decision delivery** | Pushed as soon as the owner decides, even days later. | Polled. Manual approval may take days, so polling must back off. |
| **New Node methods** | Target: `POST connections`. Requester: `POST connections/{id}/decision`. Every Credential-Exchange-capable Node implements both. | Target: `POST connections`, `GET connections/{id}`. |
| **Reuse** | None. | Same keys as Option B1 of §5.6. Prepares for credential-bound signing (§3.7.4). |
| **Natural pairing** | Option B2 (public discovery, no keys anywhere). | Option B1 (listed and signed discovery). |

*Editor's note.* The pairings are not forced, but mixing them gives SPs key management *and* an extra callback endpoint. The WG may find it easier to decide B and C together.

### 6.5 Approval and connection policy

1. Every connection MUST be approved by, or on behalf of, the Target's [=data owner=]. Discoverability never implies approval (§5.8).
2. In this version, approval is expected to be a human decision. Policy-based automatic approval MAY be offered by an implementation, but its rules are not specified in this version.
3. The Target SHOULD show the approver: the Requester's name and `companyIds`; the assurance level with its attesting Service and evidence (§5.7), obtained by resolving the Requester through discovery where possible; the Requester's Operator; and any `message`.
4. Each party decides which minimum assurance level it accepts (§3.5.3). The working position for this version is that *Email-verified* or *Domain-verified* is sufficient **where both parties agree**. A Target MAY decline, with `AssuranceBelowPolicy`, a request below its minimum.
5. A declined request MUST move the connection to `declined`. Under Option C1 the decision is sent to the Requester; under Option C2 it is returned on the next poll.
6. A pending request that is not decided within a period set by the Target SHOULD expire. The Target MUST report `expired` to the Requester in the same way.

### 6.6 Credential provisioning

Approval yields a one-time **initial access token** in the sense of [RFC7591] §3, which the Requester spends at the Target's `registration_endpoint`.

1. The initial access token MUST be single-use, MUST be bound to exactly one `connectionId`, and MUST expire. An expiry of no more than 72 hours after approval is RECOMMENDED.
2. The Requester MUST send `POST {registration_endpoint}` with `Authorization: Bearer {initial access token}` and client metadata including at least:
   - `grant_types`: `["client_credentials"]`;
   - `token_endpoint_auth_method`: the method the Target's host system uses for the base-specification token flow, advertised in its well-known document;
   - `client_name`; and
   - `pact_connection_id`: the `connectionId`.
3. On success the Target MUST return, per [RFC7591] §3.2.1 and [RFC7592] §3: `client_id`, `client_secret`, `client_secret_expires_at`, `registration_access_token`, and `registration_client_uri`. The connection moves to `active`.
4. The Target MUST reject metadata requesting any grant type other than `client_credentials`.
5. The resulting `client_id` and `client_secret` are used in the base-specification §5.5 flow unchanged (§6.9).

*Editor's note.* Many off-the-shelf authorisation servers already implement RFC 7591/7592, so for such SPs the provisioning step is configuration rather than new code. `pact_connection_id` is a provisional client-metadata extension. It needs registering in the IANA OAuth Dynamic Client Registration Metadata registry or renaming.

### 6.7 Connection lifecycle

*States (see the flow document for the diagram):* `pending` → `approved` → `active`; `pending` → `declined` or `expired`; `approved` → `expired` (initial access token not used in time); `active` ↔ `suspended`; `active` or `suspended` → `revoked`.

#### 6.7.1 Rotation

1. The Requester MAY rotate its secret at any time by `PUT {registration_client_uri}` with its `registration_access_token` [RFC7592] §2.2. The Target MUST issue a new `client_secret` and SHOULD keep the previous secret valid for a short, stated overlap.
2. Where `client_secret_expires_at` is non-zero, the Requester MUST rotate before that time.

#### 6.7.2 Suspension

1. A Target MAY suspend a connection: the credentials are kept but token requests are refused. Reasons include the Requester's assurance falling below the Target's policy, or the Requester's Operator being suspended in the PACT Directory.
2. Suspension in the Directory, delisting, or withdrawing discoverability MUST NOT, by themselves, suspend or revoke a connection (§4.4.3). A Target that wants that consequence applies it through its own policy.

#### 6.7.3 Revocation

1. **Target-initiated.** The Target MUST invalidate the client registration and every access token issued under it, and move the connection to `revoked`. Subsequent token requests MUST fail.
2. **Requester-initiated.** The Requester ends the connection with `DELETE {registration_client_uri}` [RFC7592] §2.3. The Target MUST then act as in 1.
3. A revoked connection MUST NOT be reactivated. A new connection requires a new request.

*Editor's note.* How a Requester learns that the Target has revoked or suspended a connection, other than through failing token requests, is open. It is related to parked item P1 on notifications.

### 6.8 Exchange in both directions

A connection is one-directional: the Requester becomes an OAuth client of the Target. Where two parties exchange in both directions, each runs the flow as Requester once, independently. The supplier/customer direction is a property of the connection, not of either identity (§3.6).

### 6.9 Relationship to base-specification authentication (§5.5)

This section **produces** the `client_id` and `client_secret` that base-specification §5.5 **consumes**. The token endpoint, the client-credentials grant, the bearer token, and the four core actions are unchanged. A host system that has received credentials manually and one that received them through this section behave identically at exchange time.

---

## Open items carried from this section

- **Option C** — request authentication and decision delivery: approval to published endpoint, or signed request and polling (§6.4). Best decided together with Option B (§5.6).
- Policy-based automatic approval rules (§6.5).
- Notifying the counterparty of suspension or revocation (§6.7.3).
- Registration or renaming of the `pact_connection_id` client-metadata field (§6.6).
- Presenting a verifiable credential at connection time (§3.7.4), as an optional addition to either option.
- Whether connection lifecycle events MUST be audit-logged (§7).

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3 (§5.3 out of scope; §5.5 authentication).
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC6749] — The OAuth 2.0 Authorization Framework.
- [RFC7591] — OAuth 2.0 Dynamic Client Registration Protocol.
- [RFC7592] — OAuth 2.0 Dynamic Client Registration Management Protocol.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC9421] — HTTP Message Signatures.
- [RFC9530] — Digest Fields.
