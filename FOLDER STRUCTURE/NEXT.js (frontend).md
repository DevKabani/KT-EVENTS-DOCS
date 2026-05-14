# Next.js App Router — Folder Structure Standard

> Applies to all Next.js apps in the monorepo (`web-public`, `web-host`, `web-admin`).
> Goal: keep routing thin, business logic isolated by feature, shared code obvious.

## TL;DR

```
app/          → routing only
features/     → feature code (UI, hooks, services, types)
components/   → shared UI (layout + common)
lib/          → shared utilities & infrastructure
```

---

## Folder Responsibilities

| Folder        | Responsibility                                                       |
| ------------- | -------------------------------------------------------------------- |
| `app/`        | Routing, layouts, loading & error boundaries, route-level pages only |
| `features/`   | Feature-specific UI, hooks, API services, types, business logic      |
| `components/` | Shared reusable UI used by multiple features                         |
| `lib/`        | API client, auth helpers, constants, utilities                       |

---

## Reference Layout

```
apps/web-host/
├── app/                        # Routing only
│   ├── layout.tsx
│   ├── page.tsx
│   ├── dashboard/
│   │   └── page.tsx
│   ├── events/
│   │   ├── page.tsx
│   │   ├── create/page.tsx
│   │   └── [eventId]/
│   │       ├── page.tsx
│   │       └── edit/page.tsx
│   ├── bookings/page.tsx
│   ├── attendees/page.tsx
│   └── settings/page.tsx
│
├── features/                   # Feature code lives here
│   ├── dashboard/
│   ├── events/
│   │   ├── components/         # EventCard, EventForm, EventTable, EventFilters
│   │   ├── hooks/              # useEvents, useEventDetails
│   │   ├── services/           # eventApi.ts
│   │   ├── types/              # event.types.ts
│   │   └── utils/              # eventHelpers.ts
│   ├── bookings/               # Same shape as events/
│   ├── attendees/
│   └── settings/
│
├── components/                 # Shared UI only
│   ├── layout/                 # AppSidebar, Header, PageHeader
│   └── common/                 # ConfirmDialog, EmptyState, DataTable
│
├── lib/                        # Shared utilities
│   ├── apiClient.ts
│   ├── auth.ts
│   ├── constants.ts
│   └── utils.ts
│
├── middleware.ts
├── next.config.ts
└── tsconfig.json
```

Every feature folder follows the same shape:

```
features/<name>/
├── components/
├── hooks/
├── services/
├── types/
└── utils/
```

---

## Rules

### `app/` — routing only

Route files just wire URLs to feature pages. No logic, no data fetching, no UI here.

```tsx
// app/events/page.tsx
import { EventsPage } from "@/features/events/components/EventsPage";

export default function Page() {
  return <EventsPage />;
}
```

### `features/` — feature code

Anything used by only one feature belongs in that feature's folder.

```
features/events/
├── components/   EventsPage.tsx, EventForm.tsx, EventTable.tsx
├── hooks/        useEvents.ts
├── services/     eventApi.ts
├── types/        event.types.ts
└── utils/        eventHelpers.ts
```

### `components/` — shared UI only

| ✅ Good                              | ❌ Wrong                          |
| ------------------------------------ | --------------------------------- |
| `components/common/ConfirmDialog.tsx` | `components/EventForm.tsx`        |
| `components/common/EmptyState.tsx`    | `components/BookingCard.tsx`      |
| `components/common/DataTable.tsx`     | `components/AttendeeList.tsx`     |
| `components/layout/AppSidebar.tsx`    |                                   |

Feature-specific components → `features/<name>/components/`.

### `lib/` — shared utilities & infrastructure

Cross-cutting code with no UI:

- API client (Axios/fetch wrapper)
- Auth helpers, token/session
- Constants
- Date formatters, common utilities

---

## Decision Guide

| Used by...                | Put it in...                    |
| ------------------------- | ------------------------------- |
| One feature only          | `features/<name>/`              |
| Multiple features (UI)    | `components/common/` or `layout/` |
| Multiple features (logic) | `lib/`                          |
| Just routing              | `app/`                          |


