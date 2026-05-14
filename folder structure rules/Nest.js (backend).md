# NestJS Backend — Folder Structure Standard

> Module-based structure. Each business domain owns its own module.

## TL;DR

```
modules/      → business domains
common/       → shared guards, decorators, filters, pipes
providers/    → third-party integrations
config/       → environment configuration
utils/        → pure helper functions
```

---

## Folder Responsibilities

| Folder           | Responsibility                                         |
| ---------------- | ------------------------------------------------------ |
| `modules/`       | Business domain modules                                |
| `common/`        | Shared guards, decorators, filters, pipes, interceptors |
| `providers/`     | Third-party service integrations                       |
| `config/`        | Environment-based configuration                        |
| `utils/`         | Shared helper functions                                |
| `main.ts`        | Application bootstrap                                  |
| `app.module.ts`  | Root module registration                               |

---

## Reference Layout

```
apps/api/src/
├── main.ts
├── app.module.ts
│
├── modules/                # auth, users, hosts, events, bookings,
│   └── <domain>/           # payments, tickets, checkin, coupons,
│                           # notifications, kyc, withdrawals,
│                           # plans, admin, platform-settings, webhooks
│
├── common/
│   ├── guards/             # firebase-auth.guard.ts, roles.guard.ts
│   ├── decorators/         # current-user.decorator.ts, roles.decorator.ts
│   ├── filters/            # http-exception.filter.ts
│   ├── interceptors/       # response.interceptor.ts
│   ├── pipes/              # validation.pipe.ts
│   └── constants/
│
├── providers/              # payment/, sms/, whatsapp/, email/, storage/
├── config/                 # app, database, firebase, payment, queue
└── utils/                  # date, slug, pagination
```

---

## Module Shape

Every module follows the same shape:

```
modules/events/
├── events.module.ts
├── events.controller.ts
├── events.service.ts
├── dto/                # create-event, update-event, event-filter
├── entities/           # event.entity.ts
├── repositories/       # events.repository.ts
├── guards/             # event-owner.guard.ts (module-specific only)
└── constants/
```

---

## Layer Rules

| Layer        | Responsibility                          | Don't                              |
| ------------ | --------------------------------------- | ---------------------------------- |
| Controller   | Request/response wiring only            | Put business logic here            |
| Service      | Business logic, validation, orchestration | Write raw DB queries              |
| Repository   | Database query logic                    | Contain business rules             |
| DTO          | Request validation & input shape        | Add behavior or methods            |
| Provider     | Vendor SDK calls                        | Be imported directly by modules — go via the abstraction |

### Controller — thin

```ts
@Post()
createEvent(@Body() dto: CreateEventDto, @CurrentUser() user: AuthUser) {
  return this.eventsService.createEvent(dto, user);
}
```

### Service — business logic

```ts
async createEvent(dto: CreateEventDto, user: AuthUser) {
  // validate host, create event, return response
}
```

---

## `common/` vs `providers/` vs `config/`

| Need                                | Put it in...   |
| ----------------------------------- | -------------- |
| Auth guard, role guard, decorator   | `common/`      |
| Exception filter, interceptor, pipe | `common/`      |
| Stripe / Razorpay / Twilio / S3     | `providers/`   |
| Env-based config (DB URL, keys)     | `config/`      |
| Pure helper (date, slug, pagination)| `utils/`       |

**Never** read `process.env` outside `config/`.
**Never** import vendor SDKs outside `providers/`.