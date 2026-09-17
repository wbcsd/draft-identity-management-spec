# PACT Identity Management — Discover, Verify, Connect (Flow)

*Draft v0.2 — 16 September 2026. Non-normative figure supporting §4 (Registration), §5 (Node Discoverability) and §6 (Automated Credential Exchange) of the PACT Identity Management addendum. Reflects the 2 September 2026 WG decisions: a **PACT Directory of Discovery Services** holding no buyer or supplier data, **domain ownership inside discovery conformance** for Operators, **LEI and verifiable credentials as optional layers**, and **email/domain assurance attested by each party's own Operator**. **RFC 7591 Dynamic Client Registration** remains the credential-provisioning binding.*

> **Changes in v0.2.** Registration now happens in two layers: the Operator is listed through conformance, and the supplier is onboarded by its Operator. The root no longer holds delegations keyed by LEI. Discovery fans out across Discovery Services. The GLEIF lookup is optional and performed by the Operator. The three open WG decisions (A fan-out, B query authorisation, C request authentication) are shown as alternatives.

The scenario: a **buyer** wants PCF data from a **supplier** it has never exchanged with. Both are customers of different solution providers: **SP-B** serves the buyer, **SP-S** serves the supplier. Nothing in the flow puts PACT in the credential or data path.

---

## End-to-end flow

```mermaid
sequenceDiagram
    participant B as Buyer Node via SP-B
    participant DB as SP-B Discovery Service
    participant D as PACT Directory
    participant CS as PACT Conformance Service
    participant DS as SP-S Discovery Service
    participant S as Supplier Node via SP-S

    rect rgb(238, 242, 250)
    Note over CS,DS: Phase 0a - Operator listing, conformance-gated (section 4.3)
    DS->>CS: Apply for listing with domains and discovery endpoint
    CS->>DS: Domain challenge token
    DS-->>CS: Token published in DNS TXT or at well-known path
    CS->>DS: Discovery conformance tests
    CS->>D: Passed, list SP-S Discovery Service as active
    Note over D: Directory holds SP listings only, no buyer or supplier data
    Note over DB,D: SP-B Discovery Service is listed the same way
    end

    rect rgb(240, 247, 250)
    Note over DS,S: Phase 0b - Supplier onboarding at its own SP (Operator-internal, section 4.5)
    S->>DS: Supplier becomes an SP-S customer and opts in to discovery
    Note over DS,S: SP-S verifies email or domain, optionally checks LEI or a vLEI, records evidence and timestamps
    end

    rect rgb(240, 247, 240)
    Note over B,S: Phase 1 - Discovery by companyIds URN (section 5)
    B->>D: GET discovery-services
    D-->>B: Active listings including SP-S
    alt Option A1 - client-side fan-out
        B->>DS: GET discovery for supplier companyId, and in parallel every other listed service
        DS-->>B: Discovery answer with nodes, assurance, attestedBy SP-S
    else Option A2 - directory-brokered fan-out
        B->>D: GET resolve for supplier companyId
        D->>DS: GET discovery for supplier companyId, and every other listed service
        DS-->>D: Discovery answer
        D-->>B: Aggregated answers, nothing retained
    end
    Note over B,DS: Option B decides who may query - B1 listed and signed, B2 public with minimal answer
    Note over B,DS: Unknown and not-opted-in identifiers both return 404
    B->>S: GET well-known openid-configuration
    S-->>B: token_endpoint and registration_endpoint
    end

    rect rgb(252, 246, 236)
    Note over B,S: Phase 2 - Assurance check (sections 3.5 and 3.7)
    Note over B: Buyer policy reads level and evidence attested by SP-S, for example domain-verified
    opt Supplier is credential-verified
        Note over B: Buyer may verify the credential itself, vLEI profile
    end
    end

    rect rgb(247, 240, 248)
    Note over B,S: Phase 3 - Connection request and approval (sections 6.3 to 6.5)
    alt Option C1 - decision sent to published endpoint
        B->>S: POST connections, unsigned
        S-->>B: 202 Accepted, connectionId, pending
        S->>DB: Resolve buyer companyId, confirm buyer node and connection endpoint
        DB-->>S: Discovery answer for the buyer
        Note over S: Owner approves - manual in V1, both parties accept email or domain assurance
        S->>B: POST connections id decision to buyer endpoint from discovery, with initial access token
        B-->>S: 204 for a connectionId it created
    else Option C2 - signed request and polling
        B->>S: POST connections, signed with SP-B key from its listed jwksUri
        S->>D: Check SP-B listing active and fetch jwksUri
        S-->>B: 202 Accepted, connectionId, pending
        Note over S: Owner approves - manual in V1
        B->>S: GET connections id, signed poll
        S-->>B: Approved, initial access token
    end
    end

    rect rgb(236, 244, 250)
    Note over B,S: Phase 4 - Credential provisioning, RFC 7591 (section 6.6)
    B->>S: POST registration_endpoint with initial access token and client metadata
    S-->>B: client_id, client_secret, registration_access_token, registration_client_uri
    Note over B,S: PACT is never a party to this exchange
    end

    rect rgb(240, 240, 240)
    Note over B,S: Phase 5 - Exchange, base spec unchanged (V3 section 5.5)
    B->>S: POST token_endpoint, client_credentials grant
    S-->>B: access_token
    B->>S: GET /3/footprints with bearer token
    S-->>B: PCF data
    end

    rect rgb(250, 240, 240)
    Note over B,S: Phase 6 - Lifecycle (section 6.7)
    opt Rotate
        B->>S: PUT registration_client_uri with registration_access_token
        S-->>B: New client_secret
    end
    opt Revoke
        S->>S: Invalidate registration and tokens, set connection revoked
        S-->>B: Subsequent token requests rejected
    end
    end
```

---

## What happens at each phase

**Phase 0a — Operator listing.** SP-S applies to the PACT Directory. The PACT Conformance Service checks that SP-S controls its domain (a DNS `TXT` record or a file under `/.well-known/`) and runs the discovery conformance tests. Only then is the Discovery Service listed. A home-built solution that has not passed these tests cannot be listed. The Directory holds the SP's listing and nothing about its customers. SP-B goes through the same steps. A company running its own node is listed in exactly the same way, as an *SP of one*.

**Phase 0b — Supplier onboarding.** The supplier signs up with SP-S and opts in to being discoverable. SP-S decides how to verify it: typically an email or domain check, optionally an LEI status check or a verifiable credential. SP-S records what it checked and when. None of this goes to PACT.

**Phase 1 — Discovery.** The buyer fetches the Directory listing and looks the supplier up by a `companyIds` URN. Under **Option A1** the buyer queries every listed Discovery Service itself. Under **Option A2** the Directory does that on its behalf and keeps nothing. **Option B** decides whether queries must be signed by a listed Operator or may be public with a minimal answer. A `404` never reveals whether an identifier is unknown or simply not opted in. The buyer then reads the supplier node's existing `/.well-known/openid-configuration` for the token and registration endpoints, so no Directory or Discovery Service ever holds, or goes stale on, those values.

**Phase 2 — Assurance check.** The answer carries an assurance level (*self-asserted*, *email-verified*, *domain-verified*, *registry-verified*, *credential-verified*), the evidence behind it, and which SP attests it. The buyer's policy decides whether that is enough. For v1, email or domain assurance is enough where both parties agree. A credential can be checked directly where one is bound.

**Phase 3 — Connection.** The buyer asks to connect, and the supplier's owner approves. **Option C1** needs no cryptography: the supplier confirms the buyer's node through SP-B's Discovery Service and sends the approval, with the one-time token, only to the endpoint published there. Anyone impersonating the buyer never receives it. **Option C2** has the buyer's SP sign the request with a key under its verified domain, and the buyer polls for the decision. **Being discoverable is not being connected:** approval is always the supplier's, and always explicit.

**Phase 4 — Credential provisioning.** The buyer spends the one-time initial access token at the supplier's RFC 7591 registration endpoint. It receives the `client_id` and `client_secret` the base specification's OAuth flow already expects, plus the RFC 7592 handles that make rotation possible without a second manual handshake. This is the exact step V3 §5.3 leaves out of scope, and it happens strictly peer-to-peer.

**Phase 5 — Exchange.** Unchanged from V3 §5.5.

**Phase 6 — Lifecycle.** Rotation is a client-initiated update against the registration URI. Revocation invalidates the registration and all live tokens. Delisting an SP, or a supplier withdrawing from discovery, is *not* revocation: credentials survive until they are revoked here (§4.4.3, §6.7.2).

---

## Appendix — connection lifecycle states

*Non-normative companion view of Phases 3 to 6.*

```mermaid
stateDiagram-v2
    [*] --> Pending: connection request accepted
    Pending --> Declined: owner declines
    Pending --> Expired: not decided in time
    Pending --> Approved: owner approves
    Approved --> Expired: initial access token unused
    Approved --> Active: credentials provisioned via RFC 7591
    Active --> Active: rotate secret
    Active --> Suspended: target policy, e.g. assurance below minimum
    Suspended --> Active: target lifts suspension
    Active --> Revoked: either party revokes
    Suspended --> Revoked: either party revokes
    Declined --> [*]
    Expired --> [*]
    Revoked --> [*]
```

---

## Notes and open items

- The flow shows a **one-directional** connection: the buyer becomes an OAuth client of the supplier. Where two parties exchange in both directions, the flow runs twice, independently.
- **Options A, B and C** are open WG decisions. The natural pairings are A1 or A2 with **B2 + C1** (no keys anywhere) or with **B1 + C2** (one key pair per SP, used for both queries and requests). Each combination costs an SP three new PACT-specific methods (§10.1).
- Under Option C1 the buyer must itself be discoverable through SP-B. Under C2 it need not be.
- How parties learn about Directory changes, new versions, and counterparty revocation is parked item P1.
- Delegated-role credentials (an SP proving it is the authorised exchange agent for a customer) would appear in Phases 0b and 3. They are a named extension, not V1.
