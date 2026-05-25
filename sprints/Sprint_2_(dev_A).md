# Sprint 2 — Host Event Management

| Field    | Value                              |
| -------- | ---------------------------------- |
| Sprint   | 2 of N                             |
| Duration | (25-may-2026 – 29-may-2026) (5WD)  |
| Owner    | Developer A                        |
| Reviewer | MOHAMED SHAFEEQ                    |
| Status   | Ready for development              |

---

## 1. Context
*Why this sprint exists. One short paragraph.*

> Sprint 1 delivered auth — hosts can sign in, but they cannot do anything yet. The platform has no way to create the events that everything else (booking, tickets, check-in) depends on. This sprint gives hosts the ability to onboard, create events, manage ticket types, and publish them to the public site.

---

## 2. Goals
*What must be true when the sprint ends.*

- A signed-in host completes a one-time onboarding form before reaching their dashboard.
- A host can create, edit, and delete events they own.
- A host can add, edit, and remove ticket types on their own events.
- A host can upload a cover image for an event.
- A host can move an event through `DRAFT → PUBLISHED → PAUSED → ENDED / CANCELLED`.
- A published event appears on the public event link consumed by Developer B's booking flow.

---

## 3. Non-Goals
*What we are not doing this sprint. This stops scope creep.*

- Payout configuration (Razorpay Route / bank verification) — placeholder fields only.
- Recurring or multi-session events.
- Sales dashboards, analytics, attendee lists.
- Promo codes / discounts (separate sprint).
- Admin approval or moderation workflow.

---

## 4. Technical Design
*The senior has already made these decisions. The junior does not need to re-decide them.*

### 4.1 Event lifecycle

| Status      | Meaning                                            |
| ----------- | -------------------------------------------------- |
| `DRAFT`     | Created, not visible publicly                      |
| `PUBLISHED` | Visible on public link, bookable                   |
| `PAUSED`    | Visible but not bookable                           |
| `ENDED`     | Event date has passed                              |
| `CANCELLED` | Host cancelled, refunds out of scope this sprint   |

Transitions are server-side only. The client sends an action (`publish`, `pause`, `cancel`), never a raw status.

### 4.2 Slug
`lower-kebab(title) + "-" + 6-char nanoid`. Generated on first create. **Immutable after first publish** so public links never break.

### 4.3 Publish gate
An event can move to `PUBLISHED` only if it has: title, description, start/end datetime, venue, cover image, and at least one ticket type with `quantityAvailable > 0`. Backend rejects with `409` and the missing field name.

### 4.4 Authorization
- `@UseGuards(FirebaseAuthGuard, RolesGuard)` with `@Roles('HOST')` on all routes.
- Resource ownership enforced in the service layer with `where: { hostId: req.user.id }` — same pattern Developer A established in Sprint 1.

### 4.5 Image upload
- Firebase Storage. Client uploads directly using a short-lived signed URL minted by the API.
- Endpoint enforces mime (`image/jpeg | image/png | image/webp`) and max size (5 MB).
- Only the storage path is persisted on the event row.

### 4.6 Ticket type guard
A ticket type that already has bookings (any status except `EXPIRED`) cannot be deleted or have its `price` reduced. Backend returns `409 TICKET_TYPE_HAS_BOOKINGS`.

### 4.7 Data model

```prisma
model Event {
  id           String       @id @default(uuid())
  hostId       String
  title        String
  slug         String       @unique
  description  String
  venue        String
  startsAt     DateTime
  endsAt       DateTime
  coverImage   String?
  status       EventStatus  @default(DRAFT)
  publishedAt  DateTime?
  createdAt    DateTime     @default(now())
  updatedAt    DateTime     @updatedAt
  deletedAt    DateTime?

  host         User         @relation(fields: [hostId], references: [id])
  ticketTypes  TicketType[]

  @@index([hostId, status])
}

model TicketType {
  id                 String   @id @default(uuid())
  eventId            String
  name               String
  price              Int      // in paise
  quantityAvailable  Int
  salesStartAt       DateTime?
  salesEndAt         DateTime?
  createdAt          DateTime @default(now())

  event              Event    @relation(fields: [eventId], references: [id])
}

enum EventStatus {
  DRAFT
  PUBLISHED
  PAUSED
  ENDED
  CANCELLED
}
```

### 4.8 Pagination
Cursor-based: `?cursor=<base64(lastId)>&limit=<n>`, default 20, max 50. Response includes `nextCursor` or `null`.

---

## 5. Tasks
*Broken into small, testable pieces. Each task should take half a day or less.*

### Backend
- [ ] **BE-1** Add `Event`, `TicketType`, and `EventStatus` to Prisma schema and run migration.
- [ ] **BE-2** Implement `POST /api/host/onboarding` — accepts org name, contact, payout placeholders, marks `User.onboardedAt`.
- [ ] **BE-3** Implement `EventsController` CRUD: `POST /api/host/events`, `GET /api/host/events`, `GET /api/host/events/:id`, `PATCH /api/host/events/:id`, `DELETE /api/host/events/:id` (soft delete).
- [ ] **BE-4** Implement `TicketTypesController`: `POST /api/host/events/:eventId/ticket-types`, `PATCH /:id`, `DELETE /:id` with the booking-guard from 4.6.
- [ ] **BE-5** Implement `EventService.transition(eventId, action)` — single source of truth for `publish`, `pause`, `cancel`. Reject invalid transitions with `409`.
- [ ] **BE-6** Implement publish-gate validator (4.3); returns the first missing field name on failure.
- [ ] **BE-7** Implement `POST /api/host/uploads/event-cover` — returns a Firebase Storage signed URL (5-min expiry, mime + size constraints).
- [ ] **BE-8** Implement `GET /api/events/:slug` — **public** endpoint consumed by Developer B's booking flow. Returns event + ticket types only if `PUBLISHED`.
- [ ] **BE-9** Add cursor-pagination utility in shared lib; apply to `GET /api/host/events`.
- [ ] **BE-10** Unit tests for `EventService.transition()` and the publish-gate validator.

### Frontend
- [ ] **FE-1** Onboarding screen at `/onboarding` in `web-host`. Blocks dashboard until `User.onboardedAt` is set.
- [ ] **FE-2** Events list page `/events` — empty state, status badges, title search.
- [ ] **FE-3** Event create/edit form at `/events/new` and `/events/[id]/edit` — sections: details, venue, schedule, cover, ticket types.
- [ ] **FE-4** Ticket-type sub-form inside the event form (add/remove rows, total capacity preview).
- [ ] **FE-5** Cover image upload component using the signed URL from BE-7 (preview + progress).
- [ ] **FE-6** Publish / pause / cancel actions with confirmation modal; surface 409 errors with the missing field clearly highlighted.

### External services
- [ ] **EX-1** Create Firebase Storage bucket for event covers; set CORS for the `web-host` origin.
- [ ] **EX-2** Add storage service account permission to backend env (reuse Sprint 1 Admin SDK credentials where possible).

---

## 6. Acceptance Criteria
*How the senior will verify the sprint is done.*

- [ ] A host cannot reach `/events` until onboarding is submitted.
- [ ] Creating an event with valid input returns `201` and persists with `status = DRAFT`.
- [ ] Editing `slug` after first publish returns `409 SLUG_LOCKED`.
- [ ] Publishing an event missing any required field returns `409` naming the missing field.
- [ ] Cover image upload >5 MB or wrong mime returns `400` from the signed-URL endpoint.
- [ ] Deleting a ticket type that has bookings returns `409 TICKET_TYPE_HAS_BOOKINGS`.
- [ ] Host A cannot read, edit, or delete Host B's events (returns `404`, never `403`, to avoid leaking existence).
- [ ] `GET /api/events/:slug` returns `404` for `DRAFT`, `PAUSED`, `ENDED`, or `CANCELLED` events.
- [ ] `GET /api/host/events` returns paginated results with `nextCursor`.

---

## 7. References
- Sprint 1 (Auth) — `FirebaseAuthGuard`, `RolesGuard`, `User` model
- [Firebase Storage signed URL docs](https://firebase.google.com/docs/storage/web/upload-files)
- Internal: PRD section 5 (Host Event Management), System design diagram v1.

---

## 8. Definition of Done
- [ ] All tasks above checked off.
- [ ] All acceptance criteria pass.
- [ ] Code merged to `develop` behind a feature flag if needed.
- [ ] PR reviewed and approved by senior.
- [ ] API endpoints documented in Postman / Swagger.
- [ ] Short demo recorded or shown in sprint review.
