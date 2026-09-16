# PACT Identity Management — Design Decisions Brief

*For the tech team and members of the PACT Technology Working Group · Updated 16 September 2026 (first issued 24 June 2026) · Draft for discussion*

## Purpose

PACT is drafting an **addendum to the PACT Technical Specifications V3** that adds Identity Management to the network: the ability for organisations to **be found, connect, and authenticate at scale** across software platforms. This brief sets out the design positions behind the draft, updated with the decisions the Technology Working Group took on **2 September 2026**, and the decisions still open. None of this changes the existing PCF data model or exchange protocol; it adds a layer *around* them.

## The problem

The V3 protocol deliberately leaves identity and trust out of scope (§5.3): a host system authenticates clients with OAuth 2.0, but the specification says nothing about how two organisations *find* each other or how they come to *trust each other's credentials* in the first place. Today that is done manually and bilaterally. That works for pilots but puts a hard ceiling on a network already spanning ~50 solutions, ~180 corporates, and thousands of suppliers. Identity Management removes that ceiling.

## The approach in one paragraph

PACT runs a **central directory of solution-provider discovery endpoints**. It holds no buyer or supplier data, which stays with each solution provider. A solution provider is listed once it has **proven ownership of its domain and passed an expanded PACT conformance test** covering discovery. A party looking for a counterparty queries the listed discovery services by a company identifier and learns which node to contact, and how strongly the provider vouches for that company's identity. Parties then establish trust **directly, peer-to-peer**, since PACT is never in the credential or data path, and automatically provision the **OAuth credentials the existing protocol already expects**. **Email or domain ownership** is sufficient identity assurance for v1 where both parties agree; **LEI and verifiable credentials (vLEI)** are optional layers ready to switch on as the stakes of fraud rise.

## Key design decisions

| Topic | Decision | Since |
|-------|----------|-------|
| **Directory** | **Central directory of SP discovery endpoints.** No central index of buyers or suppliers. Every discovery service answers for its own customers. A company that runs its own node is listed as an *SP of one*. Regional directories are a possible later extension. | 2 Sept |
| **Operator identity & listing** | SP identity is verified through **domain ownership**, as part of an **expanded conformance test** covering the discovery endpoints. Non-conformant home-built solutions are not eligible for listing as they stand. V3 exchange conformance is shown as an attribute and does not gate listing. | 2 Sept |
| **Party identity** | **Email or domain ownership is sufficient** assurance for buyers and suppliers where both parties agree. It is verified and attested by each party's own solution provider. | 2 Sept |
| **LEI / vLEI** | **Optional layer on a generic verifiable-credentials architecture**, not mandatory in v1. Where held, an LEI enables a *registry-verified* assurance level and parent/subsidiary hierarchy from LEI Level-2 data. vLEI is the first credential profile. | 2 Sept (replaces "LEI recommended canonical identifier") |
| **Identifier** | Existing open **`companyIds`** URN array; discovery is keyed on it. **No new PACT-specific identifier.** | June |
| **Assurance** | An attribute, never a precondition: *self-asserted < email-verified < domain-verified < registry-verified (LEI) < credential-verified*. Each party sets the minimum it accepts. | Extended 16 Sept draft |
| **Privacy** | Being discoverable is **opt-in** and set per customer at its provider. An unknown identifier and a non-visible one return the same answer. **Connection always requires approval.** | June, tightened in 16 Sept draft |
| **Credential exchange** | **Peer-to-peer** after discovery. Request → approve → provision → rotate/revoke is standardised. **OAuth 2.0 Dynamic Client Registration (RFC 7591/7592)** issues the credentials. Approval is manual in v1. | June |
| **Simplicity** | A solution provider implements **three new PACT-specific API methods** (discovery, connection request, and a decision callback or status poll), plus standard OAuth registration endpoints. | 16 Sept draft |

## Decisions for the Working Group

Three design choices are drafted **side by side** for the WG to decide:

- **A — Who performs the fan-out.** The requester queries every discovery service itself (PACT publishes only a list), or the PACT directory brokers the query (one call for the requester, but PACT sees every lookup). *§5.5*
- **B — Who may query a discovery service.** Only listed providers, with signed requests (stops customer-list harvesting and needs a key pair per SP), or anyone, with a minimal answer (no keys, but open to harvesting). *§5.6*
- **C — How a connection request is authenticated.** The approval is sent only to the requester's published endpoint (no cryptography), or the request is signed and the requester polls (no need to be discoverable). *§6.4*

B and C are easiest to decide together: **B2 + C1** means no keys anywhere; **B1 + C2** means one key pair per SP used for both.

## What this means for GLEIF

The architecture no longer depends on LEI coverage or GLEIF's participation for v1, which answers the dependency concern raised in the Working Group. The LEI and vLEI remain the **strongest assurance layer** and the basis for hierarchy and delegated authority (an SP proving it is the authorised exchange agent for a customer). That is where a GLEIF collaboration adds value: coverage in PACT's supply chains, a low-cost issuance route, and maturing vLEI tooling. It is valuable, and no longer on the critical path.

## Timeline

Draft addendum **June/July** → Technology Working Group reopened **August** → draft sections §3–§6 and §10 for WG review **September** → initial testers **October** → final release **target early 2027**.

## Still open

Parked on 2 September: how providers and parties are **notified of directory updates and new-version roll-outs**; whether an **invoice-based identifier** (VAT or Chamber of Commerce number) is practical, to be tested with a real enterprise supply chain; and whether **registry access is restricted to conformant providers or public** (largely decision B). Also open: policy-based auto-approval of connections; whether a normative hook is needed for **SP portability**; and whether the network should mandate audit logging.
