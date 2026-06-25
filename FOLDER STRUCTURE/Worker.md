# Worker — Folder Structure Standard

**Stack:** Express _(health + Bull Board only)_ · TypeScript · BullMQ · Redis · Prisma/PostgreSQL

## TL;DR

One worker app, one folder per domain. Each domain owns 4 files: `queue` (enqueue), `worker` (consume), `processor` (logic), `types` (payload). Processors never touch SDKs directly — they call `providers/`. Queue names live in one enum. Every processor is idempotent and retry-safe.

```text
modules/       → domain queues, workers, logic
providers/     → 3rd-party SDK wrappers
config/        → redis, bullmq defaults, env
cron/          → scheduled job enqueuers
common/        → shared constants/types/logger
utils/         → pure functions
dev-docs/      → per-module docs
```

## Folder responsibilities

| Folder       | Responsibility                                                           |
| ------------ | ------------------------------------------------------------------------ |
| `modules/`   | Domain queues, workers, logic (no cross-module imports except via queue) |
| `providers/` | 3rd-party SDK wrappers (the _only_ place an SDK is imported)             |
| `config/`    | Redis, bullmq defaults, env (one shared Redis connection)                |
| `cron/`      | Scheduled job enqueuers (enqueue only, never run logic inline)           |
| `common/`    | Shared constants/types/logger (no business logic)                        |
| `utils/`     | Pure functions (no I/O, no side effects)                                 |
| `dev-docs/`  | Per-module docs (required before merge)                                  |

## Reference Layout

```text
apps/worker/
├── src/
│   ├── index.ts                 # boot: env → redis → register workers → start cron → health server
│   ├── modules/                 # one folder per domain (identical 4-file pattern)
│   │   ├── email/
│   │   │   ├── email.queue.ts        # exports queue + typed addJob() helpers
│   │   │   ├── email.worker.ts       # new Worker(name, processor, { concurrency })
│   │   │   ├── email.processor.ts    # business logic, calls providers
│   │   │   └── email.types.ts        # job-name enum + payload interfaces
│   │   ├── notifications/  … (same 4 files)
│   │   ├── payments/       … (same)
│   │   ├── reports/        … (same)
│   │   └── webhooks/       … (same)
│   ├── providers/               # thin wrappers over 3rd-party SDKs
│   │   ├── email/    resend.provider.ts · nodemailer.provider.ts
│   │   ├── storage/  minio.provider.ts · s3.provider.ts
│   │   ├── payment/  razorpay.provider.ts
│   │   ├── push/     firebase.provider.ts
│   │   └── sms/      twilio.provider.ts
│   ├── config/      env.ts · redis.ts · bullmq.ts     # single connection + default job opts
│   ├── cron/        reminders · cleanup · reports      # scheduled *enqueuers* (see §6)
│   ├── common/      constants/ types/ helpers/ logger/
│   └── utils/       date.ts · retry.ts · sleep.ts
├── dev-docs/worker/  email.md · notifications.md · …   # one per module
├── package.json · tsconfig.json · Dockerfile
```

## Module anatomy (every domain is identical)

| File             | Role                                      | Imports                  |
| ---------------- | ----------------------------------------- | ------------------------ |
| `*.queue.ts`     | creates `Queue`, exposes typed `addX()`   | config, types            |
| `*.worker.ts`    | creates `Worker`, sets concurrency/events | processor, config        |
| `*.processor.ts` | the actual work                           | providers, prisma, types |
| `*.types.ts`     | job-name enum + payload interfaces        | —                        |

**Flow:** `API → queue.add() → Redis → Worker → Processor → Provider → External`

Swap a provider = one file change, processors untouched.

## Queue registry (single source of truth)

```ts
// common/constants/queues.ts
export enum QueueName {
  Email = "email",
  Notification = "notification",
  Payment = "payment",
  Report = "report",
  Webhook = "webhook",
}
```

Never type a queue name as a string literal anywhere else.

## Reliability defaults (set in `config/bullmq.ts`)

| Concern     | Standard                                                               |
| ----------- | ---------------------------------------------------------------------- |
| Retries     | `attempts: 3`, `backoff: { type: 'exponential', delay: 5000 }`         |
| Idempotency | pass `jobId` to dedupe; payment/webhook processors must be re-run-safe |
| Retention   | `removeOnComplete: 1000`, `removeOnFail: 5000`                         |
| Failures    | log + emit on `failed`; exhausted → dead-letter queue                  |
| Shutdown    | SIGTERM → `await worker.close()` + `queue.close()` (no half-done jobs) |
| Concurrency | per-worker, tuned per domain (email high, payment low)                 |

## Rules

1. One queue per domain; name comes from `QueueName` enum only.
2. Processor logic never imports an SDK — go through `providers/`.
3. Every payload typed in `*.types.ts`; no `any` jobs.
4. Processors are idempotent (assume every job can run twice).
5. `cron/` enqueues, never executes.
6. No cross-module calls — communicate by adding to another queue.
7. No module merges without its `dev-docs/worker/*.md`.
8. Graceful shutdown wired in `index.ts` for every worker.

---
