# Express Backend — Folder Structure Standard

> Feature-based structure. Each business domain owns its own routes, controllers, services, and data access logic.

## TL;DR

```text
features/     → business domains (routes, controllers, services)
common/       → shared middlewares, errors, validators
database/     → MongoDB connection setup
providers/    → third-party integrations
config/       → environment configuration
utils/        → pure helper functions
```

---

## Folder Responsibilities

| Folder           | Responsibility                                         |
| ---------------- | ------------------------------------------------------ |
| `features/`      | Business domain features                               |
| `common/`        | Shared middlewares, error handlers, and validations    |
| `database/`      | MongoDB connection and lifecycle setup                 |
| `providers/`     | Third-party service integrations                       |
| `config/`        | Environment-based configuration                        |
| `utils/`         | Shared helper functions                                |
| `server.ts`      | Application bootstrap / server listen                  |
| `app.ts`         | Express app setup and global middleware registration   |

---

## Reference Layout

```text
apps/api/src/
├── server.ts
├── app.ts
│
├── features/               # auth, users, hosts, events, bookings,
│   └── <domain>/           # payments, tickets, checkin, coupons,
│                           # notifications, kyc, withdrawals,
│                           # plans, admin, platform-settings, webhooks
│
├── common/
│   ├── middlewares/        # auth.middleware.ts, role.middleware.ts
│   ├── errors/             # error-handler.middleware.ts, custom-errors.ts
│   ├── validations/        # validator.middleware.ts
│   └── constants/
│
├── database/               # mongodb.connection.ts
├── providers/              # payment/, sms/, whatsapp/, email/, storage/
├── config/                 # app, mongodb, firebase, payment, queue
└── utils/                  # date, slug, pagination
```

---

## Feature Shape

Every feature follows the same shape:

```text
features/events/
├── events.route.ts     # Express router definition
├── events.controller.ts# Request/Response handling
├── events.service.ts   # Business logic
├── events.schema.ts    # Zod validation schemas & TypeScript types
├── events.model.ts     # MongoDB/Mongoose model definitions
├── events.queue.ts     # BullMQ queue (enqueuing jobs)
└── constants/
```

> **Rule:** If a single feature requires multiple models or multiple schemas, group them into `models/` and `schemas/` folders respectively instead of using flat files (e.g., `models/event.model.ts`, `models/ticket.model.ts`).

---

## Layer Rules

| Layer        | Responsibility                          | Don't                              |
| ------------ | --------------------------------------- | ---------------------------------- |
| Route        | Define endpoints and attach middlewares | Put controller logic here          |
| Controller   | Request/response wiring only            | Put business logic here            |
| Service      | Business logic, validation, data access, orchestration | Put HTTP req/res logic here |
| Schema       | Zod request validation & TypeScript type inference | Add behavior or methods            |
| Model        | MongoDB/Mongoose schema and model definition       | Put business logic or HTTP logic here |
| Queue        | BullMQ queue instance & addJob helpers  | Process jobs here (done in worker app) |
| Provider     | Vendor SDK calls                        | Be imported directly by features — go via the abstraction |

### Route & Controller — thin

```ts
// events.route.ts
const router = Router();
router.post('/', authMiddleware, validateRequest(createEventSchema), eventsController.createEvent);
export default router;

// events.controller.ts
export const createEvent = asyncHandler(async (req: Request, res: Response) => {
  const event = await eventsService.createEvent(req.body, req.user);
  res.status(201).json(event);
});
```

### Service — business logic

```ts
// events.service.ts
export const createEvent = async (data: CreateEventDTO, user: AuthUser) => {
  // validate host, create event, return response
};
```

---

## `common/` vs `providers/` vs `config/`

| Need                                | Put it in...   |
| ----------------------------------- | -------------- |
| Auth middleware, role middleware    | `common/middlewares/` |
| Error handling, generic validation  | `common/errors/` or `common/validations/` |
| MongoDB connection lifecycle        | `database/`    |
| Stripe / Razorpay / Twilio / S3     | `providers/`   |
| Env-based config (MongoDB URI, keys)| `config/`      |
| Pure helper (date, slug, pagination)| `utils/`       |

**Never** read `process.env` outside `config/`.
**Never** import vendor SDKs outside `providers/`.
