# Sprint 2 — Ticket Delivery & Attendee Check-in

| Field    | Value                              |
| -------- | ---------------------------------- |
| Sprint   | 2 of N                             |
| Duration | (25-may-2026 – 29-may-2026) (5WD)  |
| Owner    | Developer B                        |
| Reviewer | MOHAMED SHAFEEQ                    |
| Status   | Ready for development              |

---

## 1. Context
*Why this sprint exists. One short paragraph.*

> Sprint 1 confirms payments and generates QR codes, but two gaps remain: the attendee never receives their ticket, and the host has no way to validate it at the venue. This sprint closes the loop — email tickets on payment success, give the attendee a re-download link, and build a host scanner that handles concurrent check-ins safely.

---

## 2. Goals
*What must be true when the sprint ends.*

- A `PAID` booking triggers an email with a PDF attachment containing one QR per ticket, within 60 seconds.
- An attendee can re-download their ticket from a signed link without logging in.
- A host can open a check-in page on `web-host`, scan a QR with the device camera, and see pass/fail instantly.
- A host can enter a short backup code manually if a QR is damaged.
- The same QR cannot be checked in twice, even from two devices at the same moment.
- Every check-in is logged with who scanned it, when, and from where.

---

## 3. Non-Goals
*What we are not doing this sprint. This stops scope creep.*

- Native mobile scanner app.
- Offline check-in mode.
- Bulk re-send of tickets to a whole event.
- Refund or transfer flows.
- Wristband / RFID / NFC integration.

---

## 4. Technical Design
*The senior has already made these decisions. The junior does not need to re-decide them.*

### 4.1 Worker job
`send-ticket-email` on BullMQ. Enqueued from the Razorpay webhook handler after the booking is marked `PAID`. **Idempotency key: `bookingId`** — re-enqueuing the same booking is a no-op.

### 4.2 PDF generation
Generated in `apps/worker` using `pdfkit` (server-side, no Chromium dependency). One page per ticket, embedded QR (reuses the HMAC payload from Sprint 1) and an 8-character human-readable backup code.

### 4.3 Email delivery
Reuse the SendGrid / SES setup from Sprint 1 (`EX-3`). Worker job sends the PDF as attachment. Retries: 3 with exponential backoff. Failure after retries → mark booking `email_failed_at` and surface in logs.

### 4.4 Re-download link
`GET /t/:ticketId?sig=<hmac>` on `web-public`. Same HMAC secret as the QR payload. Signature carries a 30-day expiry. Invalid or expired sig → `403`.

### 4.5 Check-in atomicity
Critical. A single Prisma transaction with `updateMany` + row count check:

```ts
const updated = await prisma.ticket.updateMany({
  where: { id: ticketId, status: 'ISSUED' },
  data:  { status: 'CHECKED_IN', checkedInAt: new Date() },
});
if (updated.count === 0) {
  // either ticket doesn't exist or already checked in — fetch to differentiate
  throw new ConflictException('ALREADY_CHECKED_IN');
}
await prisma.checkInLog.create({ data: { ticketId, scannedBy, deviceLabel } });
```

The status-guarded `updateMany` is what makes concurrent scans safe — only one wins. Do **not** use `findUnique` + `update`; that has a TOCTOU window.

### 4.6 QR verification
Reuse the HMAC verify helper from Sprint 1. Reject:
- foreign-event QR → `400 WRONG_EVENT`
- booking not `PAID` → `409 BOOKING_NOT_PAID`
- bad signature → `400 INVALID_QR`

### 4.7 Scanner UX
`@zxing/browser` in `web-host` for camera scanning. Manual entry input accepts the 8-char backup code printed on the PDF.

### 4.8 Rate limiting
Check-in endpoint: 30 requests/min per host user via NestJS `ThrottlerGuard`.

### 4.9 Data model

```prisma
model Ticket {
  id           String       @id @default(uuid())
  bookingId    String
  qrCode       String       @unique
  backupCode   String       @unique  // 8-char human-typeable
  status       TicketStatus @default(ISSUED)
  checkedInAt  DateTime?
  issuedAt     DateTime     @default(now())

  booking      Booking      @relation(fields: [bookingId], references: [id])
  checkInLogs  CheckInLog[]
}

model CheckInLog {
  id           String   @id @default(uuid())
  ticketId     String
  scannedBy    String   // userId of the host/staff
  deviceLabel  String?
  scannedAt    DateTime @default(now())

  ticket       Ticket   @relation(fields: [ticketId], references: [id])

  @@index([ticketId])
}

enum TicketStatus {
  ISSUED
  CHECKED_IN
  VOID
}
```

> `Ticket.status` and `backupCode` are additions to Sprint 1's `Ticket` model. Coordinate the migration with Developer A's schema changes.

---

## 5. Tasks
*Broken into small, testable pieces. Each task should take half a day or less.*

### Backend
- [ ] **BE-1** Extend Prisma schema: add `status`, `backupCode`, `checkedInAt` on `Ticket`; add `CheckInLog`; add `TicketStatus` enum. Run migration.
- [ ] **BE-2** Update QR generation utility from Sprint 1 to also emit an 8-char backup code (collision-checked).
- [ ] **BE-3** Implement `send-ticket-email` BullMQ processor in `apps/worker` (retry + backoff, idempotent on `bookingId`).
- [ ] **BE-4** Implement PDF generator module (booking → Buffer) using `pdfkit`; unit-tested on fixtures.
- [ ] **BE-5** Enqueue the job from the Razorpay webhook handler (replaces the inline email send from Sprint 1 BE-6).
- [ ] **BE-6** Implement `GET /api/tickets/:id` — verifies `sig` query, returns a short-lived download URL for the PDF.
- [ ] **BE-7** Implement `POST /api/host/events/:eventId/check-in` — input `{ qrPayload?, backupCode? }`, atomic update per 4.5, returns ticket + attendee summary or `409`.
- [ ] **BE-8** Apply `ThrottlerGuard` (30/min/user) to the check-in route.
- [ ] **BE-9** Unit tests: HMAC verify, PDF generator, worker idempotency.
- [ ] **BE-10** Integration test: two concurrent check-ins on the same ticket → exactly one `CHECKED_IN`, one `409`.

### Frontend
- [ ] **FE-1** Check-in page at `/events/:id/check-in` in `web-host` — camera view, last-5-scans list, success/error toasts.
- [ ] **FE-2** Manual backup-code entry form on the same page for damaged QRs.
- [ ] **FE-3** Ticket re-download page `/t/:ticketId` in `web-public` — event details + "Download Ticket" button.
- [ ] **FE-4** Dev-only `/dev/email-preview` route to inspect rendered email + PDF (gated by feature flag).

### External services
- [ ] **EX-1** Configure BullMQ Redis connection in `apps/worker` env (staging).
- [ ] **EX-2** Add a SendGrid / SES sandbox sender for `dev@kt-events` to test attachments without sending to real inboxes.

---

## 6. Acceptance Criteria
*How the senior will verify the sprint is done.*

- [ ] A `PAID` booking with N tickets results in exactly one email with one N-page PDF within 60s.
- [ ] Re-enqueuing the same `bookingId` does **not** send a duplicate email.
- [ ] Scanning the same QR twice returns `409 ALREADY_CHECKED_IN` with the original timestamp.
- [ ] Scanning a QR from a different event returns `400 WRONG_EVENT`.
- [ ] Scanning a QR whose booking is not `PAID` returns `409 BOOKING_NOT_PAID`.
- [ ] Two concurrent check-in requests for the same ticket → exactly one `CHECKED_IN`, one `409` (verified by integration test BE-10).
- [ ] `/t/:ticketId` with an invalid or expired `sig` returns `403`.
- [ ] Check-in endpoint returns `429` after 30 requests/min per user.
- [ ] Manual backup-code entry checks in successfully when the QR is unavailable.

---

## 7. References
- Sprint 1 (Public Booking) — HMAC payload format, webhook handler, ticket schema
- Attendance service review notes — atomic `updateMany` pattern for state guards
- [`pdfkit` docs](https://pdfkit.org/), [`@zxing/browser` docs](https://github.com/zxing-js/browser)
- Internal: PRD section 9 (Check-in), System design diagram v1.

---

## 8. Definition of Done
- [ ] All tasks above checked off.
- [ ] All acceptance criteria pass on staging.
- [ ] Code merged to `develop` behind a feature flag if needed.
- [ ] PR reviewed and approved by senior.
- [ ] API endpoints documented in Postman / Swagger.
- [ ] Short demo recorded or shown in sprint review.
