# Root Layout & Project Structure

This document outlines the high-level architecture and folder structure for the **KT-Events** monorepo. The project is managed using [pnpm workspaces](https://pnpm.io/workspaces) with root-level scripts for development, builds, linting, and testing.

---

## Directory Tree

```text
kt-events/
├── apps/                   # Individual applications (Next.js, NestJS)
├── packages/               # Shared libraries and configurations
├── docs/                   # Documentation and ADRs
├── openapi/                # API Specifications
└── root files              # Infrastructure and workspace config
```

---

## Applications (`/apps`)

| App | Technology | Domain | Description |
| :--- | :--- | :--- | :--- |
| `web-public` | Next.js | `yourdomain.com` | Marketing site, event discovery, and guest booking. |
| `web-host` | Next.js | `app.yourdomain.com` | Dashboard for event organizers (hosts) to manage events. |
| `web-admin` | Next.js | `admin.yourdomain.com` | Internal platform management and moderation. |
| `api` | Express | `api.yourdomain.com` | Core backend services and business logic. |
| `worker` | BullMQ | Background | Asynchronous job processing (emails, PDFs, etc). |

### Detailed App Breakdowns

<details>
<summary><b>web-public</b> (Marketing & Booking)</summary>

- `(marketing)/`: Home, pricing, and informational pages.
- `e/[slug]/`: Dynamic public event booking pages.
- `checkout/`: Guest checkout and payment flow.
- `scanner/`: Browser-based QR code scanner for check-ins.
</details>

<details>
<summary><b>web-host</b> (Organizer Dashboard)</summary>

- `dashboard/`: Overview of sales and active events.
- `events/`: Event creation and configuration.
- `bookings/`: Manage ticket sales and reservations.
- `attendees/`: Participant lists and data exports.
- `payouts/`: Financial management and withdrawals.
- `kyc/`: Identity verification for hosts.
</details>

<details>
<summary><b>web-admin</b> (Internal Management)</summary>

- `hosts/`: Review and manage organizer accounts.
- `kyc-queue/`: Verify host identities.
- `plans/`: Manage subscription tiers and platform fees.
- `audit-log/`: Track administrative actions (Phase 2).
</details>

<details>
<summary><b>api</b> (Express Backend)</summary>

- `features/`: Feature-sliced business domains:
  - `auth`, `users`, `hosts`, `events`, `bookings`
  - `payments`, `tickets`, `checkin`, `coupons`
  - `notifications`, `kyc`, `withdrawals`, `plans`
  - `admin`, `platform-settings`, `webhooks`
- `providers/`: Abstractions for external services (Payments, SMS, Email, Storage).
- `common/`: Shared middlewares, validations, and error handlers.
</details>

<details>
<summary><b>worker</b> (Background Jobs)</summary>

- `jobs/`: BullMQ processor definitions:
  - `send-email.job.ts`
  - `send-sms.job.ts`
  - `generate-qr.job.ts`
  - `generate-invoice-pdf.job.ts`
  - `generate-badge.job.ts`
</details>

---

## Shared Packages (`/packages`)

These packages are used across multiple applications to ensure consistency.

- **`db`**: Prisma schema, client, and migration scripts.
- **`shared-types`**: Common TypeScript interfaces, DTOs, and Enums.
- **`ui`**: Design system and shared React components (based on **shadcn/ui**).
- **`config`**: Shared configurations for ESLint, TypeScript, Tailwind, and Prettier.

---

## Other Directories

- **`openapi/`**: Contains `openapi.yaml`. This is the source of truth for API contracts, regenerated on every PR.
- **`docs/`**:
  - `architecture/`: High-level system diagrams.
  - `adrs/`: Architecture Decision Records tracking major technical choices.
  - `design/`: Figma exports and UI/UX documentation.
- **`docker-compose.yml`**: Spins up local Postgres, Redis, and MinIO instances.

---

## Root Configuration Files

- `package.json`: Root workspace scripts for running app and package tasks.
- `pnpm-workspace.yaml`: Defines the monorepo workspace members.
- `PROJECT.md`: Main project overview (this file).
- `README.md`: Getting started guide for developers.
- `CONTRIBUTING.md`: Guidelines for contributing code.
