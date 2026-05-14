# Worker — Folder Structure Standard

> Domain-based job structure. Jobs grouped by business domain, not flat.

## TL;DR

```
jobs/         → domain-based job handlers
queues/       → BullMQ queue definitions
processors/   → queue processors
services/     → shared worker services (email, PDF, QR, storage)
common/       → logger, redis, constants, retry config
config/       → worker / redis / queue config
```

---

## Folder Responsibilities

| Folder         | Responsibility                                     |
| -------------- | -------------------------------------------------- |
| `jobs/`        | Background job handlers, grouped by domain         |
| `queues/`      | BullMQ queue definitions                           |
| `processors/`  | Queue processors that dispatch to jobs             |
| `services/`    | Shared services (email, SMS, PDF, QR, storage)     |
| `common/`      | Logger, Redis connection, constants, retry config  |
| `config/`      | Worker, Redis, queue configuration                 |
| `main.ts`      | Worker bootstrap                                   |

---

## Reference Layout

```
apps/worker/src/
├── main.ts
│
├── jobs/                       # Grouped by domain
│   ├── notifications/          # send-email, send-sms, send-whatsapp
│   ├── tickets/                # generate-qr, generate-badge, resend-ticket
│   ├── bookings/               # booking-expiry, booking-reminder, booking-confirmation
│   ├── billing/                # generate-invoice-pdf, process-payout, retry-payout
│   └── webhooks/               # payment-webhook-retry, failed-webhook-retry
│
├── queues/                     # one queue per domain
│   ├── notification.queue.ts
│   ├── ticket.queue.ts
│   ├── booking.queue.ts
│   ├── billing.queue.ts
│   └── webhook.queue.ts
│
├── processors/                 # one processor per domain
│   └── <domain>.processor.ts
│
├── services/                   # email, sms, pdf, qr, storage
├── common/                     # logger.ts, redis.ts, queue.constants.ts, retry.config.ts
└── config/                     # redis.config, queue.config, worker.config
```

---

## Domain → Jobs Mapping

| Domain          | Example Jobs                                        |
| --------------- | --------------------------------------------------- |
| `notifications` | Send email, SMS, WhatsApp                           |
| `tickets`       | Generate QR, generate badge, resend ticket          |
| `bookings`      | Expire unpaid booking, send reminders               |
| `billing`       | Generate invoice PDF, process payout, retry payout  |
| `webhooks`      | Retry failed payment webhooks                       |

---

## Job Naming

Format: `<verb>-<noun>.job.ts` — one clear task per file.

| ✅ Good                     | ❌ Bad                |
| --------------------------- | --------------------- |
| `send-email.job.ts`         | `notification.job.ts` |
| `generate-qr.job.ts`        | `ticket.job.ts`       |
| `booking-expiry.job.ts`     | `booking.job.ts`      |
| `process-payout.job.ts`     | `helper.job.ts`       |
| `payment-webhook-retry.job.ts` | `main-job.ts`     |

The **folder** says the domain. The **filename** says the exact task.

---

## Rules

1. One job = one task. If a job does two things, split it.
2. Jobs must be **idempotent** — they retry on failure.
3. Vendor SDKs (email, SMS, storage) go through `services/`, not directly in jobs.
4. No `process.env` outside `config/`.