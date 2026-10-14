# PACT Identity Management — Technical Specification (Working Drafts)

Background documentation and a draft technical specification for **PACT Identity Management (IM)** — an addition to the existing [PACT Technical Specifications for PCF Data Exchange](https://github.com/wbcsd/data-exchange-protocol) (V3). The base specifications define a data model and a peer-to-peer REST protocol for exchanging Product Carbon Footprint (PCF) data between companies; this work adds the missing **identity, discovery, and trust** layer so that participants can find each other and establish connections at scale.

IM is being drafted as an **addendum to V3** covering two coupled topics on one shared identity model:

- **Node Discoverability** — letting participants find each other, with privacy controls.
- **Automated Credential Exchange** — automating the credential/trust bootstrap that V3 §5.3 explicitly leaves out of scope.

> **Status:** early-stage drafting and brainstorming. These documents are working drafts, not ratified specifications. The intent is to evolve them into a specification in `wbcsd/data-exchange-protocol`. Design positions here are inputs to the PACT Technology Working Group, not final decisions.

## Documents

Suggested reading order is top to bottom — from background, to summary, to structure, to draft spec text.

| File | Role |
|------|------|
| [`PACT-IM-Decisions-Brief.md`](PACT-IM-Decisions-Brief.md) | **2-page decisions brief.** A circulatable summary of the proposed architecture and the key design decisions, with a section on what it means for GLEIF. For the tech team and prospective Working Group members. |
| [`PACT-IM-Addendum-Outline.md`](PACT-IM-Addendum-Outline.md) | **Annotated addendum structure + decisions log.** The full section-by-section structure of the planned addendum (mirroring the V3 spec conventions), and §12 the consolidated design-decisions log — including the 2 September 2026 WG decisions (§12.1), editorial positions awaiting confirmation (§12.2), and open options A–C (§12.3). The working master for what the spec will contain. |
| [`PACT-IM-Addendum-Section-3-Identity-Model.md`](PACT-IM-Addendum-Section-3-Identity-Model.md) | **Draft spec text — §3 Identity Model (v0.2).** Normative prose (RFC 2119 keywords): entities, operators, nodes and discovery services; `companyIds` identifiers with LEI optional; the extended assurance ladder (self-asserted → email → domain → LEI → verifiable credential) attested by each party's own provider; LEI Level-2 hierarchy; and the identity-verification options side by side. |
| [`PACT-IM-Addendum-Section-4-Registration.md`](PACT-IM-Addendum-Section-4-Registration.md) | **Draft spec text — §4 Registration (v0.2).** The PACT Directory of discovery services: conformance-gated listing with domain-ownership verification, listing attributes, suspension and delisting; and what a discovery service must publish about the entities and nodes it fronts. |
| [`PACT-IM-Addendum-Section-5-Discovery.md`](PACT-IM-Addendum-Section-5-Discovery.md) | **Draft spec text — §5 Node Discoverability (v0.1).** Directory-vs-DNS rationale, resolution procedure, discovery answer and visibility, with open decisions **A** (fan-out) and **B** (who may query) set out side by side. |
| [`PACT-IM-Addendum-Section-6-Credential-Exchange.md`](PACT-IM-Addendum-Section-6-Credential-Exchange.md) | **Draft spec text — §6 Automated Credential Exchange (v0.1, first pass).** Connection request, approval and policy, RFC 7591/7592 provisioning, rotation, suspension and revocation, with open decision **C** (request authentication) side by side. |
| [`PACT-IM-Addendum-Section-10-API.md`](PACT-IM-Addendum-Section-10-API.md) | **Draft API — §10 (v0.1).** Draft endpoints written for SP implementers, a method count against the "handful of new API methods" guardrail, error codes, an inline OpenAPI 3.1 fragment, and an implementation checklist for solution providers. |
| [`PACT-IM-Flow-Discover-Verify-Connect.md`](PACT-IM-Flow-Discover-Verify-Connect.md) | **Flow diagram (non-normative, v0.2).** End-to-end Mermaid sequence — SP listing, supplier onboarding, discovery, assurance check, connect, auto-provision OAuth credentials via RFC 7591, exchange — showing options A and C as alternatives, plus a connection-lifecycle state view. |

## Architecture in one line

A central PACT directory of **conformance-verified solution-provider discovery endpoints** (no central buyer/supplier data); parties are discovered by `companyIds` through their own provider, with **email/domain assurance** attested by that provider and **LEI/vLEI as optional stronger layers**; once found, nodes establish trust **peer-to-peer** and auto-provision the existing **OAuth** exchange credentials via **RFC 7591** — leaving the base-spec data model and token flow untouched. *(Updated after the 2 September 2026 Working Group session; see the outline §12.1.)*

## Related resources

- Base specification: <https://github.com/wbcsd/data-exchange-protocol> · published at <https://docs.carbon-transparency.org/tr/data-exchange-protocol/latest/>
- PACT Network Services platform (reference implementation): <https://github.com/wbcsd/pact-directory>
- PACT: <https://www.carbon-transparency.org>

## Contact

Editor

- Gertjan Schuurmans (schuurmans@wbcsd.org) / pact-support@wbcsd.org.
