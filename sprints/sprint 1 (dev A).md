

# Sprint 1 — Authentication & Authorization

| Field          | Value                                  |
| -------------- | -------------------------------------- |
| Sprint         | 1 of N                                 |
| Duration       | (14-may-2026 – 20-may-2026) (5WD)      |
| Owner          | Developer A                            |
| Reviewer       | MOHAMED SHAFEEQ                        |
| Status         | Ready for development                  |

---

## 1. Context

Why this sprint exists. One short paragraph.

> We are building a ticketing platform with three user types: customers, hosts, and admins. Before any feature work can start, we need a working auth system that identifies who is making each API call and what they are allowed to do. This sprint delivers that foundation.

---

## 2. Goals

What must be true when the sprint ends.

- A customer can sign in with email OTP or phone OTP.
- A host can sign in with email and password or Google.
- An admin can sign in only if their email is on the allow-list.
- Every protected API route rejects requests without a valid Firebase token.
- Every protected API route enforces the correct role.

---

## 3. Non-Goals

What we are **not** doing this sprint. This stops scope creep.

- Password reset flows.
- Multi-factor auth beyond Firebase's built-in OTP.
- `SUPER_ADMIN` role and sub-permissions (Phase 2).
- Social logins other than Google.

---

## 4. Technical Design

The senior has already made these decisions. The junior does not need to re-decide them.

### 4.1 Auth provider
Firebase Auth. Accepted vendor lock-in.

### 4.2 Sign-in methods

| Role     | Methods                                        |
| -------- | ---------------------------------------------- |
| Customer | Email OTP, Phone OTP, optional Google          |
| Host     | Email + password, Google, optional phone       |
| Admin    | Email + password only, allow-list enforced     |

### 4.3 Token flow
1. Client signs in through Firebase, receives an ID token.
2. Client sends `Authorization: Bearer <token>` on every API call.
3. API verifies the token with Firebase Admin SDK.
4. API looks up the user by `firebaseUid`. If missing, it auto-creates the row.
5. No separate backend JWT. The Firebase token is the session.

### 4.4 Auto user creation (guest checkout)
- Customer pays without signing up.
- If email or phone already maps to a user, the booking links to them.
- If not, a `User` row is created with `role = CUSTOMER` and `firebaseUid` is filled on first OTP verification.

### 4.5 Authorization
- NestJS guards: `@UseGuards(FirebaseAuthGuard, RolesGuard)` with `@Roles('HOST')`.
- Resource ownership is enforced in the service layer with explicit `where: { hostId: req.user.id }` clauses. Role check alone is not enough.

### 4.6 Data model

```prisma
model User {
  id          String   @id @default(uuid())
  firebaseUid String   @unique
  email       String?  @unique
  phone       String?  @unique
  name        String?
  roles       Role[]
  createdAt   DateTime @default(now())
}
```

---
### 4.7 Testing routes

Temporary protected testing APIs and Next.js routes will be created
to verify Firebase authentication, role guards, and frontend token flow.

#### Backend test routes
- `/api/test/customer`
- `/api/test/host`
- `/api/test/admin`

#### Frontend test pages
- `/test/customer`
- `/test/host`
- `/test/admin`

## 5. Tasks

Broken into small, testable pieces. Each task should take half a day or less.

### Backend
- [ ] **BE-1** Add Firebase Admin SDK and load credentials from env.
- [ ] **BE-2** Create `User` Prisma model and run migration.
- [ ] **BE-3** Implement `FirebaseAuthGuard`. Verifies token, attaches `req.user`.
- [ ] **BE-4** Implement `RolesGuard` and `@Roles()` decorator.
- [ ] **BE-5** Add auto-create-user logic on first authenticated request.
- [ ] **BE-6** Seed admin email allow-list.
- [ ] **BE-7** Unit tests for both guards (valid token, invalid token, wrong role).
- [ ] **BE-8** Create temporary protected test APIs for CUSTOMER, HOST, and ADMIN roles.

### Frontend
- [ ] **FE-1** Initialise Firebase SDK on web and mobile.
- [ ] **FE-2** Build login screens for each role.
- [ ] **FE-3** Attach ID token to every API call via Axios interceptor.
- [ ] **FE-4** Handle token refresh via Firebase SDK.
- [ ] **FE-5** Create Next.js protected test pages for role validation.

### Firebase console
- [ ] **FB-1** Enable email/password, email OTP, phone OTP, and Google providers.
- [ ] **FB-2** Add Admin SDK service account to backend env.

---

## 6. Acceptance Criteria

How the senior will verify the sprint is done.

- [ ] Calling any protected route without a token returns 401.
- [ ] Calling a `@Roles('HOST')` route as a customer returns 403.
- [ ] A new user signing in for the first time creates a `User` row automatically.
- [ ] A host can only fetch their own events, not another host's.
- [ ] An admin email not on the allow-list cannot get the `ADMIN` role.
- [ ] `/api/test/customer` is accessible only by CUSTOMER users.
- [ ] `/api/test/host` is accessible only by HOST users.
- [ ] `/api/test/admin` is accessible only by ADMIN users.
- [ ] Next.js protected test pages correctly validate Firebase auth state.
- [ ] Unauthorized users are redirected or blocked from protected pages.

---



## 7. References

- [Firebase Admin SDK docs](https://firebase.google.com/docs/admin/setup)
- [NestJS guards docs](https://docs.nestjs.com/guards)
- Internal: PRD section 4.2, System design diagram v1.

---

## 8. Definition of Done

- [ ] All tasks above checked off.
- [ ] All acceptance criteria pass.
- [ ] Code merged to `develop` behind a feature flag if needed.
- [ ] PR reviewed and approved by senior.
- [ ] Short demo recorded or shown in sprint review.



