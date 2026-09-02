# Core HR (Group 4) — System Architecture

**Related documents:**
[`requirements.md`](./requirements.md) ·
[`ai-architecture.md`](./ai-architecture.md) ·
[`database-design.md`](./database-design.md) ·
[`integration-contract.md`](./integration-contract.md)

**Status:** Draft. Deployment topology, hosting, and infra choices are marked
`ASSUMPTION` where not yet confirmed by the client/infra team.

---

## 1. Architectural Principles

1. **Core HR is the single source of truth** for employee master data and
   organizational structure across the entire Microfinancial Management System (MMS).
   No other group persists its own copy of these facts; they either call Core HR's
   APIs at read time or consume published domain events.
2. **AI is assistive, never authoritative.** The AI Service can only produce `DRAFT`
   artifacts. It has no write access to authoritative tables and no capability to
   invoke approval, finalize, or transmit actions.
3. **Clear module boundaries.** Core HR is decomposed into cohesive modules (see
   `requirements.md` §9) that map to bounded contexts, each with its own data
   ownership within the Core HR database.
4. **Defense in depth for sensitive data.** RBAC, data minimization, and audit logging
   are layered, not relied upon individually.
5. **Loose coupling with other groups.** Integration happens through versioned APIs
   and/or asynchronous domain events, never direct cross-group database access.
6. **Provider abstraction.** The Gemini integration sits behind an internal `AIProvider`
   interface so the concrete LLM vendor/SDK can change without impacting business logic.

---

## 2. High-Level System Context

```mermaid
flowchart TB
    subgraph Clients
        WebUI["Core HR Web App\n(HR Admin/Manager/Staff UI)"]
        ESSUI[Employee Self-Service\nWeb Portal]
    end

    subgraph Group4["Group 4 — Core HR Subsystem"]
        API["Core HR API Gateway\n(REST/JSON, versioned)"]
        CoreSvc["Core HR Domain Services\n(Employee, Org, Lifecycle, Documents)"]
        AISvc["AI Service Module\n(Gemini integration)"]
        DB[(Core HR Database)]
        Audit[(Audit Log Store)]
    end

    Gemini[(Google Gemini API)]

    subgraph OtherGroups["Other MMS Subsystems (Groups 1,2,3,5,6,7,8)"]
        G2[Group 2\nRecruitment & Onboarding]
        G6[Group 6\nPayroll & Benefits]
        G7[Group 7\nWorkforce Management]
        G8[Group 8\nPerformance & Development]
        G5[Group 5\nFleet & Transportation]
        G13[Groups 1 & 3\nInventory / Financial]
    end

    WebUI --> API
    ESSUI --> API
    API --> CoreSvc
    API --> AISvc
    CoreSvc --> DB
    CoreSvc --> Audit
    AISvc --> Audit
    AISvc -->|server-side only,\nAPI key never exposed| Gemini
    AISvc -.reads via CoreSvc.-> DB

    G2 -->|reads employee/org data| API
    G6 -->|reads employee/org data| API
    G7 -->|reads employee/org data| API
    G8 -->|reads employee/org data| API
    G5 -->|reads employee/org data| API
    G13 -->|reads employee/org data| API

    CoreSvc -.publishes domain events\n(employee hired/updated/\ntransferred/terminated).-> EventBus[(Event Bus / Webhook\nASSUMPTION: technology TBD)]
    EventBus -.-> G2
    EventBus -.-> G6
    EventBus -.-> G7
    EventBus -.-> G8
```

`ASSUMPTION`: The exact event-bus technology (message broker vs. webhook callbacks vs.
polling) is not yet specified by the client/infra team. Both a synchronous API and an
asynchronous event option are described in `integration-contract.md` so the final
choice can be made without redesigning the domain model.

---

## 3. Layered Architecture (within Group 4)

```mermaid
flowchart TB
    subgraph Presentation["Presentation Layer"]
        A1[HR Web App]
        A2[ESS Web Portal]
    end

    subgraph API_Layer["API / Gateway Layer"]
        B1[Core HR REST API]
        B2[AuthN/AuthZ Middleware\nRBAC enforcement]
        B3[Rate limiting & request logging]
    end

    subgraph Application["Application / Service Layer"]
        C1[Employee Service]
        C2[Organization Service]
        C3["Employment Lifecycle Service\n(Transfers/Promotions/Offboarding)"]
        C4[Document & Template Service]
        C5[Employee Profile Service]
        C6[Self-Service Request Service]
        C7[Audit Service]
        C8[AI Orchestration Service]
    end

    subgraph AIModule["AI Service Module (see ai-architecture.md)"]
        D1[Context Builder /\nData Minimization Filter]
        D2[Prompt Template Engine]
        D3[Gemini Client Adapter]
        D4[Input/Output Validator]
        D5[AI Request Logger]
    end

    subgraph Persistence["Persistence Layer"]
        E1[(Core HR Relational DB)]
        E2[(Document/File Storage)]
        E3[(Audit Log Store)]
    end

    subgraph External["External"]
        F1[(Google Gemini API)]
        F2[Other MMS Subsystems]
    end

    A1 --> B1
    A2 --> B1
    B1 --> B2 --> B3 --> Application
    C1 --> E1
    C2 --> E1
    C3 --> E1
    C3 --> C7
    C4 --> E1
    C4 --> E2
    C5 --> C8
    C6 --> C7
    C8 --> D1 --> D2 --> D3 --> F1
    D3 --> D4 --> C8
    C8 --> D5 --> E3
    C7 --> E3
    C1 -->|read-only exposure| F2
    C2 -->|read-only exposure| F2
```

---

## 4. Module Responsibilities

| Layer | Module | Responsibility | Talks to |
|---|---|---|---|
| Application | **Employee Service** | CRUD for employee master/personal/contact/emergency data; enforces field-level RBAC. | Core HR DB, Audit Service |
| Application | **Organization Service** | Departments, positions, branches, reporting hierarchy. | Core HR DB, Audit Service |
| Application | **Employment Lifecycle Service** | Transfers, promotions, resignation/termination workflows; append-only employment history ledger; status-change approvals. | Core HR DB, Audit Service, publishes domain events |
| Application | **Document & Template Service** | Employee document uploads (metadata + storage pointer); HR document templates; generated document lifecycle state machine. | Core HR DB, Document/File Storage, AI Orchestration Service, Audit Service |
| Application | **Employee Profile Service** | Orchestrates AI profile generation requests, review, versioning of approved profiles. | AI Orchestration Service, Core HR DB, Audit Service |
| Application | **Self-Service Request Service** | Employee-submitted change requests and their HR approval workflow. | Core HR DB, Audit Service |
| Application | **Audit Service** | Central append-only writer/reader for the HR audit trail. | Audit Log Store |
| Application | **AI Orchestration Service** | Coordinates a generation request end-to-end: fetch approved data → hand off to AI Service Module → receive validated draft → persist as `DRAFT`. Never itself calls approval/finalize logic. | AI Service Module, Document & Template Service, Employee Profile Service |
| AI Module | **Context Builder / Data Minimization Filter** | Applies per-task field allow-lists; strips/masks excluded fields (see `ai-architecture.md` §6). | Called by AI Orchestration Service |
| AI Module | **Prompt Template Engine** | Renders versioned prompt templates with minimized context. | Context Builder |
| AI Module | **Gemini Client Adapter** | Sole component with Gemini API credentials; performs the actual API call, applies retries/timeouts. | Google Gemini API |
| AI Module | **Input/Output Validator** | Pre-call input validation; post-call schema + fact-grounding validation. | Gemini Client Adapter |
| AI Module | **AI Request Logger** | Writes privacy-aware metadata records of each AI call to the audit store (not raw sensitive content). | Audit Log Store |

---

## 5. API Boundaries

Core HR exposes a single **versioned REST API** (`/api/core-hr/v1/...`) as the only
sanctioned entry point into its data and functions. Internal service-to-service calls
within Group 4 may use direct in-process calls or an internal RPC, but that is an
implementation detail invisible to other groups.

### 5.1 API surface groups

| Surface | Audience | Examples |
|---|---|---|
| **HR Management API** | Group 4's own HR Web App | Full CRUD on employees, org structure, lifecycle events, templates, documents, profiles. Protected by RBAC per `requirements.md` §6.1. |
| **Employee Self-Service API** | Group 4's own ESS Portal | Read own record; submit self-service change requests. Enforces "self only" row-level security. |
| **AI Generation API** | Group 4's own HR Web App (never called directly by frontend to Gemini) | `POST /ai/employee-profile/generate`, `POST /ai/documents/generate`, review/approve endpoints. |
| **Cross-Group Integration API** | Other MMS subsystems (Groups 1,2,3,5,6,7,8) | Read-mostly employee/org endpoints + optional domain events. Detailed contract in [`integration-contract.md`](./integration-contract.md). |
| **Admin/Audit API** | HR Admin, System Admin | Role/permission management, audit trail query. |

### 5.2 API boundary rules

- All authoritative **writes** to employee/org/employment data happen only inside
  Group 4's own services, triggered only by Group 4's own UI (HR Web App / ESS Portal)
  or well-defined internal workflows (e.g., approval transitions). No external group is
  ever granted write access to Core HR data.
- Other groups integrate **read-only** against the Cross-Group Integration API (or
  subscribe to domain events for change notifications) — see
  [`integration-contract.md`](./integration-contract.md) for exact endpoints/events.
  If another group needs to *request* a Core HR change (e.g., Payroll flags a status
  discrepancy), that is modeled as a request/notification back to Core HR, not a direct
  write.
- The **AI Generation API** is only reachable by authenticated Group 4 HR users through
  the Group 4 backend; the Gemini API itself is never reachable from any frontend, and
  never reachable by other groups at all.
- All API responses use consistent envelope, pagination, and error formats
  (`ASSUMPTION`: exact envelope/error schema to be aligned with the MMS-wide API
  standard, if one exists at the program level — not yet supplied).

---

## 6. Deployment View (indicative)

```mermaid
flowchart TB
    subgraph Client_Tier["Client Tier"]
        Browser[Employee/HR Browser Sessions]
    end

    subgraph App_Tier["Application Tier (Group 4 backend)"]
        LB[Load Balancer / API Gateway]
        AppSvc1[Core HR App Instance]
        AppSvc2[Core HR App Instance]
        AISvcNode["AI Service Module\n(same deployable or\nseparate microservice)"]
    end

    subgraph Data_Tier["Data Tier"]
        RDBMS[(Relational DB\ne.g., PostgreSQL)]
        ObjectStore[(Object/File Storage\nfor documents & attachments)]
        AuditStore[(Audit Log Store\ne.g., append-only table\nor dedicated log store)]
        SecretStore[(Secret Manager\nGemini API Key)]
    end

    subgraph External_Tier["External"]
        GeminiAPI[(Google Gemini API)]
        OtherGroupAPIs[Other Group Services]
    end

    Browser --> LB --> AppSvc1
    LB --> AppSvc2
    AppSvc1 --> RDBMS
    AppSvc2 --> RDBMS
    AppSvc1 --> ObjectStore
    AppSvc1 --> AuditStore
    AppSvc1 --> AISvcNode
    AISvcNode --> SecretStore
    AISvcNode -->|HTTPS, key from\nSecretStore only| GeminiAPI
    AppSvc1 <-->|versioned REST\n+ optional events| OtherGroupAPIs
```

`ASSUMPTION`: Whether the AI Service Module is deployed as a separable microservice or
as an in-process module of the same Core HR backend is an infrastructure decision, not
a domain-modeling one; the logical boundary (module) is fixed regardless, so this can
change later without affecting the design in this document set.

---

## 7. Cross-Cutting Concerns

### 7.1 Security
- RBAC enforced at the API Gateway/middleware layer on every request (see
  `requirements.md` §4.5, §6.1).
- Row-level scoping for ESS users (self-record only) and, if confirmed, branch/
  department scoping for HR Manager (see `requirements.md` Assumption A2).
- Gemini API key and any other AI-provider credentials live only in a server-side
  secret store; the AI Service Module is the only component with runtime access to it.
- All inter-service and external calls use TLS.

### 7.2 Auditability
A single **Audit Service** is the only writer to the Audit Log Store. Every mutating
domain action (CRUD on employee/org/lifecycle data, template changes, document status
transitions) and every AI action (generation request, validation outcome, human review
decision) is emitted as a structured audit event. See §21-equivalent detail in
[`ai-architecture.md`](./ai-architecture.md) §7 for AI-specific redaction rules and
[`database-design.md`](./database-design.md) §5 for the audit log schema.

### 7.3 Extensibility
- New document types are added by publishing a new `DocumentTemplate` (data), not by
  code changes, as long as they fit the existing template/field-merge model.
- New AI tasks (e.g., a future "exit interview summary") plug into the same AI Service
  Module by adding a new prompt template + allow-list entry, reusing the same
  validation/audit/human-review pipeline.
- New consumer groups integrate against the existing Cross-Group Integration API
  surface; contract versioning (see `integration-contract.md` §1) allows additive
  changes without breaking existing consumers.

### 7.4 Failure isolation
- Core HR's non-AI functionality (CRUD, lifecycle workflows, ESS) has **no runtime
  dependency** on the Gemini API being available. AI features degrade independently.
- The AI Service Module applies timeouts, retries with backoff, and (recommended) a
  circuit breaker so repeated Gemini failures don't degrade the rest of the platform
  (see `ai-architecture.md` §2.5).

---

## 8. Document Lifecycle (Architectural View)

The document lifecycle state machine is owned by the **Document & Template Service**
and is identical regardless of whether the document was AI-drafted or manually created
(manually-created documents simply skip the "AI generation" trigger and start directly
in `DRAFT`, authored by a human).

```mermaid
stateDiagram-v2
    [*] --> DRAFT: AI generation OR manual creation
    DRAFT --> DRAFT: Edit content
    DRAFT --> FOR_REVIEW: Submit for review\n(HR Staff/Manager/Admin)
    FOR_REVIEW --> DRAFT: Send back for edits
    FOR_REVIEW --> APPROVED: Approve\n(HR Manager/Admin only)
    APPROVED --> FINALIZED: Finalize\n(HR Manager/Admin only)
    FINALIZED --> ARCHIVED: Archive\n(HR Manager/Admin)
    APPROVED --> DRAFT: Revoke approval\n(exceptional, audited)
    ARCHIVED --> [*]
```

Full field-level detail (who can trigger each transition, what gets logged, what data
is required) is in [`ai-architecture.md`](./ai-architecture.md) §4.3 and
[`database-design.md`](./database-design.md) entity `HRDocument`.

---

## 9. Integration Points Summary

Detailed request/response contracts, event payloads, and versioning policy are in
[`integration-contract.md`](./integration-contract.md). Summary of *why* each group
integrates with Core HR:

| Group | Needs from Core HR | Direction |
|---|---|---|
| 1 — Supply Chain & Inventory | Employee identity for accountability on inventory transactions (e.g., "issued by", "received by") `(ASSUMPTION: exact use case unconfirmed)` | Core HR → Group 1 (read) |
| 2 — Recruitment & Onboarding | Create/link a Core HR employee master record once a candidate is hired; org structure (departments/positions) to place new hires into | Group 2 → Core HR (create-on-hire callback) + Core HR → Group 2 (read org structure) |
| 3 — Financial Management | Employee identity for expense/disbursement approval chains `(ASSUMPTION)` | Core HR → Group 3 (read) |
| 5 — Fleet & Transportation | Employee/driver identity, department/branch for vehicle assignment and accountability | Core HR → Group 5 (read) |
| 6 — Payroll & Benefits | Employee master data, employment status, position/department, employment history (hire/termination dates) as the factual basis for pay computation and benefits eligibility | Core HR → Group 6 (read + change events) |
| 7 — Workforce Management | Employee master data, department/branch, employment status for scheduling/attendance/leave eligibility | Core HR → Group 7 (read + change events) |
| 8 — Performance & Development | Employee master data, position, department, employment history for performance review and succession planning context | Core HR → Group 8 (read + change events) |

---

*End of `architecture.md`. See [`ai-architecture.md`](./ai-architecture.md) for AI
subsystem internals and [`integration-contract.md`](./integration-contract.md) for
concrete API/event contracts.*
