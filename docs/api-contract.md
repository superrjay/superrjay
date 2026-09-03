# Core HR (Group 4) — API Contract (React ↔ Laravel)

Defines the concrete Laravel REST API surface that the Core HR React frontend
consumes. This is the **first-party API contract** (Group 4's own frontend talking to
Group 4's own backend); the **cross-group** contract (how Groups 1–3, 5–8 consume this
same backend) is documented separately in
[`integration-contract.md`](./integration-contract.md), since it has different
authentication and payload-shaping concerns.

**Related documents:**
[`requirements.md`](./requirements.md) ·
[`architecture.md`](./architecture.md) ·
[`ai-architecture.md`](./ai-architecture.md) ·
[`database-design.md`](./database-design.md) ·
[`integration-contract.md`](./integration-contract.md)

**Status:** Design only — no `routes/api.php`, controllers, or Form Requests have been
created as part of this analysis phase. Route names, request/response shapes, and
status codes below are the proposed contract for implementation.

---

## 1. Conventions

- **Base path:** `/api/core-hr/v1/...` — versioned in the URL so a future `v2` can be
  introduced additively (see `integration-contract.md` §6 for the versioning/
  deprecation policy, which applies equally to this first-party API).
- **Format:** JSON request/response bodies (`Content-Type: application/json`,
  `Accept: application/json`).
- **Response shaping:** Laravel **API Resources** (`JsonResource` /
  `ResourceCollection`) wrap every model response, so response shape is decoupled from
  raw Eloquent attribute names and can hide internal-only columns (e.g., never
  serialize `users.password`, and `EmployeeProfile`/`HrDocument` resources explicitly
  separate `source_data`/`ai_generated` sections per `ai-architecture.md` §3.3).
- **Validation:** every write endpoint is backed by a Laravel **Form Request**
  (`App\Http\Requests\*`), per `FR-EMP-16`. Validation failures return Laravel's
  standard `422 Unprocessable Entity` shape:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "hire_date": ["The hire date field is required."]
  }
}
```

- **Success envelope (single resource):**

```json
{
  "data": { "id": 42, "full_name": "Jane Dela Cruz", "...": "..." }
}
```

- **Success envelope (collection, paginated via Laravel's `paginate()`):**

```json
{
  "data": [ { "id": 42, "...": "..." } ],
  "links": { "first": "...", "last": "...", "prev": null, "next": "..." },
  "meta": { "current_page": 1, "per_page": 20, "total": 137 }
}
```

- **Error envelope (non-validation errors):**

```json
{
  "message": "You are not authorized to approve this document.",
  "error_code": "POLICY_DENIED"
}
```

- **Pagination:** all index/list endpoints support `?page=`, `?per_page=` (bounded,
  e.g., max 100) using Laravel's built-in paginator.
- **Filtering/sorting:** list endpoints accept simple query filters (e.g.,
  `?department_id=`, `?status=`, `?sort=-created_at`) — `ASSUMPTION`: exact filter set
  per endpoint to be finalized during implementation; a reasonable default is provided
  per resource below.
- **Rate limiting:** Laravel's `throttle` middleware on all routes (`ASSUMPTION`:
  default suggestion 60 requests/minute per authenticated user for standard CRUD;
  a stricter limit, e.g., 10/minute, on AI generation endpoints given their cost and
  latency — see §5).

---

## 2. Authentication

**Mechanism:** Laravel Sanctum (see `architecture.md` §3.2).

| Client | Auth mode | Header/mechanism |
|---|---|---|
| React SPA (first-party, same top-level domain) | Sanctum SPA cookie session | `GET /sanctum/csrf-cookie` then standard session cookies; React's HTTP client sends `X-XSRF-TOKEN` automatically (e.g., via `axios` + `withCredentials: true`) |
| React SPA (if deployed cross-origin) `ASSUMPTION` | Sanctum personal access token | `Authorization: Bearer <token>` header, token obtained via `POST /api/core-hr/v1/auth/login` |
| Other MMS groups' backends | Sanctum personal access token (service token) | `Authorization: Bearer <service-token>` — see `integration-contract.md` §2 |

### 2.1 Auth endpoints

| Method | Path | Purpose | Auth required |
|---|---|---|---|
| `POST` | `/api/core-hr/v1/auth/login` | Authenticate, returns user + (if token mode) a Sanctum token | No |
| `POST` | `/api/core-hr/v1/auth/logout` | Revoke current session/token | Yes |
| `GET` | `/api/core-hr/v1/auth/me` | Return the authenticated user + roles/permissions | Yes |

`ASSUMPTION`: If the client's real environment has a shared SSO/auth system across all
8 groups, these endpoints should be replaced/adapted accordingly (see
`architecture.md` §3.2).

---

## 3. Employee Management

| Method | Path | Purpose | Form Request | Policy |
|---|---|---|---|---|
| `GET` | `/employees` | List/search employees (filters: `department_id`, `branch_id`, `status`, `q` for name search) | — | `EmployeePolicy::viewAny` |
| `POST` | `/employees` | Create employee (master + personal + contact + emergency + initial employment info) | `StoreEmployeeRequest` | `EmployeePolicy::create` |
| `GET` | `/employees/{employee}` | View one employee (full profile incl. contact, emergency contacts, current employment info) | — | `EmployeePolicy::view` |
| `PUT`/`PATCH` | `/employees/{employee}` | Update employee personal/contact info | `UpdateEmployeeRequest` | `EmployeePolicy::update` |
| `DELETE` | `/employees/{employee}` | Soft-delete an employee record (rare; typically separation is used instead) | — | `EmployeePolicy::delete` |
| `GET` | `/employees/{employee}/history` | Employment history ledger for one employee | — | `EmployeePolicy::view` |
| `GET` | `/employees/me` | ESS: current authenticated user's own employee record | — | (self, via `linked_employee_id`) |
| `PATCH` | `/employees/me/contact-info` | ESS: submit a self-service contact-info change request (creates a `SelfServiceUpdateRequest`, does not directly modify the record) | `SubmitSelfServiceUpdateRequest` | (self only) |

---

## 4. Organization Management

| Method | Path | Purpose | Policy |
|---|---|---|---|
| `GET`/`POST` | `/departments` | List / create departments | `DepartmentPolicy::viewAny` / `create` |
| `GET`/`PUT`/`DELETE` | `/departments/{department}` | View / update / soft-delete a department | `DepartmentPolicy::*` |
| `GET`/`POST` | `/positions` | List / create positions (filter: `department_id`) | `PositionPolicy::*` |
| `GET`/`PUT`/`DELETE` | `/positions/{position}` | View / update / soft-delete a position | `PositionPolicy::*` |
| `GET`/`POST` | `/branches` | List / create branches | `BranchPolicy::*` |
| `GET`/`PUT`/`DELETE` | `/branches/{branch}` | View / update / soft-delete a branch | `BranchPolicy::*` |
| `GET` | `/org-structure` | Composed read-only view of the full department → position hierarchy + branch groupings, for org-chart rendering | `OrganizationPolicy::view` |

---

## 5. Employment Lifecycle

| Method | Path | Purpose | Form Request | Policy |
|---|---|---|---|---|
| `POST` | `/employees/{employee}/transfers` | Initiate a transfer (`status=pending`) | `StoreTransferRequest` | `EmployeeTransferPolicy::create` |
| `GET` | `/employees/{employee}/transfers` | List transfer history for an employee | — | `EmployeeTransferPolicy::viewAny` |
| `PATCH` | `/transfers/{transfer}/approve` | Approve a pending transfer (applies change + appends `EmploymentHistory`) | — | `EmployeeTransferPolicy::approve` |
| `PATCH` | `/transfers/{transfer}/reject` | Reject a pending transfer | `RejectTransferRequest` (reason) | `EmployeeTransferPolicy::approve` |
| `POST` | `/employees/{employee}/promotions` | Initiate a promotion (`status=pending`) | `StorePromotionRequest` | `EmployeePromotionPolicy::create` |
| `PATCH` | `/promotions/{promotion}/approve` | Approve a pending promotion | — | `EmployeePromotionPolicy::approve` |
| `PATCH` | `/promotions/{promotion}/reject` | Reject a pending promotion | `RejectPromotionRequest` | `EmployeePromotionPolicy::approve` |
| `POST` | `/employees/{employee}/separations` | Initiate a resignation/termination record | `StoreSeparationRequest` | `SeparationRecordPolicy::create` |
| `PATCH` | `/separations/{separation}/approve` | Approve separation (updates `employment_status`) | — | `SeparationRecordPolicy::approve` |
| `GET` | `/self-service-requests` | List pending self-service update requests (HR queue view) | — | `SelfServiceUpdateRequestPolicy::viewAny` |
| `PATCH` | `/self-service-requests/{request}/approve` | Approve an employee's self-service info change | — | `SelfServiceUpdateRequestPolicy::approve` |
| `PATCH` | `/self-service-requests/{request}/reject` | Reject an employee's self-service info change | `RejectSelfServiceRequest` | `SelfServiceUpdateRequestPolicy::approve` |

---

## 6. Employee Documents (non-AI)

| Method | Path | Purpose | Form Request | Policy |
|---|---|---|---|---|
| `GET` | `/employees/{employee}/documents` | List an employee's uploaded documents | — | `EmployeeDocumentPolicy::viewAny` |
| `POST` | `/employees/{employee}/documents` | Upload a document (multipart/form-data) | `StoreEmployeeDocumentRequest` (file type/size validation) | `EmployeeDocumentPolicy::create` |
| `GET` | `/employee-documents/{document}/download` | Download/stream a document file | — | `EmployeeDocumentPolicy::view` |
| `DELETE` | `/employee-documents/{document}` | Remove a document (soft-delete) | — | `EmployeeDocumentPolicy::delete` |

---

## 7. AI Feature 1 — Employee Profiling

| Method | Path | Purpose | Form Request | Policy | Rate limit |
|---|---|---|---|---|---|
| `POST` | `/employees/{employee}/profile/generate` | Trigger Gemini-assisted profile generation (see `ai-architecture.md` §3) | `GenerateProfileRequest` | `EmployeeProfilePolicy::generate` (maps to `ai.profile.generate`) | Stricter (e.g., 10/min) |
| `GET` | `/employees/{employee}/profiles` | List all profile versions for an employee (draft + approved history) | — | `EmployeeProfilePolicy::viewAny` | Standard |
| `GET` | `/employee-profiles/{profile}` | View one profile (source data vs. AI narrative, separated per `ai-architecture.md` §3.3) | — | `EmployeeProfilePolicy::view` | Standard |
| `PATCH` | `/employee-profiles/{profile}` | Edit the AI-generated narrative text before approval | `UpdateProfileDraftRequest` | `EmployeeProfilePolicy::update` | Standard |
| `POST` | `/employee-profiles/{profile}/approve` | Approve a draft profile as the official version | — | `EmployeeProfilePolicy::approve` | Standard |
| `POST` | `/employee-profiles/{profile}/reject` | Reject/discard a draft profile | `RejectProfileRequest` | `EmployeeProfilePolicy::approve` | Standard |

If asynchronous generation is adopted (`ai-architecture.md` §2.6):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/employee-profiles/generation-jobs/{job}` | Poll generation job status (`pending`/`completed`/`failed`) |

---

## 8. AI Feature 2 — Document Drafting

| Method | Path | Purpose | Form Request | Policy | Rate limit |
|---|---|---|---|---|---|
| `GET` | `/document-types` | List supported document types (Certificate of Employment, Appointment Letter, etc. — `FR-DOC-11`) | — | Authenticated | Standard |
| `GET` | `/document-templates` | List approved/published templates (filter: `document_type`) | — | `DocumentTemplatePolicy::viewAny` | Standard |
| `POST` | `/document-templates` | Create a new template version | `StoreDocumentTemplateRequest` | `DocumentTemplatePolicy::create` (HR Admin only) | Standard |
| `PATCH` | `/document-templates/{template}/publish` | Publish a template version as `approved` | — | `DocumentTemplatePolicy::publish` | Standard |
| `POST` | `/hr-documents/generate` | Trigger Gemini-assisted document draft generation (`{employee_id, document_type, template_id}`, see `ai-architecture.md` §4) | `GenerateDocumentRequest` | `HrDocumentPolicy::generate` (maps to `ai.document.generate`) | Stricter (e.g., 10/min) |
| `GET` | `/hr-documents` | List documents (filters: `employee_id`, `status`, `document_type`) — reviewer queue view via `?status=for_review` | — | `HrDocumentPolicy::viewAny` | Standard |
| `GET` | `/hr-documents/{document}` | View one document (current content + status) | — | `HrDocumentPolicy::view` | Standard |
| `PATCH` | `/hr-documents/{document}` | Edit draft content (only while `status=draft`) | `UpdateHrDocumentRequest` | `HrDocumentPolicy::update` | Standard |
| `PATCH` | `/hr-documents/{document}/status` | Transition status: `{to: "for_review"\|"approved"\|"finalized"\|"archived"\|"draft"}` (see `architecture.md` §9 for the state machine) | `TransitionHrDocumentStatusRequest` | `HrDocumentPolicy::{submitForReview,approve,finalize,archive,revokeApproval}` (method chosen based on requested transition) | Standard |
| `GET` | `/hr-documents/{document}/history` | Full draft/edit/transition history | — | `HrDocumentPolicy::view` | Standard |
| `GET` | `/hr-documents/{document}/export` | Export the `finalized` document (e.g., as PDF) for manual printing/sending — **never automated** | — | `HrDocumentPolicy::view` + status must be `finalized` | Standard |

If asynchronous generation is adopted:

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/hr-documents/generation-jobs/{job}` | Poll generation job status |

---

## 9. Audit Trail

| Method | Path | Purpose | Policy |
|---|---|---|---|
| `GET` | `/audit-logs` | List/search `hr_audit_logs` (filters: `employee_id`, `action_type`, `ai_feature_used`, `date_from`, `date_to`) | `HrAuditLogPolicy::viewAny` (HR Admin/Manager, scoped) |
| `GET` | `/audit-logs/{log}` | View one audit entry | `HrAuditLogPolicy::view` |
| `GET` | `/ai-request-logs` | List/search `ai_request_logs` metadata (for AI usage/cost monitoring) | `AiRequestLogPolicy::viewAny` (HR Admin/System Admin) |

---

## 10. Roles & Permissions Administration

| Method | Path | Purpose | Policy |
|---|---|---|---|
| `GET` | `/roles` | List roles | `RolePolicy::viewAny` |
| `GET` | `/permissions` | List permissions | `RolePolicy::viewAny` |
| `POST` | `/users/{user}/roles` | Assign a role to a user | `RolePolicy::assign` (HR Admin/System Admin) |
| `DELETE` | `/users/{user}/roles/{role}` | Remove a role from a user | `RolePolicy::assign` |

---

## 11. Endpoint-Level Rate Limiting Summary

| Endpoint group | Suggested limit | Rationale |
|---|---|---|
| Auth (`/auth/*`) | Strict (e.g., 5/min for `login`) | Brute-force mitigation |
| Standard CRUD (employees, org structure, lifecycle, documents) | Moderate (e.g., 60/min per user) | Normal interactive usage |
| AI generation (`/*/generate`) | Strict (e.g., 10/min per user) | Cost control + Gemini rate-limit protection (`ai-architecture.md` §2.5) |
| Read-only reporting/audit endpoints | Moderate | |

`ASSUMPTION`: Exact numeric limits above are starting-point recommendations, to be
tuned once real usage patterns and Gemini quota/cost constraints are known.

---

## 12. Sample Request/Response — AI Document Draft Generation

Illustrative only (no implementation), showing how the pieces in
`ai-architecture.md` §4 surface through this contract:

**Request**

```
POST /api/core-hr/v1/hr-documents/generate
Authorization: Bearer <sanctum-token-or-session-cookie>
Content-Type: application/json

{
  "employee_id": 42,
  "document_type": "certificate_of_employment",
  "template_id": 7
}
```

**Response — `201 Created`**

```json
{
  "data": {
    "id": 501,
    "employee_id": 42,
    "document_type": "certificate_of_employment",
    "template_id": 7,
    "status": "draft",
    "content": "This is to certify that Jane Dela Cruz has been employed as...",
    "needs_review_sections": [],
    "ai_generated": true,
    "ai_request_log_id": 1032,
    "created_by_user_id": 8,
    "created_at": "2026-09-02T23:59:00Z"
  }
}
```

**Response — `422` (validation, e.g., no approved template for this document type)**

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "template_id": ["The selected template is not approved for this document type."]
  }
}
```

**Response — `503` (Gemini unavailable after retries)**

```json
{
  "message": "The AI drafting service is temporarily unavailable. Please try again shortly, or create the document manually.",
  "error_code": "AI_PROVIDER_UNAVAILABLE"
}
```

---

*End of `api-contract.md`. See [`integration-contract.md`](./integration-contract.md)
for the separate cross-group (service-to-service) API surface, and
[`database-design.md`](./database-design.md) for the underlying Eloquent models these
endpoints operate on.*
