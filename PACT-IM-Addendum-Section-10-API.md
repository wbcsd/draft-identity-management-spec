# PACT Identity Management Addendum — §10 API Specification (Draft)

*Draft v0.1 — 16 September 2026. First draft of the interface additions for the PACT Identity Management addendum to [DATA-EXCHANGE-PROTOCOL] V3, for discussion at the Technology Working Group. Defines the HTTP interfaces behind §4 (Registration), §5 (Node Discoverability) and §6 (Automated Credential Exchange), written from a solution-provider implementation perspective. Editorial conventions follow the base specification: the key words MUST, MUST NOT, SHOULD, SHOULD NOT, MAY, REQUIRED, and OPTIONAL are to be interpreted as in [RFC2119]/[RFC8174] when, and only when, in all capitals.*

> **Status of this section.** Draft endpoints for discussion; paths, parameter names and error codes are provisional. Discovery is now a single design — one PACT-operated service that fans a query out (W5), private to registered providers (W6). How every caller authenticates is now one proposal of 11 November 2026, for WG confirmation: OAuth client credentials in both directions between providers and PACT, exchanged at listing; and unsigned connection requests, with the decision delivered only to the Requester's Operator at its listed base URL (§4.3.6, §5.5, §5.6, §6.4). The design guardrail from the WG is that implementing the addendum should require **only a handful of new API methods**. §10.1 counts them.

---

## 10. API Specification

### 10.1 Overview and method count

The addendum adds interfaces in three places. Only the second and third are implemented by solution providers.

| Where | Operation | Implemented by | Status | PACT-specific? |
|---|---|---|---|---|
| PACT Discovery Service | `GET /im/service-info` | PACT | normative — unversioned, lists supported and deprecated versions (§5.10) | yes |
| PACT Discovery Service | `GET /im/{major}/resolve?companyId=` | PACT | normative | yes |
| PACT Discovery Service | `GET /im/{major}/discovery-services` | PACT | normative — registry listing, publicly readable | yes |
| PACT Discovery Service | `GET /im/{major}/discovery-services/{serviceId}` | PACT | normative | yes |
| PACT-operated services | `/.well-known/openid-configuration` and the token endpoint it names | PACT | normative (§10.5.1) | no — [RFC8414]/[RFC6749] |
| Solution Provider | `GET /im/{major}/discovery?companyId=` | **SP** | normative — called by the Discovery Service | **yes** |
| Target Node | `POST /im/{major}/connections` | **SP** | normative | **yes** |
| Requester's Operator | `POST {discoveryBaseUrl}/im/{major}/connections/{connectionId}/decision` | **SP** | normative (§6.4.2) | **yes** |
| Target Node | `POST {registration_endpoint}` | SP's authorisation server | normative | no — [RFC7591] |
| Target Node | `GET`/`PUT`/`DELETE {registration_client_uri}` | SP's authorisation server | normative | no — [RFC7592] |

**Result against the guardrail.** A Solution Provider implements **three new PACT-specific methods**: the discovery endpoint, the connection request, and the decision callback (§6.4.2). Authentication adds nothing new: in both directions it is the OAuth client-credentials flow that providers already run for the base specification. Central fan-out removes the client-side aggregation that the earlier draft put on every provider. Everything else is standard OAuth 2.0 that many authorisation servers already provide.

**Authentication at a glance** (§4.3.6, §5.5, §5.6, §6.4):

| Call | From → to | Credential | Issued by |
|---|---|---|---|
| Token request, then `GET /im/{major}/resolve` | Provider → PACT | `client_id` / `client_secret`, then a PACT access token (`Bearer`) | PACT, at listing |
| Token request, then `GET {discoveryBaseUrl}/im/{major}/discovery` | PACT → provider | `client_id` / `client_secret`, then the provider's own access token (`Bearer`) | The provider, at listing |
| `POST {baseUrl}/im/{major}/connections` | Requester's Operator → Target Node | None. The Target checks that the Requester's listing is active | — |
| `POST {discoveryBaseUrl}/im/{major}/connections/{id}/decision` | Target → Requester's Operator | None. Sent only to the listed base URL, matched on the unguessable `connectionId` | — |
| `GET /im/{major}/discovery-services`, `GET /im/service-info` | Anyone | None — public | — |

### 10.2 PACT Directory interface

*Implemented by PACT. Base URL `$directory-url$` (to be assigned).*

#### 10.2.1 `GET /im/{major}/discovery-services`

Returns all listed Discovery Services, including `suspended` ones, so that clients can stop using them (§5.4).

1. The response MUST be `200` with a `data` array of `DiscoveryService` objects (§10.8).
2. The Directory MUST send HTTP caching headers [RFC9111], including `ETag`. Clients SHOULD make conditional requests.
3. The listing MUST be readable without authentication. It contains no secret material (§4.3.2). A Target reads it to check a Requester's listing and find where to send its decision (§6.4).
4. Withdrawn listings MUST NOT be returned. `GET /im/{major}/discovery-services/{serviceId}` for a withdrawn `serviceId` MUST return `410 Gone`.

#### 10.2.2 `GET /im/{major}/resolve`

The discovery query. The service performs the fan-out of §5.4 on the requester's behalf. This is the only way a provider resolves an identifier.

1. Query parameter `companyId` (REQUIRED) is a single URN per §3.4, percent-encoded.
2. The response MUST be `200` with `companyId`, an `answers` array of `DiscoveryAnswer` objects (§5.7), and a `timedOutServiceIds` array listing the `serviceId` of each active endpoint that timed out (§5.4). Either array MAY be empty; a non-empty `timedOutServiceIds` means the fan-out was incomplete. A `404` MUST NOT be used for "no answers".
3. The service MUST apply §5.3 (no index, no retention) and §5.4 (fan-out, timeouts, every positive answer returned, each labelled with the vouching provider).
4. The caller MUST present a PACT access token obtained as in §5.5.1, otherwise `401`. A token for a listing that is no longer `active` is answered with `403 RequesterNotListed`.
5. The response MUST carry the full interface version in the `PACT-IM-Version` header (§5.9).

#### 10.2.3 `GET /im/service-info`

The one unversioned path, so that a client can find out which versions exist before choosing one (§5.10).

1. The response MUST be `200` with a `ServiceInfo` object (§10.8): the current IM interface version, the major versions supported, and for each deprecated version its deprecation date, withdrawal date once fixed, and a link to release notes.
2. The endpoint MUST be readable without authentication.

#### 10.2.4 Version headers on every IM response

Applies to every operation in §10.2–§10.4, whoever implements it.

1. Every response MUST carry `PACT-IM-Version: {MAJOR.MINOR}` — the full version it was served under (§5.9).
2. A response served under a deprecated major version MUST carry `Deprecation` [RFC9745] and, once a withdrawal date is fixed, `Sunset` [RFC8594], each with a `Link` header pointing to the release notes (§5.10). This applies to the PACT Discovery Service; Solution Providers do not need to set them.

### 10.3 Discovery Service interface

*Implemented by every listed Operator, including SPs-of-one. Its URL is `{discoveryBaseUrl}/im/{major}/discovery`, where `discoveryBaseUrl` comes from the listing (§4.3.2) and `major` is the version the service calls at (§5.9.1).*

#### 10.3.1 `GET {discoveryBaseUrl}/im/{major}/discovery?companyId={urn}`

1. Query parameter `companyId` (REQUIRED) is a single URN, percent-encoded.
2. Where the provider publishes an Entity whose `companyIds` contains the value, it MUST return `200` with one `DiscoveryAnswer` (§5.7).
3. Otherwise it MUST return `404` with error code `NoSuchEntity`, identically for unknown and not-visible identifiers (§5.8).
4. A malformed `companyId` MUST return `400` with `BadRequest`.
5. The caller is the PACT Discovery Service, with an access token the provider's own authorisation server issued to PACT's client (§5.6). The endpoint MUST answer `401` to any other caller, including other Solution Providers.
6. The Service SHOULD rate-limit and MAY return `429` with `Retry-After`.
7. The response MUST carry the full interface version in the `PACT-IM-Version` header (§5.9).
8. The Service SHOULD send `Cache-Control` headers (§5.11).

### 10.4 Node connection interface

*The connection request is implemented by Credential-Exchange-capable Nodes at `{baseUrl}/im/{major}/connections`, at a major version supported by both the Requester and Target Nodes (§5.9.1). The decision callback is implemented by the Requester's Operator at the `discoveryBaseUrl` of its listing.*

#### 10.4.1 `POST {baseUrl}/im/{major}/connections` — connection request (Target)

1. Request body: `ConnectionRequest` (§6.3).
2. Success: `202 Accepted` with `ConnectionAccepted` (`connectionId`, `status: pending`).
3. Errors: `400 BadRequest`; `403 RequesterNotListed`; `404 NoSuchEntity` where `target.companyId` is not served by this Node; `422 AssuranceBelowPolicy` MAY be returned immediately where the Target applies an automatic minimum; `429` with `Retry-After`.
4. Carries no credentials. The Target checks the Requester's listing as in §6.4.1: a listing that is not `active` returns `403 RequesterNotListed`.
5. Where the major in the path is not declared in the listing of the Requester's Operator, the Target MUST return `409 IncompatibleInterfaceVersion`, because it could not deliver the decision (§6.4.2).

#### 10.4.2 `POST {discoveryBaseUrl}/im/{major}/connections/{connectionId}/decision` — decision callback (Requester's Operator)

1. Sent by the Target to the `discoveryBaseUrl` in the Directory listing of `requester.serviceId`, at the major of the original request (§6.4.2).
2. Carries no credentials. The `connectionId`, unguessable and known only to the two parties (§6.3), identifies the request.
3. Request body: `ConnectionDecision`.
4. The Requester's Operator MUST return `204` where `connectionId` is one its Nodes created and is pending; otherwise `404 NoSuchConnection`.
5. The Target SHOULD retry delivery with exponential back-off for at least 24 hours on `5xx` or network failure.

### 10.5 Standard OAuth interfaces

These are not defined by this addendum. They are listed here because the conformance tests for the *Credential-Exchange-capable Node* class exercise them.

| Interface | Standard | Addendum requirements |
|---|---|---|
| `$base-url$/.well-known/openid-configuration` | Already REQUIRED by the base specification | MUST include `registration_endpoint` (§4.5). |
| `POST {registration_endpoint}` | [RFC7591] | Initial access token per §6.6; `grant_types` restricted to `client_credentials`; `pact_connection_id` metadata. |
| `GET`/`PUT`/`DELETE {registration_client_uri}` | [RFC7592] | Rotation (§6.7.1) and Requester-initiated revocation (§6.7.3). |
| PACT token endpoint | [RFC6749] §4.4 | Client credentials with the `client_id` and `client_secret` issued at listing (§5.5.1). |
| The Operator's token endpoint | Base specification §5.5 | PACT obtains tokens for discovery calls with the credentials the Operator issued to it (§5.6). |

#### 10.5.1 PACT-operated services

*Normative. Added after the first external implementer could not find how to obtain credentials for the PACT identity management service, and used a route that exists in no published document.*

1. A PACT-operated service that requires an access token MUST publish OAuth 2.0 authorisation server metadata at `/.well-known/openid-configuration`, as the base specification expects of a host system (§5.2 of [DATA-EXCHANGE-PROTOCOL]), and that document MUST name the `token_endpoint` in use.
2. The token endpoint, the grant type, and how a registered Solution Provider obtains its credentials MUST be documented in this specification and in the service documentation. A route that is only described in correspondence does not satisfy this requirement.
3. Where a PACT-operated service serves a token route that predates this section, that route MUST remain documented and supported until a deprecation is announced under §5.9.
4. For the Discovery Service this is §5.5.1: the token endpoint is named in its well-known document, the grant is client credentials, and the credentials are issued at listing (§4.3.6). The client-secret route already in use is this mechanism, now documented.

*Editor's note.* This is a conformance point for PACT, not for Solution Providers. It is in the specification because the gap was found in the field rather than in review.

### 10.6 Errors

Error responses reuse the base-specification error response shape: a JSON object with `code` and `message`. The following codes are added:

| Code | HTTP | Used by | Meaning |
|---|---|---|---|
| `NoSuchEntity` | 404 | Discovery Service; Target | Identifier not published by this Service, or not served by this Node. |
| `NoSuchConnection` | 404 | Requester's Operator (§10.4.2) | Unknown `connectionId`, or not visible to this caller. |
| `RequesterNotListed` | 403 | Discovery Service; Target | The caller's listing (the token's client, or `requester.serviceId`) is not active. |
| `IncompatibleInterfaceVersion` | 409 | Target | The request uses a major that the Requester's Operator does not declare, so the decision could not be delivered (§6.4.2). |
| `AssuranceBelowPolicy` | 422 | Target | The Requester's assurance level is below the Target's minimum (§6.5). |

`BadRequest`, `TooManyRequests` (429) and `InternalError` follow base-specification usage.

### 10.7 Reserved names

| Name | Purpose | Status |
|---|---|---|
| `_pact-challenge.<domain>` DNS `TXT` | Domain challenge for listing (§4.3.3) | Provisional; underscore label per [RFC8552] |
| `/.well-known/pact-challenge/` | HTTPS domain challenge (§4.3.3) | Provisional; to be registered per [RFC8615] |
| `pact_connection_id` | RFC 7591 client metadata (§6.6) | Provisional; to be registered with IANA or renamed |
| `PACT-IM-Version` header | Full IM interface version on every response (§5.9) | Provisional |
| `/im/` path prefix | Root of every IM interface path, versioned by major (§5.9) | Provisional |

### 10.8 OpenAPI fragment

*Non-normative in this draft; becomes normative once the WG confirms the authentication proposal. OpenAPI 3.1. Operations carry `x-pact-implementer` to show who builds them and `security` to show how each is authenticated.*

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
  /im/service-info:
    get:
      tags: [directory]
      operationId: getServiceInfo
      x-pact-implementer: PACT
      summary: Supported and deprecated IM interface versions (unversioned path)
      security: []
      responses:
        '200':
          description: Version information.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ServiceInfo' }

  /im/1/discovery-services:
    get:
      tags: [directory]
      operationId: listDiscoveryServices
      x-pact-implementer: PACT
      summary: List Discovery Services (public)
      security: []
      responses:
        '200':
          description: All active and suspended listings.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
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

  /im/1/discovery-services/{serviceId}:
    get:
      tags: [directory]
      operationId: getDiscoveryService
      x-pact-implementer: PACT
      summary: Get one Discovery Service listing (public)
      security: []
      parameters:
        - name: serviceId
          in: path
          required: true
          schema: { type: string }
      responses:
        '200':
          description: The listing.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/DiscoveryService' }
        '404':
          $ref: '#/components/responses/Error'
        '410':
          $ref: '#/components/responses/Error'

  /im/1/resolve:
    get:
      tags: [directory]
      operationId: resolveViaDirectory
      x-pact-implementer: PACT
      summary: Resolve an identifier; the service fans out to registered discovery endpoints
      security:
        - pactAccessToken: []
      parameters:
        - $ref: '#/components/parameters/CompanyId'
      responses:
        '200':
          description: >
            Aggregated answers and the serviceIds of active discovery endpoints
            that timed out. timedOutServiceIds is empty when no endpoint timed out.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
            Deprecation: { $ref: '#/components/headers/Deprecation' }
            Sunset: { $ref: '#/components/headers/Sunset' }
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
        '401':
          $ref: '#/components/responses/Error'
        '403':
          $ref: '#/components/responses/Error'

  /im/1/discovery:
    get:
      tags: [discovery]
      operationId: discoverEntity
      x-pact-implementer: SP
      summary: Resolve a companyIds value to Node(s)
      description: >
        Fixed path {discoveryBaseUrl}/im/{major}/discovery; shown here for major version 1.
        Called only by the PACT Discovery Service, with an access token from the
        provider's own authorisation server (section 5.6).
      security:
        - providerAccessToken: []
      parameters:
        - $ref: '#/components/parameters/CompanyId'
      responses:
        '200':
          description: The Entity is published by this Service.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
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

  /im/1/connections:
    post:
      tags: [connections]
      operationId: requestConnection
      x-pact-implementer: SP
      summary: Submit a connection request to a Target Node
      description: >
        Carries no credentials. The Target checks that the Requester's listing is active
        and delivers its decision only to that listing's base URL (section 6.4).
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ConnectionRequest' }
      responses:
        '202':
          description: Request accepted and pending approval.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
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
        '403':
          $ref: '#/components/responses/Error'
        '404':
          $ref: '#/components/responses/Error'
        '409':
          $ref: '#/components/responses/Error'
        '422':
          $ref: '#/components/responses/Error'
        '429':
          $ref: '#/components/responses/Error'

  /im/1/connections/{connectionId}/decision:
    post:
      tags: [connections]
      operationId: receiveConnectionDecision
      x-pact-implementer: SP
      summary: Receive the Target's decision at the Requester's Operator
      description: >
        Fixed path {discoveryBaseUrl}/im/{major}/connections/{connectionId}/decision
        on the Requester's Operator, taken from its Directory listing; shown here for
        major version 1. Carries no credentials; matched on the unguessable connectionId (section 6.4.2).
      security: []
      parameters:
        - $ref: '#/components/parameters/ConnectionId'
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ConnectionDecision' }
      responses:
        '204':
          description: Decision received for a pending request one of this Operator's Nodes created.
          headers:
            PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
        '404':
          $ref: '#/components/responses/Error'

components:
  securitySchemes:
    pactAccessToken:
      type: http
      scheme: bearer
      description: >
        Access token from the PACT Discovery Service token endpoint, client-credentials
        grant with the client_id and client_secret PACT issued at listing (section 5.5.1).
    providerAccessToken:
      type: http
      scheme: bearer
      description: >
        Access token issued by the provider's own authorisation server to the client
        it registered for PACT at listing (sections 4.3.6 and 5.6).

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

  headers:
    PactImVersion:
      description: Full IM interface version the response was served under, MAJOR.MINOR (section 5.9).
      required: true
      schema: { type: string, pattern: '^[0-9]+[.][0-9]+$' }
    Deprecation:
      description: RFC 9745. Present when the version is deprecated.
      schema: { type: string }
    Sunset:
      description: RFC 8594. Present once a withdrawal date is fixed.
      schema: { type: string }

  responses:
    Error:
      description: Error response (base-specification shape).
      headers:
        PACT-IM-Version: { $ref: '#/components/headers/PactImVersion' }
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
          examples: [NoSuchEntity, NoSuchConnection, RequesterNotListed, IncompatibleInterfaceVersion, AssuranceBelowPolicy, BadRequest]
        message: { type: string }

    DiscoveryService:
      type: object
      required: [serviceId, operator, domains, discoveryBaseUrl, interfaceVersions, discoveryConformance, conformantVersions, domainVerifiedAt, status]
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
        discoveryBaseUrl: { type: string, format: uri, pattern: '^https://' }
        interfaceVersions:
          type: array
          minItems: 1
          description: Major versions of the IM interface the discovery endpoint supports (section 5.9.1).
          items: { type: string, pattern: '^[0-9]+$' }
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
        interfaceVersions:
          type: array
          description: Major versions of the IM connection interface; the connection endpoint is {baseUrl}/im/{major}/connections. Required for Credential-Exchange-capable Nodes.
          items: { type: string, pattern: '^[0-9]+$' }
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

    ServiceInfo:
      type: object
      required: [currentVersion, supportedMajors]
      properties:
        currentVersion: { type: string, pattern: '^[0-9]+[.][0-9]+$', examples: ['1.0'] }
        supportedMajors:
          type: array
          items: { type: string, pattern: '^[0-9]+$' }
        deprecated:
          type: array
          items:
            type: object
            required: [major, deprecatedAt, releaseNotes]
            properties:
              major: { type: string, pattern: '^[0-9]+$' }
              deprecatedAt: { type: string, format: date }
              withdrawnAt: { type: string, format: date }
              releaseNotes: { type: string, format: uri }
```

### 10.9 Implementation checklist for solution providers

*Non-normative. What a solution provider builds, by capability. The conformance classes of §9 follow the same split.*

**To make customers discoverable** (*Discoverable Node* / *Discovery Service Provider*)

1. Publish the domain challenge for the domain(s) of your Discovery Service (§4.3.3).
2. Implement `GET {discoveryBaseUrl}/im/{major}/discovery` over your customer base, returning only customers that opted in (§5.8).
3. Record for each customer which checks you performed (email, domain, LEI, VC) and when, and publish the resulting assurance (§3.5.2).
4. Declare the IM major versions you support (`interfaceVersions`), pass the discovery conformance tests and get listed (§4.3.4, §5.9.1).
5. Issue PACT a `client_id` and `client_secret` on your authorisation server, hand them over at listing, and accept discovery queries only with a token issued to that client (§4.3.6, §5.6).

**To accept connection requests** (*Credential-Exchange-capable Node*)

6. Implement `POST /im/{major}/connections` and an approval step in your product for the data owner.
7. Enable RFC 7591/7592 on your authorisation server and advertise `registration_endpoint` in `.well-known/openid-configuration`.
8. Issue single-use initial access tokens bound to the approved connection (§6.6).
9. Check that the Requester's listing is active (§6.4.1), and deliver the decision to the `discoveryBaseUrl` in that listing (§6.4.2).

**To request connections for your customers**

10. Obtain a PACT access token with the credentials issued to you at listing (§5.5.1) and resolve the target with one call to the PACT Discovery Service (§10.2.2). No client-side fan-out.
11. Send the connection request (§6.3), and implement `POST {discoveryBaseUrl}/im/{major}/connections/{id}/decision` to receive decisions for your Nodes (§10.4.2).
12. Register as an OAuth client with the initial access token, store the credentials, and use them in the unchanged §5.5 token flow.

---

## Open items carried from this section

- WG confirmation of the authentication proposal (§4.3.6, §5.5, §5.6, §6.4), including the channel for exchanging credentials at listing and the rotation overlap.
- The announcement channel and transition period for interface version changes (§5.9).
- `$directory-url$` hostname and API versioning of the Directory (parked item P1).
- Registration of the reserved names in §10.7.
- Whether the Directory should also offer a change feed or webhook for listing changes (parked item P1).

---

### References used in this section

- [DATA-EXCHANGE-PROTOCOL] — Technical Specifications for PCF Data Exchange, V3.0.3.
- [RFC2119], [RFC8174] — requirement-level keywords.
- [RFC6749] — The OAuth 2.0 Authorization Framework.
- [RFC7591] — OAuth 2.0 Dynamic Client Registration Protocol.
- [RFC7592] — OAuth 2.0 Dynamic Client Registration Management Protocol.
- [RFC8414] — OAuth 2.0 Authorization Server Metadata.
- [RFC8552] — Scoped Interpretation of DNS Resource Records through "Underscored" Naming of Attribute Leaves.
- [RFC8594] — The Sunset HTTP Header Field.
- [RFC8615] — Well-Known Uniform Resource Identifiers.
- [RFC9111] — HTTP Caching.
- [RFC9745] — The Deprecation HTTP Response Header Field.
- [OPENAPI] — OpenAPI Specification 3.1.
