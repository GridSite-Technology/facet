# Project FACET (Facility Agent Control & Exchange Transport)

Project **FACET** is a vendor-neutral standards suite for connecting AI agents to real-world facilities—securely, audibly, and with deterministic safety enforcement.

FACET is designed for **data centers** but intentionally generalized for **any facility** with operational technology (OT), building automation, or mission-critical infrastructure: factories, warehouses, hospitals, campuses, labs, telecom huts, microgrids, water plants, and more.

FACET defines two primary specifications:

- **TIF** — *“This Is the Facility”*  
  A structured, LLM-optimized facility definition that describes:
  - what the facility contains (assets, zones, systems),
  - how it is connected (topology),
  - what can be observed (metrics and events),
  - what can be controlled (actions),
  - and what constraints apply (authority tiers, approvals, envelopes, lockouts, windows).

- **FTAI** — *Facility-to-Agent Interface*  
  A hardened API boundary that:
  - serves TIF,
  - provides real-time and historical telemetry/events/state,
  - enables advisory and case creation,
  - implements action request + approval + (optional) execution,
  - and enforces safety and authority as the **deterministic control gate**.

> **Core principle:** The AI agent is not trusted.  
> FACET requires that the facility-side FTAI gateway enforces safety constraints, approvals, rate limits, and authority. Agents must use FTAI tooling rather than connecting directly to facility control systems.

---

## Table of Contents

- [Why FACET exists](#why-facet-exists)
- [What FACET enables](#what-facet-enables)
- [High-level architecture](#high-level-architecture)
- [Key concepts](#key-concepts)
  - [TIF](#tif)
  - [FTAI](#ftai)
  - [Authority tiers](#authority-tiers)
  - [Constraints and safety envelopes](#constraints-and-safety-envelopes)
  - [Approvals and execution](#approvals-and-execution)
  - [Advisories and cases](#advisories-and-cases)
  - [Adapters](#adapters)
- [Conformance profiles](#conformance-profiles)
- [Repository layout](#repository-layout)
- [Quick start (facility implementers)](#quick-start-facility-implementers)
- [Quick start (agent developers)](#quick-start-agent-developers)
- [Reference workflows](#reference-workflows)
  - [1) Discovery and onboarding](#1-discovery-and-onboarding)
  - [2) Read state, metrics, and events](#2-read-state-metrics-and-events)
  - [3) Publish advisories](#3-publish-advisories)
  - [4) Create and manage cases](#4-create-and-manage-cases)
  - [5) Request approvals and execute actions](#5-request-approvals-and-execute-actions)
- [Security model](#security-model)
- [Policy engine requirements](#policy-engine-requirements)
- [Audit and compliance](#audit-and-compliance)
- [Streaming (SSE)](#streaming-sse)
- [Versioning and compatibility](#versioning-and-compatibility)
- [Recommended implementation patterns](#recommended-implementation-patterns)
- [Reference examples](#reference-examples)
  - [cURL examples](#curl-examples)
  - [Implementation pseudo-code](#implementation-pseudo-code)
- [FAQ](#faq)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why FACET exists

Facilities are increasingly instrumented, but automation and operational intelligence remain fragmented across:

- vendor-specific OT protocols and point lists,
- disconnected dashboards and “single pane of glass” attempts,
- inconsistent naming, topology models, and asset identity,
- risky/opaque “AI” integrations without enforcement gates,
- inadequate audit trails for regulated customers.

FACET standardizes **facility representation** (TIF) and **facility–agent interaction** (FTAI) so that:

- any compliant agent can understand any compliant facility,
- facilities can adopt AI safely without surrendering control,
- outcomes are explainable and audit-ready,
- deployments can start read-only and progressively enable higher authority.

FACET specifically prevents failure modes common to naive “agent controls everything” designs:

- hallucinated commands against non-existent equipment,
- unauthorized actions outside policy scope,
- unbounded setpoint changes,
- execution without validating preconditions,
- lack of auditability (“AI did something, nobody knows why”).

---

## What FACET enables

FACET enables an AI agent to:

- **Read** facility state, metrics, and events in a normalized form.
- **Understand** facility layout/topology and system dependencies.
- **Explain** conditions and recommend actions with evidence and confidence.
- **Publish advisories** to humans through portals, kiosks, and mobile apps.
- **Open and manage cases** to group incidents and investigations.
- **Request actions** through a safe gate (FTAI) with deterministic validation.
- **Execute actions** only when explicitly approved and when execution is enabled.

FACET enables facilities to:

- “plug in” multiple agents without bespoke integrations,
- keep all control logic policy-bound and auditable,
- offer read-only or approval-only modes to reduce risk,
- expand capabilities over time with explicit governance.

---

## High-level architecture

FACET is typically deployed as a hardened gateway between facility systems and clients (agents + humans):

```text
+-------------------+             +----------------------+
|    AI Agent(s)    |             |  Human Interfaces    |
| (LLM + Tool Use)  |             | (Portal/Kiosk/Mobile)|
+---------+---------+             +----------+-----------+
          |                                    |
          |  FTAI API (read/write)             |
          v                                    v
+--------------------------------------------------------+
|                 FTAI Gateway (FACET)                   |
|  - AuthN/AuthZ (OIDC/mTLS)                             |
|  - Policy Engine (authority, envelopes, windows)       |
|  - Approvals + Audit Logs                              |
|  - TIF generator + signing/versioning                  |
|  - Streaming (SSE)                                     |
|  - Adapters (BMS/EPMS/ACS/VMS/DCIM/SCADA/etc.)         |
+--------------------------+-----------------------------+
                           |
                           | Vendor / site-specific connectors
                           v
+--------------------------------------------------------+
| Facility Systems (BMS/EPMS/DCIM/ACS/VMS/SCADA/IoT/etc.) |
+--------------------------------------------------------+
```

**Important:** The agent never connects directly to BMS/SCADA/ACS/etc.  
The FTAI Gateway is the enforcement point.

---

## Key concepts

### TIF

**TIF** is the facility’s structured identity and capability map. A compliant TIF includes (at minimum):

- `meta`: versioning, hashes, signatures
- `facility`: identity, location hint, timezone, contacts
- `zones`: zones and adjacency
- `systems`: domains and system boundaries
- `assets`: typed assets and relationships
- `topology`: graphs and ASCII representations
- `observability`: metrics and events catalogs with semantics
- `controls`: action definitions (schemas, envelopes, approvals)
- `constraints`: windows, lockouts, rate limits
- `authority`: AI tiers, allowed operations, role mappings
- `runbooks`: runbook references, triggers, step contracts
- `glossary`: terms, abbreviations, local naming conventions

TIF is served via `GET /v1/facility/tif` using content negotiation:

- `application/tif+yaml;version=1`
- `application/tif+json;version=1`
- `text/tif+plain;version=1` (deterministic rendering from canonical)

**Recommended:** TIF responses include:
- `ETag`
- `X-Canonical-Hash` (SHA-256 hash)
- `X-Generated-At`
- optional signatures (e.g., Ed25519)

---

### FTAI

**FTAI** is the facility-to-agent interface and enforcement boundary. It defines:

- **Discovery:** server capabilities and links
- **Definition:** TIF retrieval and caching semantics
- **Observability:** state, metrics, events (historical + streaming)
- **Operations:** advisories and cases for human coordination
- **Control:** action request → approval → (optional) execution
- **Governance:** effective constraints and authority exposure
- **Auditability:** deterministic records of access and decisions

FTAI is intentionally layered:
- Northbound: agents and human UIs
- Southbound: adapters to vendor and site systems

---

### Authority tiers

FACET models authority as explicit **tiers** (configured in TIF, enforced by FTAI). A typical pattern:

- **Tier 0 — Observe:** read-only data access
- **Tier 1 — Recommend:** can publish advisories/cases/work order drafts
- **Tier 2 — RequestApproval:** can request actions (creates approval artifacts)
- **Tier 3 — ExecuteApproved:** can execute actions only after approval (if enabled)
- **Tier 4+ — Reserved:** implementation-specific (rare, high governance)

The facility exposes the **effective tier** (runtime) via:

- `GET /v1/facility/constraints/effective`

FTAI MUST enforce the effective tier regardless of agent behavior.

---

### Constraints and safety envelopes

FACET treats facility controls like industrial controls: every action must be bounded.

FTAI MUST enforce (as applicable):

- **Parameter schemas** (type, required fields)
- **Envelopes** (min/max setpoints, allowed states)
- **Rate limits** (max delta per time window, max frequency)
- **Lockouts** (global/system/asset disablement)
- **Windows** (maintenance/change)
- **Preconditions** (live checks required before approval/execution)
- **Execution enablement** (global kill switch)

Constraints can be static (declared in TIF) and/or dynamic (effective overrides).

---

### Approvals and execution

FACET separates *requesting* an action from *executing* it.

- **Action request** creates a validated intent and may produce an approval artifact.
- **Approval** is typically performed by a human with required roles.
- **Execution** is optional; if enabled, FTAI MUST re-check preconditions at execution time.

This design supports deployments where:
- agents recommend actions only,
- agents request approvals but never execute,
- or mature sites allow execution of certain approved actions.

---

### Advisories and cases

FACET standardizes how an agent communicates operationally:

- **Advisories:** real-time notifications, recommendations, updates, optimization opportunities
- **Cases:** containers for incidents and investigations; group events, metrics, approvals, notes

Facilities can drive consistent operator workflows via:
- required acknowledgements,
- audience targeting by role,
- correlation IDs linking advisories, cases, and actions.

---

### Adapters

FACET does not replace low-level OT protocols (Modbus, BACnet, OPC UA, SNMP, etc.). Instead:

- FTAI defines a normalized northbound interface.
- The facility implements adapters southbound to existing systems.
- The policy engine remains facility-side regardless of adapter details.

Recommended adapter responsibilities:
- map FACET action IDs to vendor commands,
- implement precondition checks and read-back verification,
- emit events and telemetry in normalized form,
- propagate correlation IDs for end-to-end traceability.

---

## Conformance profiles

FACET supports progressive adoption through conformance profiles:

### 1) `FTAI-Profile-RO` (Read-Only)

Includes:
- discovery
- TIF
- state
- metrics
- events
- (optional) streaming

Excludes:
- action request, approvals, execution

Recommended for:
- early adoption
- external auditors and customers
- “AI assistant” that observes and explains

---

### 2) `FTAI-Profile-Approve` (Human-in-the-loop)

Includes RO plus:
- advisories and cases
- action requests (creates approval artifacts)
- approval workflows (approve/deny)

Execution is optional and typically disabled initially.

Recommended for:
- production deployments
- regulated environments
- enterprise customers
- high-consequence systems

---

### 3) `FTAI-Profile-Exec` (Execution-enabled)

Includes Approve plus:
- action execution endpoint for approved actions
- required precondition re-checks at execution time

Recommended for:
- mature operations
- limited, vetted action sets (narrow scope)
- strong change management and incident response maturity

---

## Repository layout

A typical FACET repository layout (recommended):

```text
.
├── README.md
├── LICENSE
├── openapi/
│   └── ftai-v1.yaml
├── tif/
│   ├── spec/
│   │   └── tif-v1.md
│   ├── schema/
│   │   └── tif-v1.schema.json
│   └── examples/
│       ├── tif-example-datacenter.yaml
│       └── tif-example-factory.yaml
├── ftai/
│   ├── spec/
│   │   └── ftai-v1.md
│   ├── examples/
│   │   ├── events.jsonl
│   │   ├── state.json
│   │   └── action-requests.jsonl
│   └── conformance/
│       ├── checklist.md
│       └── fixtures/
│           └── ...
├── docs/
│   ├── security-considerations.md
│   ├── profiles.md
│   ├── governance.md
│   └── glossary.md
└── reference/
    ├── gateway-skeleton/   # optional reference implementation (not required)
    └── tools/             # conformance validators, renderers, test clients
```

---

## Quick start (facility implementers)

This section describes how to implement an FTAI gateway and generate TIF.

### Step 1 — Establish stable identifiers

Assign immutable identifiers:
- `facility_id`
- `zone_id`
- `asset_id`
- `system_id`
- `redundancy_group_id`
- `action_id`
- event codes (`code`)

Avoid renaming IDs after deployment. If naming must change, version and alias explicitly.

---

### Step 2 — Generate canonical TIF

Your TIF generator should source from:
- asset inventory / DCIM / CMDB
- BMS/EPMS naming + point lists
- topology model (graphs for power/cooling/security)
- action catalog and parameter schemas
- authority and approval policies
- maintenance/change windows
- lockouts and constraints
- runbook references and triggers

Minimum outputs:
- canonical: `application/tif+yaml` or `application/tif+json`
- deterministic rendering: `text/tif+plain`

Strongly recommended:
- `X-Canonical-Hash` (SHA-256)
- signatures (e.g., Ed25519)
- ETag support for caching

---

### Step 3 — Implement read plane endpoints

Implement:
- `GET /v1/` discovery
- `GET /v1/facility/tif`
- `GET /v1/facility/state`
- `GET /v1/facility/events`
- `POST /v1/facility/metrics/query`
- optional: streaming endpoints (`/v1/stream/events`, `/v1/stream/state`)

---

### Step 4 — Implement policy engine (required for control plane)

Even if execution is disabled, your policy engine MUST be able to validate requests and create approvals.

Mandatory enforcement checks:
- caller authentication and scope validation
- action existence (must be defined by TIF controls / effective catalog)
- parameter schema validation
- envelope checks
- rate limit checks
- lockout and window checks
- precondition evaluation (live checks)
- approval policy determination

---

### Step 5 — Implement approvals

Implement:
- `GET /v1/approvals`
- `GET /v1/approvals/{approval_id}`
- `POST /v1/approvals/{approval_id}/approve`
- `POST /v1/approvals/{approval_id}/deny`

Approval MUST validate:
- approver identity
- required roles
- approval not expired
- action request parameters unchanged (or require re-approval)

---

### Step 6 — Implement execution (only for Exec profile)

If you support execution:
- gate with `execution_enabled` (global kill switch)
- re-check preconditions at execution time
- ensure adapter confirms read-back / state verification when possible
- emit events for execution start/finish/failure
- record audit records for every step

---

## Quick start (agent developers)

### Step 1 — Discover capabilities

Call:

- `GET /v1/` or `GET /v1/.well-known/ftai`

Read:
- conformance profile(s)
- streaming support
- whether execution is enabled
- endpoint links

---

### Step 2 — Fetch and cache TIF

Call:

- `GET /v1/facility/tif`

Cache by:
- `ETag` and/or `X-Canonical-Hash`

Revalidate via:
- `If-None-Match: <ETag>`

---

### Step 3 — Read state and subscribe to events

- `GET /v1/facility/state`
- subscribe (if enabled): `GET /v1/stream/events`

Use events to decide which deeper queries to run; avoid “pull everything constantly.”

---

### Step 4 — Observe constraints before proposing actions

Always call (or keep fresh):

- `GET /v1/facility/constraints/effective`

Especially before requesting actions, because:
- lockouts may be active,
- execution may be disabled,
- query and action limits may be tightened during incidents,
- authority tiers may be reduced.

---

### Step 5 — Request actions only through `/actions/request`

Never invent actions. Only request `action_id` values declared by:
- TIF controls, or
- the effective actions catalog (if exposed)

Always include:
- justification summary
- evidence references (metrics/events/document pointers)
- correlation ID (case/work order) if applicable

---

## Reference workflows

### 1) Discovery and onboarding

```text
1. GET /v1/            -> capabilities, endpoints, profiles
2. GET /v1/facility/tif -> facility definition (cache by ETag/hash)
3. GET /v1/facility/constraints/effective -> runtime authority and limits
4. GET /v1/facility/state -> current state snapshot
5. (optional) SSE subscribe: /v1/stream/events
```

---

### 2) Read state, metrics, and events

- Read snapshot:
  - `GET /v1/facility/state`

- Query events:
  - `GET /v1/facility/events?from=...&to=...&severity=...`

- Query metrics:
  - `POST /v1/facility/metrics/query`

---

### 3) Publish advisories

- Create:
  - `POST /v1/advisories`

- List:
  - `GET /v1/advisories`

- Acknowledge:
  - `POST /v1/advisories/{advisory_id}/ack`

---

### 4) Create and manage cases

- Create:
  - `POST /v1/cases`

- List:
  - `GET /v1/cases`

- Update:
  - `PATCH /v1/cases/{case_id}`

Cases should link:
- event IDs
- asset IDs
- zone IDs
- evidence links
- action request IDs / approval IDs

---

### 5) Request approvals and execute actions

**Request action** (agent):

- `POST /v1/actions/request`

FTAI will return one of:
- `DENIED` (with policy reasons)
- `PENDING_APPROVAL` (approval created)
- `EXECUTED` (only if allowed and enabled)
- `ACCEPTED` (accepted but not executed; implementation-specific)

**Approve / deny** (human):

- `POST /v1/approvals/{approval_id}/approve`
- `POST /v1/approvals/{approval_id}/deny`

**Execute approved action** (agent or automation, if enabled):

- `POST /v1/actions/{action_request_id}/execute`

FTAI MUST:
- re-check preconditions
- refuse execution if conditions changed
- record audit outcomes

---

## Security model

FACET is designed for high-consequence systems and assumes adversarial conditions.

### Authentication
Supported patterns:
- OAuth2 client credentials (service principals for agents and services)
- mutual TLS (OT boundary common)

### Authorization
FTAI enforces least privilege using:
- OAuth scopes for coarse capability
- policy engine checks for fine-grained, facility-specific permissions

### Mandatory auditing
FTAI MUST audit at least:
- TIF reads (who accessed definition)
- state/metrics/event reads (optional to sample, but recommended for high-trust)
- advisory/case writes
- action requests and decisions
- approvals and approvers
- action execution and results

### Operational safeguards
Recommended safeguards:
- global execution kill switch
- domain-level lockouts
- per-asset lockouts
- rate limiting per principal
- dedicated “break-glass” operator workflow
- token rotation and revocation

---

## Policy engine requirements

The policy engine is the heart of FACET control-plane safety.

### Deterministic decisioning
FTAI MUST produce deterministic outcomes given:
- TIF control definition,
- caller identity and scopes,
- effective constraints,
- and current facility state.

### Required checks
On `POST /actions/request`, FTAI MUST:
1. authenticate principal
2. validate scopes for action request
3. verify action exists in TIF (or effective catalog)
4. validate request schema and parameter types
5. enforce envelope constraints (min/max)
6. enforce rate limits (frequency and/or delta)
7. enforce lockouts (global/system/asset)
8. enforce windows (maintenance/change)
9. evaluate preconditions (live checks)
10. decide approval requirement (roles, minimum approvers)
11. emit audit record (request, decision, reasons)

On `POST /actions/{id}/execute`, FTAI MUST:
- re-check steps 5–9 at execution time
- refuse if anything fails (return `412 Precondition Failed` where appropriate)
- write an execution audit record
- emit execution events

### Example: policy enforcement outline

```text
ACTION REQUEST
  - validate caller + scopes
  - validate action_id exists
  - validate schema(parameters)
  - envelope check: setpoint in [min, max]
  - rate limit check: max_delta/time window
  - lockout/window check
  - precondition check: asset online, alarms stable, etc.
  - if requires approval -> create approval artifact
  - else -> (optional) execute if direct exec is allowed + enabled
```

---

## Audit and compliance

FACET is intended for environments requiring strong governance.

### Audit record characteristics
Audit records SHOULD be:
- append-only
- tamper-evident (hash chaining or WORM storage)
- retained per policy (customer/regulatory requirements)
- searchable by correlation ID and principal

### Audit query endpoint
If enabled:
- `GET /v1/audit/records`

Access SHOULD be restricted to:
- compliance roles
- security leadership
- authorized auditors

---

## Streaming (SSE)

FACET uses Server-Sent Events to provide firewall-friendly streaming:

- `GET /v1/stream/events`
- `GET /v1/stream/state`

SSE payloads are JSON objects emitted as `data:` lines.

Recommended stream behaviors:
- heartbeat events
- optional resume with `Last-Event-ID`
- backpressure strategies (drop, summarize, or require polling for replay)

---

## Versioning and compatibility

- FACET uses semantic versioning.
- Breaking changes require new major versions.
- TIF documents MUST include:
  - `tif_version`
  - `generated_at`
  - canonical hash
- Clients should cache TIF by `ETag` / `X-Canonical-Hash` and revalidate regularly.
- Facilities SHOULD support a controlled deprecation process for old versions.

---

## Recommended implementation patterns

### 1) Start read-only, then add approvals
Most production deployments should follow:
1. **RO** for observation and explanation
2. **Approve** for human-in-the-loop action gating
3. **Exec** only after action sets are narrow and mature

### 2) Treat “actions” like change management
Actions should map to:
- well-defined operational runbooks,
- bounded parameters,
- approval requirements,
- and rollback behavior where feasible.

### 3) Separate analytics from control
Time-series analytics services should:
- emit events and evidence
- open cases and advisories
- leave command gating to FTAI

### 4) Make topology and naming first-class
Invest early in:
- stable IDs,
- topology graphs,
- consistent zones and asset types,
- and a rigorous metrics/event catalog.

---

## Reference examples

### cURL examples

#### Discovery

```bash
curl -H "Authorization: Bearer $TOKEN"   https://facility.example.com/v1/
```

#### Get TIF (YAML)

```bash
curl -H "Accept: application/tif+yaml;version=1"   -H "Authorization: Bearer $TOKEN"   https://facility.example.com/v1/facility/tif
```

#### Get state snapshot

```bash
curl -H "Authorization: Bearer $TOKEN"   https://facility.example.com/v1/facility/state
```

#### Query events

```bash
curl -G https://facility.example.com/v1/facility/events   -H "Authorization: Bearer $TOKEN"   --data-urlencode "from=2026-01-07T00:00:00Z"   --data-urlencode "to=2026-01-07T01:00:00Z"   --data-urlencode "severity=HIGH"
```

#### Query metrics time series

```bash
curl -X POST https://facility.example.com/v1/facility/metrics/query   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -d '{
    "from": "2026-01-07T00:00:00Z",
    "to": "2026-01-07T01:00:00Z",
    "series": [
      {"key": "zone2.temp_swing_c", "aggregation": "avg", "interval_seconds": 60},
      {"key": "crah_12.vfd_speed_pct", "aggregation": "p95", "interval_seconds": 60}
    ],
    "max_points": 20000
  }'
```

#### Publish an advisory

```bash
curl -X POST https://facility.example.com/v1/advisories   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -d '{
    "type": "RECOMMENDED_ACTION",
    "severity": "HIGH",
    "title": "Cooling loop instability detected",
    "message": "Zone 2 temperature swing exceeded threshold; CRAH-12 VFD hunting observed.",
    "audience": {"roles": ["CoolingLead", "OpsManager"]},
    "evidence_links": ["trend://TQ-992188#zone2_temp_swing"],
    "requires_ack": true,
    "correlation_id": "CASE-1842"
  }'
```

#### Request an action approval

```bash
curl -X POST https://facility.example.com/v1/actions/request   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -H "Idempotency-Key: d87a8f6c-7e3c-4d2b-a9d0-0fe7cb6a6e2b"   -d '{
    "action_id": "ACT.CRAH.SETPOINT",
    "parameters": {"asset_id": "CRAH-12", "setpoint_c": 22.0},
    "justification": {
      "summary": "Stabilize Zone 2 swing and reduce VFD oscillation.",
      "evidence": [
        {"type":"event","code":"BMS.CRAH.VFD_HUNT","note":"14 events in 10m"},
        {"type":"metric","key":"zone2.temp_swing_c","range":"last_30m"}
      ]
    },
    "requested_mode": "request_approval",
    "correlation_id": "CASE-1842",
    "requested_by": {"agent_id": "gridiris-prod"}
  }'
```

---

### Implementation pseudo-code

#### Agent boot sequence

```text
discovery = GET /v1/
tif = GET /v1/facility/tif (cache by ETag/hash)
constraints = GET /v1/facility/constraints/effective
state = GET /v1/facility/state
subscribe SSE /v1/stream/events

on event:
  if event indicates anomaly:
    gather evidence with metrics query
    create/update case
    publish advisory
    if allowed -> request approval for bounded action
```

#### FTAI action enforcement outline

```text
on POST /actions/request:
  validate caller scopes
  validate action_id exists in TIF controls
  validate parameter schema
  enforce envelopes and rate limits
  evaluate live preconditions
  enforce lockouts/windows
  if approvals required:
     create approval artifact
     return PENDING_APPROVAL with policy evaluation details
  else if requested_mode=execute_if_allowed and policy permits:
     execute via adapter
     return EXECUTED
  else:
     return ACCEPTED
```

---

## FAQ

### Is FACET only for data centers?
No. FACET is facility-agnostic. Data centers are a motivating use case due to high consequence and instrumentation density.

### Do we have to allow the agent to execute actions?
No. Most sites should begin with RO or Approve profiles. Execution is optional and should be limited to narrow, vetted action sets.

### How does FACET prevent hallucinated actions?
The agent can only request actions defined by TIF (or the effective catalog). FTAI rejects unknown actions and out-of-envelope parameters.

### How are vendor protocols handled?
FACET does not replace Modbus/OPC UA/BACnet/SNMP/etc. FTAI is the normalized boundary; adapters handle vendor specifics.

### Does FACET require biometrics or identity tracking?
No. FACET intentionally avoids mandating high-risk privacy features. Facilities may implement optional modules with strong governance.

---

## Roadmap

Planned spec components and ecosystem artifacts:

- **TIF JSON Schema** (canonical validation)
- Deterministic **TIF plaintext rendering rules**
- Domain-specific **profiles** (data center, microgrid, factory)
- **Conformance test suite** (fixtures + validators)
- Reference gateway **skeleton** implementations
- Security hardening guide for OT boundary deployments
- Guidance for approval UX patterns (portal/kiosk/mobile)

---

## Contributing

FACET welcomes contributions in:
- specification clarity and examples
- schemas and validators
- conformance fixtures and test harnesses
- security considerations and best practices
- domain profiles and mapping guidance

Recommended process:
1. Open an issue describing the proposal and motivation.
2. Provide compatibility considerations and examples.
3. Submit a PR updating specs, schemas, and fixtures.
4. Include tests/fixtures for any normative changes.

---

## License

FACET is intended to be published with:
- Specification text: **CC-BY-4.0** 
- Reference code: **Apache-2.0** 


