# PACT Identity Management — Design Decisions Brief

*For the tech tesm and potential members of the PACT Technology Working Group · 24 June 2026 · Draft for discussion*

## Purpose

PACT is drafting an **addendum to the PACT Technical Specifications V3** that adds Identity Management to the network: the ability for organisations to **register, be found, connect, and authenticate at scale**. This brief sets out the design positions PACT is proposing to take into the Technology Working Group when it reopens, so partners — GLEIF in particular — can react before drafting hardens. None of this changes the existing PCF data model or exchange protocol; it adds a layer *around* them.

## The problem

The V3 protocol deliberately leaves identity and trust out of scope (§5.3): a host system authenticates clients with OAuth 2.0, but the specification says nothing about how two organisations *find* each other or how they come to *trust each other's credentials* in the first place. Today that is done manually and bilaterally — workable for pilots, a hard ceiling for a network already spanning ~50 solutions, ~180 corporates, and thousands of suppliers. Identity Management removes that ceiling.

## The approach in one paragraph

A **federated, DNS-like directory** of organisations identified by their **LEI**, with hierarchical roots (a global root, optional regional roots for national networks, and solution-provider sub-directories). Discovery is **opt-in and privacy-controlled**. Once two participants find each other, they establish trust **directly, peer-to-peer** — PACT is never in the credential or data path — verifying identity with a **vLEI** where available and automatically provisioning the **OAuth credentials the existing protocol already expects**. The base-spec data model and token flow are untouched.

## Key design decisions

| Topic | Decision |
|-------|----------|
| **Ambition** | Launch as a **low-friction directory** (self-asserted identities, no onboarding gate), with a **verified tier following soon**. Assurance is an *attribute* of an identity, never a barrier to joining. |
| **Identifier** | The **LEI is the recommended canonical identifier**, carried inside the protocol's existing open `companyIds` field. **No new PACT-specific identifier** — organisations use what they already have. LEI is recommended, not required, in V1. |
| **Hierarchy** | Identity is per **operating entity**; parent/subsidiary relationships are **read from LEI Level-2 data**, not maintained by PACT. (No hierarchy until an entity has an LEI — which also makes the LEI worth getting.) |
| **Verification** | **vLEI is the verified tier**, in-scope but optional. It proves organisational identity at connection time; **entity-level** verification ships first, **delegated-role** credentials (e.g. a solution provider proving it is the authorised exchange agent for a customer) are the named next step. Natural-person credentials are out of scope. |
| **Authentication** | **vLEI verifies *identity*; OAuth secures the *runtime* exchange.** The proven OAuth flow is left in place so the wider, lower-maturity community is not blocked; vLEI is layered on top for trust establishment. |
| **Discovery** | A **thin central index** (identifier → endpoint, or → a delegated solution-provider directory) plus **federated** provider endpoints. **Regional roots** are allowed and any root can be tied to a parent. PACT never replicates the open LEI reference data. |
| **Privacy** | Two independent controls: you can be **resolvable** to members who already know your LEI (default on) without being **listable** in open search (default off, opt-in). **Connection always requires approval** — being found never means being connected. |
| **Credential exchange** | **Peer-to-peer** after discovery. The trust/credential lifecycle (request → approve → rotate → revoke) is standardised; **OAuth 2.0 Dynamic Client Registration (RFC 7591)** is the recommended way to auto-issue credentials, leaving room for vLEI-native bindings later. |
| **Conformance** | Registration is **not** gated on conformance; a node's PACT **conformant-version(s)** are shown as an attribute so counterparties know what it can exchange. Suppliers without tooling use a provisional demo node as an on-ramp. |

## What this means for GLEIF

The architecture makes **LEI the backbone of network identity** and **vLEI the mechanism for verifiable trust and delegated authority** — directly the collaboration discussed on 4 and 17 June. Concretely, it depends on: (1) LEI coverage across PACT's ecosystem, where the joint **gap analysis** and a **negotiated low-cost / anchor-absorbed issuance route** are the critical enablers, especially for SMEs and emerging-market suppliers; and (2) maturing **vLEI** tooling and issuers for the verified tier. PACT acts as a design partner and a scalable, real-economy use case; GLEIF provides the identity root of trust PACT explicitly does not want to build itself.

## What we are asking the Working Group

To pressure-test these positions — in particular the **LEI-as-recommended-identifier** stance, the **peer-to-peer credential model with RFC 7591**, and the **two-control privacy model** — ahead of a first draft addendum.

## Timeline

Draft addendum **June/July** → Technology Working Group reopens **August** → initial testers **October** → final release **target early 2027**.

## Still open

Policy-based auto-approval of connections; the exact discovery API methods and schemas; the precise LEI URN encoding; whether the network should mandate audit logging; and the "invite a non-member supplier to join" journey (likely a platform feature rather than normative spec).

---

*Companion materials: the full annotated addendum structure and the resolved-decisions log; a first draft of the Identity Model section (§3). Available on request.*
