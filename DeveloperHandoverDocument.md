

# Developer Handover Document - KT Events

| | |
|---|---|
| **Version** | 1.0 |
| **Owner** | Kabani |
| **Scope** | All features across backend (Express), frontend (Next.js), worker (BullMQ) |

## The Rule

```text
Feature Complete = Implementation ✓ + Testing ✓ + Documentation ✓
```

Docs are committed in the **same PR** as the feature.

## Directory Structure

```text
dev-docs/
├── backend/                 # Express — one .api.md + one .postman.json per feature
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
| An Express route/controller | `backend/<feature>.api.md` **+** `.postman.json` |
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

Must include: full collection, environment variables (`{{baseUrl}}`, , …), one sample request **and** saved sample response per endpoint, and at least one authenticated example.

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
