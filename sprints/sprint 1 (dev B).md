# Sprint 1 — Public Booking Flow (Guest Checkout)

| Field | Value |
|---|---|
| Sprint | 1 of N |
| Duration | (14-may-2026 – 20-may-2026) (5WD) |
| Owner | Developer B |
| Reviewer | MOHAMED SHAFEEQ |
| Status | Ready for development |

## 1. Context
*Why this sprint exists. One short paragraph.*

This is the core revenue path of the platform: a customer opens a shared event link, picks tickets, verifies via OTP, pays through Razorpay, and receives a QR-coded ticket on email — without signing up. Developer B builds this in parallel with Developer A's auth sprint. The OTP and user-creation pieces depend on Developer A's Firebase work; both developers must agree on the User model and Firebase setup on Day 1 so this sprint is not blocked.

## 2. Goals
*What must be true when the sprint ends.*

- A customer can open a public event link and view ticket types without logging in.
- A customer can select a ticket type, quantity, and apply an optional coupon code.
- A customer can complete OTP verification via email or phone (their choice).
- A verified customer is auto-created as a `CUSTOMER` user, or matched to an existing row by email or phone.
- A customer can pay through Razorpay Checkout opened via SDK.
- A booking is marked `PAID` only by a verified Razorpay webhook — never from the client.
- A QR-coded ticket is generated and emailed on payment success.

## 3. Non-Goals
*What we are not doing this sprint. This stops scope creep.*

- Logged-in customer bookings (later sprint).
- Refunds, cancellations, or rescheduling.
- Host or admin dashboards.
- Multi-currency or international payments.
- SMS and WhatsApp delivery (only if the plan flag is already wired; otherwise later sprint).
- Seat selection or reserved seating.

## 4. Technical Design
*The senior has already made these decisions. The junior does not need to re-decide them.*

### 4.1 Payment provider
Razorpay. Standard Checkout opened via Razorpay's web SDK. Vendor lock-in accepted.

### 4.2 OTP provider
Firebase Auth (delivered by Developer A in the parallel sprint). Customer chooses email link or phone OTP at checkout time.

### 4.3 Booking lifecycle
| Status | Meaning |
|---|---|
| `PENDING` | Booking row created, awaiting payment |
| `PAID` | Razorpay webhook confirmed payment, QR issued |
| `FAILED` | Payment attempted and failed |
| `EXPIRED` | Pending booking older than 15 minutes, auto-cleaned |

Status transitions are server-side only. The client never sends the status.

### 4.4 Checkout flow
1. Client calls `POST /api/checkout/initiate` with `eventId`, `ticketTypeId`, `quantity`, optional `couponCode`.
2. Backend validates ticket availability and coupon, returns a `bookingId` (status = `PENDING`) and the final amount.
3. Client triggers Firebase OTP using the flow built by Developer A. On success, backend matches or creates the user and links `firebaseUid` to the booking.
4. Client calls `POST /api/payment/order` with `bookingId`. Backend creates a Razorpay order and returns `orderId` + key.
5. Client opens Razorpay Checkout SDK with the order.
6. Razorpay calls `POST /api/webhooks/razorpay` on payment events. Backend verifies the signature, marks booking `PAID`, generates QR, sends email.
7. Client polls `GET /api/bookings/:id` (or uses SDK callback) and shows the confirmation page.

### 4.5 Auto user creation (shared with Developer A)
- If email or phone already maps to a `User`, link the booking to that user.
- Otherwise create a new `User` with `role = CUSTOMER` and fill `firebaseUid` on OTP verification.
- This logic lives in the auth module owned by Developer A. Developer B **calls** it, does not re-implement it.

### 4.6 Webhook security
- Razorpay signature **must** be verified using the webhook secret. Reject on mismatch.
- The webhook handler must be idempotent — Razorpay can retry. Use `paymentId` as the dedupe key.
- Never trust client-side payment success callbacks for marking `PAID`.

### 4.7 Data model
```prisma
model Booking {
  id            String        @id @default(uuid())
  userId        String
  eventId       String
  ticketTypeId  String
  quantity      Int
  couponId      String?
  amount        Int           // in paise
  status        BookingStatus @default(PENDING)
  razorpayOrderId   String?   @unique
  razorpayPaymentId String?   @unique
  createdAt     DateTime      @default(now())
  paidAt        DateTime?
  user          User          @relation(fields: [userId], references: [id])
}

model Ticket {
  id         String   @id @default(uuid())
  bookingId  String
  qrCode     String   @unique
  issuedAt   DateTime @default(now())
  booking    Booking  @relation(fields: [bookingId], references: [id])
}

enum BookingStatus {
  PENDING
  PAID
  FAILED
  EXPIRED
}
```

### 4.8 QR code
Generated server-side after webhook confirms payment. Stored as a signed string (`bookingId` + `ticketId` + HMAC). Rendered as a PNG and attached to the confirmation email.

## 5. Tasks
*Broken into small, testable pieces. Each task should take half a day or less.*

### Backend
- [ ] BE-1 Create `Booking` and `Ticket` Prisma models and run migration (coordinate with Developer A on `User`).
- [ ] BE-2 Implement `POST /api/checkout/initiate` — validates ticket, coupon, creates `PENDING` booking.
- [ ] BE-3 Implement `POST /api/payment/order` — creates Razorpay order, stores `razorpayOrderId`.
- [ ] BE-4 Implement `POST /api/webhooks/razorpay` — signature verification, idempotency, status update.
- [ ] BE-5 Implement QR code generation utility (signed token + PNG).
- [ ] BE-6 Implement confirmation email service (HTML template + QR attachment).
- [ ] BE-7 Implement `GET /api/bookings/:id` — returns booking status for the polling client.
- [ ] BE-8 Add cron / scheduled job to expire `PENDING` bookings older than 15 minutes.
- [ ] BE-9 Unit tests for webhook (valid signature, invalid signature, replay).

### Frontend
- [ ] FE-1 Public event page — fetch event, render ticket types, quantity selector, coupon input.
- [ ] FE-2 Checkout page — name, email, phone, OTP channel selector.
- [ ] FE-3 OTP verification screen (consumes Firebase flow from Developer A).
- [ ] FE-4 Razorpay Checkout SDK integration — open with `orderId` from backend.
- [ ] FE-5 Confirmation page — "Booking confirmed, check your email."
- [ ] FE-6 Error states — payment failed, OTP failed, ticket sold out, coupon invalid.

### Razorpay / Email console
- [ ] EX-1 Add Razorpay test keys to backend env.
- [ ] EX-2 Configure Razorpay webhook URL (staging) with secret; use ngrok for local dev.
- [ ] EX-3 Verify SendGrid / SES sender domain for ticket emails.

## 6. Acceptance Criteria
*How the senior will verify the sprint is done.*

- [ ] Opening a shared event link shows ticket types without requiring login.
- [ ] Applying a valid coupon reduces the final amount; invalid coupon shows an error.
- [ ] OTP works on both email and phone channels.
- [ ] A first-time customer is auto-created with `role = CUSTOMER`; an existing email or phone reuses the row.
- [ ] Booking status is `PAID` **only** after Razorpay webhook verification — never from the client.
- [ ] Razorpay webhook rejects requests with an invalid signature (returns 400).
- [ ] Razorpay webhook is idempotent — retrying the same `paymentId` does not duplicate tickets or emails.
- [ ] Failed payments leave the booking in `PENDING` or `FAILED`.
- [ ] `PENDING` bookings older than 15 minutes are auto-expired by the cron job.
- [ ] QR code is generated and attached to the confirmation email.
- [ ] Confirmation page is shown after successful payment.

## 7. References
- Razorpay Checkout (Standard) docs
- Razorpay Webhooks + signature verification docs
- Firebase Auth setup (owned by Developer A, Sprint 1)
- Internal: PRD section 8.3 (Customer Guest Checkout), System design diagram v1.

## 8. Definition of Done
- [ ] All tasks above checked off.
- [ ] All acceptance criteria pass on staging.
- [ ] Code merged to `develop` behind a feature flag if needed.
- [ ] PR reviewed and approved by senior.
- [ ] API endpoints documented in Postman / Swagger.
- [ ] Short demo recorded or shown in sprint review.