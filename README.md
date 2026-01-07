# Project FACET (FTAI + TIF)
**Facility Agent Control & Exchange Transport**

Project FACET is a vendor-neutral standards suite for connecting AI agents to real-world facilities—securely, audibly, and with deterministic safety enforcement.

FACET is designed for **data centers**, but is intentionally generalized for **any facility** with operational technology (OT) and/or building automation systems: factories, warehouses, hospitals, campuses, labs, telecom huts, microgrids, water plants, retail facilities, and more.

FACET defines two primary specifications:

1) **TIF** — *“This Is the Facility”*  
   A structured, LLM-optimized facility definition that describes:
   - what the facility contains (assets, zones, systems),
   - how it is connected (topology),
   - what can be observed (metrics and events),
   - what can be controlled (actions),
   - and what constraints apply (authority tiers, approvals, envelopes, lockouts, windows).

2) **FTAI** — *Facility-to-Agent Interface*  
   A hardened API boundary that:
   - serves TIF,
   - provides real-time and historical telemetry/events/state,
   - enables advisory and case creation,
   - implements action request + approval + (optional) execution,
   - and enforces safety and authority as the **deterministic control gate**.

> **Core Principle:** The AI agent is not trusted.  
> FACET requires that the facility-side FTAI gateway enforces safety constraints, approvals, rate limits, and authority. Agents must use the FTAI tooling rather than “direct control” of facility systems.

---

## Table of Contents
- [Why FACET exists](#why-facet-exists)
- [What FACET enables](#what-facet-enables)
- [Architecture overview](#architecture-overview)
- [FACET components](#facet-components)
  - [TIF: This Is the Facility](#tif-this-is-the-facility)
  - [FTAI: Facility-to-Agent Interface](#ftai-facility-to-agent-interface)
- [Conformance profiles](#conformance-profiles)
- [Security model](#security-model)
- [Authority model](#authority-model)
- [Data model overview](#data-model-overview)
- [Quick start (implementers)](#quick-start-implementers)
- [Quick start (agent developers)](#quick-start-agent-developers)
- [Key workflows](#key-workflows)
  - [1) Discovery and onboarding](#1-discovery-and-onboarding)
  - [2) Reading state and events](#2-reading-state-and-events)
  - [3) Publishing advisories](#3-publishing-advisories)
  - [4) Creating and updating cases](#4-creating-and-updating-cases)
  - [5) Requesting approvals and executing actions](#5-requesting-approvals-and-executing-actions)
- [Action safety and policy enforcement](#action-safety-and-policy-enforcement)
- [Streaming (SSE)](#streaming-sse)
- [Audit and compliance](#audit-and-compliance)
- [Versioning and compatibility](#versioning-and-compatibility)
- [Recommended implementation patterns](#recommended-implementation-patterns)
- [Reference examples](#reference-examples)
  - [cURL examples](#curl-examples)
  - [Pseudo-code examples](#pseudo-code-examples)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## Why FACET exists

Facilities are increasingly instrumented, but automation and operational intelligence remain fragmented across:
- vendor-specific OT protocols,
- disconnected dashboards,
- inconsistent naming and topology models,
- and risky/opaque “AI” integrations.

FACET standardizes **facility representation** and **facility-agent interaction** so that:
- any compliant agent can understand any compliant facility,
- facilities can adopt AI safely without surrendering control,
- and outcomes are audit-ready and policy-compliant.

FACET is particularly focused on avoiding failure modes common to naive “agent controls everything” designs:
- hallucinated commands,
- unauthorized actions,
- unbounded setpoint changes,
- action execution without preconditions,
- lack of audit trails,
- and “AI did something and nobody knows why.”

---

## What FACET enables

FACET enables an AI agent to:
- **Read** facility state, metrics, and events in a normalized form
- **Understand** facility layout/topology and system dependencies
- **Explain** conditions and recommend actions with evidence and confidence
- **Publish advisories** to humans through portals/kiosks/mobile
- **Open and manage cases** to group incidents and investigations
- **Request actions** through a safe gate (FTAI) with approvals and constraints
- **Execute actions** only when explicitly approved and enabled

FACET also enables facilities to:
- “plug in” multiple agents without bespoke integrations,
- keep all control logic policy-bound and auditable,
- offer read-only or approval-only modes to reduce risk,
- and expand capabilities over time.

---

## Architecture overview

FACET is typically deployed as a **gateway** between the facility systems and AI agents:

```
 +-------------------+       +----------------------+
 |    AI Agent(s)    |       |  Human Interfaces    |
 | (LLM + Tool Use)  |       | (Portal/Kiosk/Mobile)|
 +---------+---------+       +----------+-----------+
           |                            |
           |  FTAI API (read/write)     |
           v                            v
 +--------------------------------------------------+
 |             FTAI Gateway (FACET)                  |
 |  - AuthN/AuthZ (OIDC/mTLS)                        |
 |  - Policy Engine (authority, envelopes, windows)  |
 |  - Approvals + Audit Logs                         |
 |  - TIF generator + versioning                     |
 |  - Adapters to facility systems                   |
 +---------------------+----------------------------+
                       |
                       | OT/IT adapters (vendor specific)
                       v
 +--------------------------------------------------+
 | Facility Systems (BMS/EPMS/DCIM/ACS/VMS/SCADA etc)|
 +--------------------------------------------------+
```


