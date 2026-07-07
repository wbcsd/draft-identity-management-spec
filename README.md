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
| [`PACT-Identity-Management-Orientation-Note.md`](PACT-Identity-Management-Orientation-Note.md) | **Background & orientation.** What exists today (V3 data model, OAuth auth, the §5.3 gap), the history of the IM initiative, current workstreams, open tensions, and the third-party standards landscape (GLEIF LEI/vLEI, EUDI, VC/DID). Start here for context. |
| [`PACT-IM-Decisions-Brief.md`](PACT-IM-Decisions-Brief.md) | **2-page decisions brief.** A circulatable summary of the proposed architecture and the key design decisions, with a section on what it means for GLEIF. For the tech team and prospective Working Group members. |
| [`PACT-Identity-Management-Addendum-Outline.md`](PACT-Identity-Management-Addendum-Outline.md) | **Annotated addendum structure + decisions log.** The full section-by-section structure of the planned addendum (mirroring the V3 spec conventions), and §12 the consolidated, resolved design-decisions log. The working master for what the spec will contain. |
| [`PACT-IM-Addendum-Section-3-Identity-Model.md`](PACT-IM-Addendum-Section-3-Identity-Model.md) | **Draft spec text — §3 Identity Model.** First section drafted as normative prose (RFC 2119 keywords): actors, entities/operators/nodes, LEI-in-`companyIds` identifiers, assurance levels, and LEI Level-2 hierarchy. |

## Architecture in one line

A federated, DNS-like directory of **LEI-identified** entities with hierarchical (global / regional / solution-provider) roots; discovery is opt-in and privacy-controlled; once two nodes find each other they establish trust **peer-to-peer**, verifying identity via **vLEI** (optional) and auto-provisioning the existing **OAuth** exchange credentials via **RFC 7591** — leaving the base-spec data model and token flow untouched.

## Related resources

- Base specification: <https://github.com/wbcsd/data-exchange-protocol> · published at <https://docs.carbon-transparency.org/tr/data-exchange-protocol/latest/>
- PACT Network Services platform (reference implementation): <https://github.com/wbcsd/pact-directory>
- PACT: <https://www.carbon-transparency.org>
- GLEIF LEI/vLEI: <https://www.gleif.org>

## Contact

- Gertjan Schuurmans (schuurmans@wbcsd.org) / pact-support@wbcsd.org.
