# Core HR (Group 4) — Existing Project Analysis

**Purpose:** Before any implementation work begins, this document records the results
of inspecting the current repository/workspace for an existing Laravel + React
codebase that Group 4 (Core HR) would need to extend, plus the tooling available in
the development environment. Per the task instructions, **no application code has
been written or modified** as part of this analysis — this is a read-only inspection.

**Related documents:**
[`requirements.md`](./requirements.md) ·
[`architecture.md`](./architecture.md) ·
[`ai-architecture.md`](./ai-architecture.md) ·
[`database-design.md`](./database-design.md) ·
[`api-contract.md`](./api-contract.md) ·
[`integration-contract.md`](./integration-contract.md)

---

## 1. Inspection Method

The following were checked directly in the repository/workspace before writing any
documentation:

1. `git status`, `git log --oneline` (all branches), `git branch -a` — full commit
   history and branch inventory.
2. Root and recursive directory listing (`find`, `ls -la`) across the entire
   repository, excluding `.git/`.
3. Search for Laravel signature files: `artisan`, `composer.json`, `composer.lock`,
   `config/app.php`, `routes/*.php`, `app/Http/Controllers/**`, `database/migrations/**`,
   `.env` / `.env.example`.
4. Search for React/frontend signature files: `package.json`, `vite.config.*`,
   `webpack.config.*`, `src/`, `resources/js/`, `tsconfig.json`.
5. Search for database configuration: `database/`, `phpMyAdmin` exports, `.sql` dump
   files, Docker/XAMPP configuration.
6. Availability of PHP, Composer, Node.js, npm, and MySQL/MariaDB tooling in the
   current environment.
7. Any Cursor/CI environment configuration (`.cursor/`, `environment.json`) that might
   declare an intended stack for this repository.

## 2. Findings

### 2.1 Repository contents

| Item checked | Result |
|---|---|
| Branches | `main`, `loruki`, and this task's working branch (`cursor/core-hr-docs-e5f1`). `main` and `loruki` point to the same 7-commit history. |
| Commit history | 7 commits total on `main`, all titled "Update README.md" / "Initial commit" / "Update last update date in README" — no application code has ever been committed to this repository. |
| Root directory contents (pre-existing, before this docs effort) | `README.md`, `.gitattributes` only. |
| `README.md` content | A personal GitHub-profile "under maintenance" README (the repository name matches the account name, which is GitHub's convention for a profile README repo), unrelated to a Microfinancial Management System or Core HR. |
| Laravel signature files (`artisan`, `composer.json`, `routes/`, `app/`, `database/migrations/`, etc.) | **None found.** |
| React/frontend signature files (`package.json`, `vite.config.*`, `src/`, `resources/js/`, `tsconfig.json`, etc.) | **None found.** |
| Database config / SQL dumps / phpMyAdmin exports | **None found.** |
| `.env` / `.env.example` | **None found.** |
| `.cursor/environment.json` or other environment/CI declarations in this repo | **None found.** |

### 2.2 Tooling available in the current environment

| Tool | Status |
|---|---|
| PHP | Not installed (`php -v` → command not found) |
| Composer | Not installed |
| MySQL / MariaDB server or client | Not installed |
| Node.js | v22.14.0 (installed) |
| npm | 10.9.7 (installed) |

### 2.3 Conclusion

**There is no existing Laravel or React codebase in this repository.** The repository
currently contains only documentation (the `/docs` set authored in the prior phase of
this engagement — `requirements.md`, `architecture.md`, `ai-architecture.md`,
`database-design.md`, `integration-contract.md`) and an unrelated personal GitHub
profile README. The development environment itself also has no PHP/Composer/MySQL
toolchain installed, consistent with no PHP application having been set up yet.

This changes the nature of the ten inspection items requested in the task brief: they
cannot be answered by reverse-engineering an existing codebase, because none exists.
Section 3 below answers each item as "not applicable / not found" and, where useful,
proposes a concrete recommended default so that downstream design documents
(`architecture.md`, `ai-architecture.md`, `database-design.md`, `api-contract.md`) have
a stable technical baseline to describe against. **Every proposed default is explicitly
marked `ASSUMPTION`** and must be confirmed — or replaced with the specifics of
whatever base project the client actually provides — before implementation begins.

`ASSUMPTION (A0 — Major)`: It is possible that the "existing project" this task
expects to find lives in a different repository (e.g., a shared monorepo already
scaffolded by another team, or a separate repository per group) that simply has not
been connected to this workspace yet, or that another Group's Laravel application is
meant to be the base into which Core HR is added as a module. If such a repository
exists, it should be provided/linked so this analysis can be redone against the real
codebase, its actual versions, structure, routes, auth setup, and conventions — rather
than against the recommended defaults below. Nothing in this document set assumes or
requires a specific resolution of this question; the architecture is written to work
whether Core HR ends up as **(a)** its own standalone Laravel + React application,
**(b)** a module inside a shared Laravel monolith with other groups' modules, or
**(c)** a set of Laravel packages consumed by a larger app — see
`architecture.md` §1 and `integration-contract.md` §1 for how the design stays neutral
to this choice.

---

## 3. Inspection Checklist (Requested Items 1–10)

### 1. Laravel version
**Not found — no Laravel installation exists in this repository.**
`ASSUMPTION`: Recommend **Laravel 11.x** (current stable LTS-track release as of this
analysis) as the baseline for a green-field build, using Laravel's default
`bigIncrements` primary keys and the streamlined Laravel 11 application skeleton
(slim `app/Http/Kernel.php`-free structure, `bootstrap/app.php`-based middleware/route
registration). If the client supplies an existing Laravel app on an older version
(e.g., 9.x or 10.x), this analysis must be redone against that version and the
architecture docs adjusted for any version-specific API differences (e.g., middleware
registration, casts syntax).

### 2. PHP version
**Not found — PHP is not installed in this environment.**
`ASSUMPTION`: Recommend **PHP 8.2 or 8.3**, matching Laravel 11's minimum PHP 8.2
requirement and enabling modern language features used in the design (readonly
properties, enums for `EmploymentStatus`/`DocumentStatus`, first-class callable syntax).

### 3. React setup
**Not found — no `package.json`, no React dependency, no bundler config.**
`ASSUMPTION`: Recommend a **React 18+ SPA built with Vite** (Laravel's default
frontend tooling via `laravel/vite-plugin` and `resources/js/`), using **TypeScript**
for type safety around API request/response shapes (especially important for the
AI-generated content structures where source-data vs. AI-narrative separation must be
strictly typed). If the client's existing frontend uses plain JavaScript, Create React
App, or a separate standalone SPA (not served from `resources/js/`), this should be
confirmed and the frontend integration approach (Sanctum SPA cookie auth vs. token
auth) adjusted accordingly — see `architecture.md` §3 (Authentication) and
`api-contract.md` §2.

### 4. Existing frontend structure
**Not applicable — no frontend code exists.** A proposed structure is given in
`architecture.md` §4 (Frontend Module Layout) as a starting point, organized by
feature (employees, org-structure, documents, ai-profiling, ai-drafting, audit) rather
than by technical layer, to keep Core HR's frontend cohesive and easy to hand off.

### 5. Existing backend structure
**Not applicable — no backend code exists.** A proposed Laravel application structure
(controllers, Form Requests, Services, Actions, Policies, Eloquent models, migrations)
is given in `architecture.md` §3–§5.

### 6. Existing routes
**Not applicable — no `routes/api.php` or `routes/web.php` exists.** The full proposed
route table is defined in `api-contract.md`.

### 7. Existing authentication
**Not found — no auth scaffolding, no Sanctum/Passport/Breeze/Jetstream installation.**
`ASSUMPTION`: Recommend **Laravel Sanctum** for authentication:
- SPA (cookie-based) session authentication if the React app is served from the same
  top-level domain as the Laravel backend (Sanctum's intended "first-party SPA" mode).
- Sanctum **personal access tokens** for (a) any mobile/other-client scenario, and (b)
  service-to-service calls from other Groups' backends into Core HR's Cross-Group
  Integration API (see `integration-contract.md` §2).
Laravel Passport (OAuth2) is not recommended here — the project does not need
third-party OAuth clients, and Sanctum is Laravel's documented default for
first-party SPA + simple token auth as of Laravel 11. This should be confirmed against
whatever auth mechanism (if any) the other 7 groups' backends already standardize on,
since Core HR's Cross-Group Integration API must be reachable by them.

### 8. Existing database configuration
**Not found — no `.env`, no `config/database.php`, no migrations.**
`ASSUMPTION`: Recommend **MySQL 8.0** as the database engine (matching the task's
specified **XAMPP** local development environment, which bundles MySQL/MariaDB),
accessed exclusively through **Laravel migrations, Eloquent models, factories, and
seeders** per the task's explicit instruction — no schema is to be created by hand
through phpMyAdmin. Full schema design is in `database-design.md`.

### 9. Existing dependencies
**Not applicable — no `composer.json` or `package.json` exists**, so there are no
existing Composer packages or npm packages to enumerate or remain compatible with.
`architecture.md` §6 lists the minimal, specifically-justified set of packages this
design anticipates needing (e.g., `laravel/sanctum`, an HTTP client for Gemini —
Laravel's built-in `Illuminate\Support\Facades\Http`, no extra package required). Per
the task instruction, **no packages have been installed**; this is a proposal only,
to be reconciled with the client's actual dependency constraints.

### 10. Existing coding conventions
**Not applicable — no code exists to derive conventions from.**
`ASSUMPTION`: In the absence of an existing style guide, this design assumes standard
Laravel conventions throughout (PSR-12 formatting via Laravel Pint, singular
model names / plural table names, `snake_case` database columns, `camelCase` PHP
variables/methods, resourceful controllers, Form Requests for validation, API
Resources for response shaping, and feature/unit tests under `tests/Feature` and
`tests/Unit`). If the client provides an existing project with different conventions
(e.g., a different code style, repository-pattern data access, or a modular monolith
package structure such as `nwidart/laravel-modules`), Core HR's implementation should
conform to those conventions rather than the defaults assumed here.

---

## 4. Impact on the Rest of This Document Set

Because there is no existing project to constrain or extend, the remaining documents
in this set (`requirements.md`, `architecture.md`, `ai-architecture.md`,
`database-design.md`, `api-contract.md`, `integration-contract.md`) describe a
**green-field Laravel + React design for Group 4**, built strictly to the tech-stack
and architectural constraints given in the task brief (Laravel REST API, React
frontend, MySQL via Laravel migrations, Gemini accessed only from the Laravel
backend). They deliberately:

- Do **not** assume any pre-existing package, route, model, or convention beyond
  Laravel/React framework defaults and the assumptions explicitly marked in this
  document.
- Stay adaptable to whichever of the three deployment topologies described in §2.3's
  `A0` assumption turns out to be correct (standalone app / shared-monolith module /
  package), since nothing in the domain design (migrations, models, services,
  controllers) depends on that choice — only the deployment and inter-group
  transport details would need adjusting.
- Will need to be **reconciled** — not necessarily rewritten — once a real base
  project (if one exists elsewhere) or the client's actual technical constraints are
  provided. Any conflict should be resolved in favor of the real project's existing
  Laravel version, auth system, and conventions, per the task instruction to never
  rewrite or replace existing project architecture.

---

## 5. Summary Table — What Was Requested vs. What Was Found

| # | Requested inspection item | Found in repo? | Where it's addressed going forward |
|---|---|---|---|
| 1 | Laravel version | No | `project-analysis.md` §3.1 (recommended default) |
| 2 | PHP version | No | §3.2 |
| 3 | React setup | No | §3.3 |
| 4 | Existing frontend structure | No | §3.4; proposed structure in `architecture.md` §4 |
| 5 | Existing backend structure | No | §3.5; proposed structure in `architecture.md` §3, §5 |
| 6 | Existing routes | No | §3.6; proposed routes in `api-contract.md` |
| 7 | Existing authentication | No | §3.7; proposed approach in `architecture.md` §3 |
| 8 | Existing database configuration | No | §3.8; proposed schema in `database-design.md` |
| 9 | Existing dependencies | No | §3.9; minimal proposed package list in `architecture.md` §6 |
| 10 | Existing coding conventions | No | §3.10; assumed Laravel defaults, to be confirmed |

---

*End of `project-analysis.md`. No application code, dependencies, or configuration
were created or modified during this analysis — per the task instruction, this phase
is documentation-only.*
