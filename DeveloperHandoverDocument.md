Quick correction before the rewrite: your stack is **NestJS** (backend) and **BullMQ** (worker), not Express — I've fixed both headings. Also normalized `notification.md` → `notifications.md` to match the tree. Here's the tightened standard inline for review:

---

# Documentation Standard — KT Events

| | |
|---|---|
| **Version** | 1.0 |
| **Owner** | Kabani |
| **Reviewer** | Mohamed Shafeeq |
| **Scope** | All features across backend (NestJS), frontend (Next.js), worker (BullMQ) |

## TL;DR

> A feature is **not done** until code + tests + docs are merged. Docs live under `dev-docs/`, one set per feature, following the fixed file-naming and section rules below. No doc → no merge.

## The Rule

```text
Feature Complete = Implementation ✓ + Testing ✓ + Documentation ✓
```

Docs are committed in the **same PR** as the feature. A PR without its docs is not review-ready.

## Directory Structure

```text
dev-docs/
├── backend/                 # NestJS — one .api.md + one .postman.json per feature
│   ├── <feature>.api.md     #   human-readable API contract
│   └── <feature>.postman.json  # importable collection (requests + samples + env)
├── frontend/                # Next.js — one .md per feature/module
│   └── <feature>.md
└── worker/                  # BullMQ — one .md per queue/processor
    └── <feature>.md
```

## File Naming

| Layer | Pattern | Example |
|---|---|---|
| Backend (contract) | `<feature>.api.md` | `bookings.api.md` |
| Backend (collection) | `<feature>.postman.json` | `bookings.postman.json` |
| Frontend | `<feature>.md` | `events.md` |
| Worker | `<feature>.md` | `email.md` |

Rules: lowercase, singular-or-plural to match the module name, one `<feature>` token only, no spaces/underscores.

## What To Create (decision table)

| If the feature touches… | Create |
|---|---|
| A NestJS module/controller | `backend/<feature>.api.md` **+** `.postman.json` |
| A Next.js page/route/component | `frontend/<feature>.md` |
| A BullMQ queue/processor | `worker/<feature>.md` |
| All three | all three sets |

---

## Backend — `<feature>.api.md`

*Required sections, in order:*

1. **Feature Overview**
2. **Authentication Requirements** — Firebase token? guest? public?
3. **Endpoints** — method + path table
4. **Request Payloads** — per endpoint
5. **Response Examples** — success shapes
6. **Validation Rules** — DTO / class-validator constraints
7. **Error Responses** — status + code + message table
8. **Business Rules** — invariants, atomic guards, side effects
9. **Notes** — gotchas, links to related docs

## Backend — `<feature>.postman.json`

Must include: full collection, environment variables (`{{baseUrl}}`, `{{token}}`, …), one sample request **and** saved sample response per endpoint, and at least one authenticated example.

---

## Frontend — `<feature>.md`

```md
# <Feature> Module

## Overview
One-paragraph purpose.

## Pages
### <Page Name>
- Behaviors, states, edge cases

## Components
### <ComponentName>
Purpose and where it's used.

## API Integration
Endpoints consumed (link to backend .api.md).

## State Management
Zustand stores used.

## Validation
Zod schemas used.

## Notes
Implementation details, flags, TODOs.
```

---

## Worker — `<feature>.md`

```md
# <Feature> Worker

## Overview
Worker responsibility.

## Queue
Queue name + config (concurrency, rate limit).

## Job Types
### <Job Name>
Payload + description.

## Flow
Producer → Queue → Worker → Provider

## Retry Strategy
Attempts + backoff (e.g. exponential, 3 attempts).

## Failure Handling
DLQ / failed-job handling + logging.

## Monitoring
Bull Board / logs.

## Notes
Implementation details.
```

---

## Definition of Done

```text
[ ] Implementation complete
[ ] Tests pass
[ ] Backend: .api.md + .postman.json (if API touched)
[ ] Frontend: <feature>.md (if UI touched)
[ ] Worker: <feature>.md (if queue touched)
[ ] Docs committed under dev-docs/ in the same PR
[ ] Review-ready (assigned to Mohamed Shafeeq)
```

---

Key changes from your draft: corrected Express → NestJS/BullMQ, collapsed the three repeated folder trees into one canonical tree with inline comments, added a naming-convention table and a "what to create" decision table, gave each layer a single copy-paste skeleton, and folded the DoD into one checklist.

Want me to export this as `dev-docs/DOCUMENTATION_STANDARD.md`?