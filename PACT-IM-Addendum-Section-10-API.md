# PACT Identity Management Addendum — §10 API Specification (Draft)

*Draft v0.1 — 16 September 2026. First draft of the interface additions for the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3, for discussion at the Technology Working Group. Defines the HTTP interfaces behind §4 (Registration), §5 (Node Discoverability) and §6 (Automated Credential Exchange), written from a solution-provider implementation perspective. Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Draft endpoints for discussion; paths, parameter names and error codes are provisional. Discovery is now a single design — one PACT-operated service that fans a query out (W5), private to registered providers (W6). What remains open is how callers authenticate (open decisions 1 and 2 of §5) and how a connection request is authenticated (option C of §6.4); operations and headers that depend on those are marked. The design guardrail from the WG is that implementing the addendum should require **only a handful of new API methods**. §10.1 counts them.

---

## 10. API Specification

### 10.1 Overview and method count

The addendum adds interfaces in three places. Only the second and third are implemented by solution providers.

| Where | Operation | Implemented by | Status | PACT-specific? |
|---|---|---|---|---|
| PACT Discovery Service | `GET /v1/resolve?companyId=` | PACT | normative | yes |
| PACT Discovery Service | `GET /v1/discovery-services` | PACT | normative — registry listing, readable by registered providers | yes |
| PACT Discovery Service | `GET /v1/discovery-services/{serviceId}` | PACT | normative | yes |
| PACT-operated services | `/.well-known/openid-configuration` and the token endpoint it names | PACT | normative (§10.5.1) | no — [RFC8414]/[RFC6749] |
| Solution Provider | `GET /3/discovery?companyId=` | **SP** | normative — called by the Discovery Service | **yes** |
| Target Node | `POST /3/connections` | **SP** | normative | **yes** |
| Target Node | `GET /3/connections/{connectionId}` | **SP** | option C2 only | **yes** |
| Requester Node | `POST /3/connections/{connectionId}/decision` | **SP** | option C1 only | **yes** |
| Target Node | `POST {registration_endpoint}` | SP's authorisation server | normative | no — [RFC7591] |
| Target Node | `GET`/`PUT`/`DELETE {registration_client_uri}` | SP's authorisation server | normative | no — [RFC7592] |
| Solution Provider | `jwks_uri` publication | SP | only if signing is chosen (§5.5, §5.6, §6.4) | no — [RFC7517] |

**Result against the guardrail.** A Solution Provider implements **three new PACT-specific methods**: the discovery endpoint, the connection request, and either the decision callback (C1) or the status poll (C2). Central fan-out removes the client-side aggregation that the earlier draft put on every provider. Everything else is standard OAuth 2.0 that many authorisation servers already provide.

| Open decision | Effect on the SP-side API |
|---|---|
| **1** — how a provider authenticates to the Discovery Service (§5.5) | Credentials: a token request before each query. Signing: `Signature`/`Signature-Input` headers and a published `jwks_uri` |
| **2** — how a provider's endpoint authenticates the service (§5.6) | Service only: verify one caller. Direct fallback as well: verify any registered provider, and accept queries from them |
| **C** — connection request authentication (§6.4) | C1 adds `POST /3/connections/{id}/decision` on the requester; C2 adds `GET /3/connections/{id}` on the target |

Every combination still costs a provider three PACT-specific methods.

### 10.2 PACT Directory interface

*Implemented by PACT. Base URL `$directory-url$` (to be assigned).*

#### 10.2.1 `GET /v1/discovery-services`

Returns all listed Discovery Services, including `suspended` ones, so that clients can stop using them (§5.4).

1. The response MUST be `200` with a `data` array of `DiscoveryService` objects (§10.8).
2. The Directory MUST send HTTP caching headers [RFC9111], including `ETag`. Clients SHOULD make conditional requests.
3. The listing is readable by registered Solution Providers. Whether any part of it is public is an operational choice and is not specified here.
4. Withdrawn listings MUST NOT be returned. `GET /v1/discovery-services/{serviceId}` for a withdrawn `serviceId` MUST return `410 Gone`.

#### 10.2.2 `GET /v1/resolve`

The discovery query. The service performs the fan-out of §5.4 on the requester's behalf. This is the only way a provider resolves an identifier.

1. Query parameter `companyId` (REQUIRED) is a single URN per §3.4, percent-encoded.
2. The response MUST be `200` with `companyId`, an `answers` array of `DiscoveryAnswer` objects (§5.7), and a `timedOutServiceIds` array listing the `serviceId` of each active endpoint that timed out (§5.4). Either array MAY be empty; a non-empty `timedOutServiceIds` means the fan-out was incomplete. A `404` MUST NOT be used for "no answers".
3. The service MUST apply §5.3 (no index, no retention) and §5.4 (fan-out, timeouts, every positive answer returned, each labelled with the vouching provider).
4. The caller MUST be a registered Solution Provider (§5.5). Authentication is open decision 1.
5. The response MUST carry the interface version it was served under (§5.9).

### 10.3 Discovery Service interface

*Implemented by every listed Operator, including SPs-of-one. Its URL is the `discoveryEndpoint` of the listing. `$base-url$/3/discovery` is RECOMMENDED.*

#### 10.3.1 `GET {discoveryEndpoint}?companyId={urn}`

1. Query parameter `companyId` (REQUIRED) is a single URN, percent-encoded.
2. Where the provider publishes an Entity whose `companyIds` contains the value, it MUST return `200` with one `DiscoveryAnswer` (§5.7).
3. Otherwise it MUST return `404` with error code `NoSuchEntity`, identically for unknown and not-visible identifiers (§5.8).
4. A malformed `companyId` MUST return `400` with `BadRequest`.
5. The caller is the PACT Discovery Service. The endpoint MUST authenticate it; the mechanism is **open decision 2** of §5.6, and under the fallback variant the endpoint MUST also accept calls from any registered Solution Provider it can verify.
6. The Service SHOULD rate-limit and MAY return `429` with `Retry-After`.
7. The response MUST carry the interface version it was served under (§5.9).
7. The Service SHOULD send `Cache-Control` headers (§5.9).

### 10.4 Node connection interface

*Implemented by Credential-Exchange-capable Nodes. Its URL is the `connectionEndpoint` published by discovery. `$base-url$/3/connections` is RECOMMENDED.*

#### 10.4.1 `POST {connectionEndpoint}` — connection request (Target)

1. Request body: `ConnectionRequest` (§6.3).
2. Success: `202 Accepted` with `ConnectionAccepted` (`connectionId`, `status: pending`).
3. Errors: `400 BadRequest`; `403 RequesterNotListed`; `404 NoSuchEntity` where `target.companyId` is not served by this Node; `422 AssuranceBelowPolicy` MAY be returned immediately where the Target applies an automatic minimum; `429` with `Retry-After`.
4. *Option C2:* signed per C2-1 of §6.4; failure returns `401 InvalidSignature`.
5. *Option C1:* returns `403 RequesterNotResolvable` where the check of C1-1 fails. The Target MAY instead accept the request and run the check before approval.

#### 10.4.2 `GET {connectionEndpoint}/{connectionId}` — status poll (Target) — *Option C2 only*

1. Signed per C2-1 with the same `serviceId` as the original request. Otherwise `401 InvalidSignature`, or `404 NoSuchConnection` (not `403`, so that valid identifiers are not revealed).
2. Response `200` with `ConnectionStatus`. While `pending`, the Target SHOULD send `Retry-After`. When `approved`, the body includes `initialAccessToken` and `initialAccessTokenExpiresAt` until the token has been used or has expired.

#### 10.4.3 `POST {connectionEndpoint}/{connectionId}/decision` — decision callback (Requester) — *Option C1 only*

1. Sent by the Target to the Requester's `connectionEndpoint` as resolved through discovery (C1-2).
2. Request body: `ConnectionDecision`.
3. The Requester MUST return `204` where `connectionId` is one it created and is pending; otherwise `404 NoSuchConnection` (C1-3).
4. The Target SHOULD retry delivery with exponential back-off for at least 24 hours on `5xx` or network failure.

### 10.5 Standard OAuth interfaces

These are not defined by this addendum. They are listed here because the conformance tests for the *Credential-Exchange-capable Node* class exercise them.

| Interface | Standard | Addendum requirements |
|---|---|---|
| `$base-url$/.well-known/openid-configuration` | Already REQUIRED by the base specification | MUST include `registration_endpoint` (§4.5); MUST include `jwks_uri` where signing is chosen (§5.5, §5.6) or under option C2. |
| `POST {registration_endpoint}` | [RFC7591] | Initial access token per §6.6; `grant_types` restricted to `client_credentials`; `pact_connection_id` metadata. |
| `GET`/`PUT`/`DELETE {registration_client_uri}` | [RFC7592] | Rotation (§6.7.1) and Requester-initiated revocation (§6.7.3). |
| `jwks_uri` | [RFC7517] | Only where signing is chosen (§5.5, §5.6) or under option C2. MUST be under a domain verified for the Operator (§4.3.3). |

#### 10.5.1 PACT-operated services

*Normative. Added after the first external implementer could not find how to obtain credentials for the PACT identity management service, and used a route that exists in no published document.*

1. A PACT-operated service that requires an access token MUST publish OAuth 2.0 authorisation server metadata at `/.well-known/openid-configuration`, as the base specification expects of a host system (§5.2 of [DATA-EXCHANGE-PROTOCOL]), and that document MUST name the `token_endpoint` in use.
2. The token endpoint, the grant type, and how a registered Solution Provider obtains its credentials MUST be documented in this specification and in the service documentation. A route that is only described in correspondence does not satisfy this requirement.
3. Where a PACT-operated service serves a token route that predates this section, that route MUST remain documented and supported until a deprecation is announced under §5.9.
4. The same applies to the Discovery Service itself once open decision 1 is settled: whatever a provider needs in order to call it is published, not communicated case by case.

*Editor's note.* This is a conformance point for PACT, not for Solution Providers. It is in the specification because the gap was found in the field rather than in review.

### 10.6 Errors

Error responses reuse the base-specification error response shape: a JSON object with `code` and `message`. The following codes are added:

| Code | HTTP | Used by | Meaning |
|---|---|---|---|
| `NoSuchEntity` | 404 | Discovery Service; Target | Identifier not published by this Service, or not served by this Node. |
| `NoSuchConnection` | 404 | Target (C2); Requester (C1) | Unknown `connectionId`, or not visible to this caller. |
| `RequesterNotListed` | 401/403 | Discovery Service (B1); Target | Caller's `serviceId` is not an active Directory listing. |
| `RequesterNotResolvable` | 403 | Target (C1) | The Requester's Node could not be confirmed through its Discovery Service. |
| `InvalidSignature` | 401 | Discovery Service (B1); Target (C2) | HTTP Message Signature missing, invalid, or made with a key not at the listed `jwksUri`. |
| `AssuranceBelowPolicy` | 422 | Target | The Requester's assurance level is below the Target's minimum (§6.5). |

`BadRequest`, `TooManyRequests` (429) and `InternalError` follow base-specification usage.

### 10.7 Reserved names

| Name | Purpose | Status |
|---|---|---|
| `_pact-challenge.<domain>` DNS `TXT` | Domain challenge for listing (§4.3.3) | Provisional; underscore label per [RFC8552] |
| `/.well-known/pact-challenge/` | HTTPS domain challenge (§4.3.3) | Provisional; to be registered per [RFC8615] |
| `pact_connection_id` | RFC 7591 client metadata (§6.6) | Provisional; to be registered with IANA or renamed |
| `PACT-Service-Id` header | Identifies the signing Operator (B1, C2) | Provisional; could instead be the `keyid` of the signature |

### 10.8 OpenAPI fragment

*Non-normative in this draft; will become normative once the options are decided. OpenAPI 3.1. Operations carry `x-pact-option` where they apply only under one option, and `x-pact-implementer` to show who builds them.*

```yaml
openapi: 3.1.0
info:
  title: PACT Identity Management — Discovery and Connection interfaces
  version: 0.1.0-draft
  description: >
    Draft interfaces of the PACT Identity Management addendum to the PACT
    Technical Specifications for PCF Data Exchange V3. Directory operations are
    implemented by PACT; discovery and connection operations by solution providers.
tags:
  - name: directory
    description: PACT Directory (implemented by PACT)
  - name: discovery
    description: Discovery Service (implemented by every listed Operator)
  - name: connections
    description: Node connection interface (implemented by Credential-Exchange-capable Nodes)

paths:
  /v1/discovery-services:
    get:
      tags: [directory]
      operationId: listDiscoveryServices
      x-pact-implementer: PACT
      summary: List Discovery Services
      responses:
        '200':
          description: All active and suspended listings.
          headers:
            ETag:
              schema: { type: string }
          content:
            application/json:
              schema:
                type: object
                required: [data]
                properties:
                  data:
                    type: array
                    items: { $ref: '#/components/schemas/DiscoveryService' }

  /v1/discovery-services/{serviceId}:
    get:
      tags: [directory]
      operationId: getDiscoveryService
      x-pact-implementer: PACT
      summary: Get one Discovery Service listing
      parameters:
        - name: serviceId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: The listing.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/DiscoveryService' }
        '404':
          $ref: '#/components/responses/Error'
        '410':
          $ref: '#/components/responses/Error'

  /v1/resolve:
    get:
      tags: [directory]
      operationId: resolveViaDirectory
      x-pact-implementer: PACT
      summary: Resolve an identifier; the service fans out to registered discovery endpoints
      parameters:
        - $ref: '#/components/parameters/CompanyId'
      responses:
        '200':
          description: >
            Aggregated answers and the serviceIds of active discovery endpoints
            that timed out. timedOutServiceIds is empty when no endpoint timed out.
          content:
            application/json:
              schema:
                type: object
                required: [companyId, answers, timedOutServiceIds]
                properties:
                  companyId: { $ref: '#/components/schemas/CompanyId' }
                  answers:
                    type: array
                    items: { $ref: '#/components/schemas/DiscoveryAnswer' }
                  timedOutServiceIds:
                    type: array
                    description: serviceIds of active discovery endpoints that timed out; a non-empty array indicates incomplete fan-out.
                    items: { type: string }
        '400':
          $ref: '#/components/responses/Error'

  /3/discovery:
    get:
      tags: [discovery]
      operationId: discoverEntity
      x-pact-implementer: SP
      summary: Resolve a companyIds value to Node(s)
      description: >
        Path is RECOMMENDED; the actual URL is the listing's discoveryEndpoint.
        Called by the PACT Discovery Service. How the caller is authenticated is
        open decision 2 of section 5.6; the signature headers below apply where
        signing is chosen.
      parameters:
        - $ref: '#/components/parameters/CompanyId'
        - $ref: '#/components/parameters/ServiceIdHeader'
        - $ref: '#/components/parameters/SignatureInput'
        - $ref: '#/components/parameters/Signature'
      responses:
        '200':
          description: The Entity is published by this Service.
          content:
            application/json:
              schema: { $ref: '#/components/schemas/DiscoveryAnswer' }
        '400':
          $ref: '#/components/responses/Error'
        '401':
          $ref: '#/components/responses/Error'
        '404':
          $ref: '#/components/responses/Error'
        '429':
          $ref: '#/components/responses/Error'

  /3/connections:
    post:
      tags: [connections]
      operationId: requestConnection
      x-pact-implementer: SP
      summary: Submit a connection request to a Target Node
      description: Under Option C2 the request MUST be signed (RFC 9421, incl. content-digest).
      parameters:
        - $ref: '#/components/parameters/ServiceIdHeader'
        - $ref: '#/components/parameters/SignatureInput'
        - $ref: '#/components/parameters/Signature'
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ConnectionRequest' }
      responses:
        '202':
          description: Request accepted and pending approval.
          content:
            application/json:
              schema:
                type: object
                required: [connectionId, status]
                properties:
                  connectionId: { type: string }
                  status: { type: string, const: pending }
        '400':
          $ref: '#/components/responses/Error'
        '401':
          $ref: '#/components/responses/Error'
        '403':
          $ref: '#/components/responses/Error'
        '404':
          $ref: '#/components/responses/Error'
        '422':
          $ref: '#/components/responses/Error'
        '429':
          $ref: '#/components/responses/Error'

  /3/connections/{connectionId}:
    get:
      tags: [connections]
      operationId: getConnectionStatus
      x-pact-implementer: SP
      x-pact-option: C2
      summary: Poll the decision on a connection request (Option C2 only)
      parameters:
        - $ref: '#/components/parameters/ConnectionId'
        - $ref: '#/components/parameters/ServiceIdHeader'
        - $ref: '#/components/parameters/SignatureInput'
        - $ref: '#/components/parameters/Signature'
      responses:
        '200':
          description: Current state; includes the initial access token once approved.
          headers:
            Retry-After:
              schema: { type: integer }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ConnectionStatus' }
        '401':
          $ref: '#/components/responses/Error'
        '404':
          $ref: '#/components/responses/Error'

  /3/connections/{connectionId}/decision:
    post:
      tags: [connections]
      operationId: receiveConnectionDecision
      x-pact-implementer: SP
      x-pact-option: C1
      summary: Receive the Target's decision at the Requester Node (Option C1 only)
      parameters:
        - $ref: '#/components/parameters/ConnectionId'
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ConnectionDecision' }
      responses:
        '204':
          description: Decision received for a pending request this Node created.
        '404':
          $ref: '#/components/responses/Error'

components:
  parameters:
    CompanyId:
      name: companyId
      in: query
      required: true
      description: A single companyIds URN (percent-encoded).
      schema: { $ref: '#/components/schemas/CompanyId' }
    ConnectionId:
      name: connectionId
      in: path
      required: true
      schema: { type: string }
    ServiceIdHeader:
      name: PACT-Service-Id
      in: header
      required: false
      description: Caller's serviceId. REQUIRED where signing is chosen (sections 5.5, 5.6) and under option C2 (connections).
      schema: { type: string }
    SignatureInput:
      name: Signature-Input
      in: header
      required: false
      description: RFC 9421. REQUIRED where signing is chosen (sections 5.5, 5.6) and under option C2 (connections).
      schema: { type: string }
    Signature:
      name: Signature
      in: header
      required: false
      description: RFC 9421. REQUIRED where signing is chosen (sections 5.5, 5.6) and under option C2 (connections).
      schema: { type: string }

  responses:
    Error:
      description: Error response (base-specification shape).
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }

  schemas:
    CompanyId:
      type: string
      pattern: '^[uU][rR][nN]:[A-Za-z0-9][A-Za-z0-9-]{0,31}:.+$'
      examples: ['urn:lei:5493001KJTIIGC8Y1R12']

    Error:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
          examples: [NoSuchEntity, NoSuchConnection, RequesterNotListed, RequesterNotResolvable, InvalidSignature, AssuranceBelowPolicy, BadRequest]
        message: { type: string }

    DiscoveryService:
      type: object
      required: [serviceId, operator, domains, discoveryEndpoint, discoveryConformance, conformantVersions, domainVerifiedAt, status]
      properties:
        serviceId: { type: string, examples: ['urn:pact:discovery-service:7d0c6b8e-2f1a-4b8e-9a51-0c3e1d5a9f10'] }
        operator:
          type: object
          required: [companyIds, name]
          properties:
            companyIds:
              type: array
              minItems: 1
              items: { $ref: '#/components/schemas/CompanyId' }
            name: { type: string }
        domains:
          type: array
          minItems: 1
          items: { type: string, format: hostname }
        discoveryEndpoint: { type: string, format: uri, pattern: '^https://' }
        discoveryConformance:
          type: object
          required: [testSuiteVersion, passedAt]
          properties:
            testSuiteVersion: { type: string }
            passedAt: { type: string, format: date-time }
        conformantVersions:
          type: array
          items: { type: string }
        domainVerifiedAt: { type: string, format: date-time }
        status: { type: string, enum: [active, suspended] }
        jwksUri:
          type: string
          format: uri
          description: Present only where signing is chosen (sections 5.5, 5.6) or under option C2.

    AssuranceLevel:
      type: string
      enum: [self-asserted, email-verified, domain-verified, registry-verified, credential-verified]

    Evidence:
      type: object
      required: [method, verifiedAt]
      properties:
        method: { type: string, enum: [email, domain, lei-status, vc] }
        verifiedAt: { type: string, format: date-time }
        domain: { type: string, format: hostname, description: 'For method email or domain.' }
        lei: { type: string, pattern: '^[A-Z0-9]{18}[0-9]{2}$', description: 'For method lei-status.' }
        profile: { type: string, examples: [vlei], description: 'For method vc.' }

    DiscoveryNode:
      type: object
      required: [nodeId, baseUrl, provisional]
      properties:
        nodeId: { type: string }
        baseUrl: { type: string, format: uri, pattern: '^https://' }
        connectionEndpoint: { type: string, format: uri, pattern: '^https://' }
        conformantVersions:
          type: array
          items: { type: string }
        provisional: { type: boolean }

    DiscoveryAnswer:
      type: object
      required: [companyIds, vouchedBy, nodes]
      properties:
        companyIds:
          type: array
          minItems: 1
          items: { $ref: '#/components/schemas/CompanyId' }
        legalName: { type: string }
        vouchedBy: { type: string, description: serviceId of the Solution Provider that answered, and therefore vouches. }
        assurance:
          type: object
          description: Optional. Absence means the provider's vouching alone.
          required: [level, evidence]
          properties:
            level: { $ref: '#/components/schemas/AssuranceLevel' }
            evidence:
              type: array
              items: { $ref: '#/components/schemas/Evidence' }
        nodes:
          type: array
          minItems: 1
          items: { $ref: '#/components/schemas/DiscoveryNode' }

    ConnectionRequest:
      type: object
      required: [requester, target]
      properties:
        requester:
          type: object
          required: [companyIds, name, nodeId, serviceId]
          properties:
            companyIds:
              type: array
              minItems: 1
              items: { $ref: '#/components/schemas/CompanyId' }
            name: { type: string }
            nodeId: { type: string }
            serviceId: { type: string }
        target:
          type: object
          required: [companyId]
          properties:
            companyId: { $ref: '#/components/schemas/CompanyId' }
        message: { type: string, maxLength: 1000 }

    ConnectionState:
      type: string
      enum: [pending, approved, declined, expired, active, suspended, revoked]

    ConnectionStatus:
      type: object
      required: [connectionId, status]
      properties:
        connectionId: { type: string }
        status: { $ref: '#/components/schemas/ConnectionState' }
        initialAccessToken: { type: string, description: Present only while approved and unused. }
        initialAccessTokenExpiresAt: { type: string, format: date-time }

    ConnectionDecision:
      type: object
      required: [connectionId, target, status]
      properties:
        connectionId: { type: string }
        target:
          type: object
          required: [companyId]
          properties:
            companyId: { $ref: '#/components/schemas/CompanyId' }
        status: { type: string, enum: [approved, declined, expired] }
        initialAccessToken: { type: string, description: Present only when approved. }
        initialAccessTokenExpiresAt: { type: string, format: date-time }
```

### 10.9 Implementation checklist for solution providers

*Non-normative. What a solution provider builds, by capability. The conformance classes of §9 follow the same split.*

**To make customers discoverable** (*Discoverable Node* / *Discovery Service Provider*)

1. Publish the domain challenge for the domain(s) of your Discovery Service (§4.3.3).
2. Implement `GET /3/discovery` over your customer base, returning only customers that opted in (§5.8).
3. Record for each customer which checks you performed (email, domain, LEI, VC) and when, and publish the resulting assurance (§3.5.2).
4. Pass the discovery conformance tests and get listed (§4.3.4).
5. Authenticate the PACT Discovery Service on incoming queries (open decision 2), and obtain whatever the service requires to be called (open decision 1).

**To accept connection requests** (*Credential-Exchange-capable Node*)

6. Implement `POST /3/connections` and an approval step in your product for the data owner.
7. Enable RFC 7591/7592 on your authorisation server and advertise `registration_endpoint` in `.well-known/openid-configuration`.
8. Issue single-use initial access tokens bound to the approved connection (§6.6).
9. *Option C1:* resolve the Requester and deliver the decision to its published endpoint. *Option C2:* verify request signatures and serve `GET /3/connections/{id}`.

**To request connections for your customers**

10. Resolve the target with one call to the PACT Discovery Service (§10.2.2). No client-side fan-out.
11. Send the connection request. *Option C1:* implement `POST /3/connections/{id}/decision`. *Option C2:* sign requests and poll.
12. Register as an OAuth client with the initial access token, store the credentials, and use them in the unchanged §5.5 token flow.

---

## Open items carried from this section

- Open decisions 1 and 2 (§5.5, §5.6) and option C (§6.4) decide which marked operations and headers become normative.
- The announcement channel and transition period for interface version changes (§5.9).
- `$directory-url$` hostname and API versioning of the Directory (parked item P1).
- Registration of the reserved names in §10.7.
- Whether `PACT-Service-Id` is replaced by the signature `keyid`.
- Whether the Directory should also offer a change feed or webhook for listing changes (parked item P1).

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC7517] — JSON Web Key (JWK).
- [RFC7591] — OAuth 2.0 Dynamic Client Registration Protocol.
- [RFC7592] — OAuth 2.0 Dynamic Client Registration Management Protocol.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC8552] — Scoped Interpretation of DNS Resource Records through "Underscored" Naming of Attribute Leaves.
- [RFC8615] — Well-Known Uniform Resource Identifiers.
- [RFC9111] — HTTP Caching.
- [RFC9421] — HTTP Message Signatures.
- [OPENAPI] — OpenAPI Specification 3.1.
