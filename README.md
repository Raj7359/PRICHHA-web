# Prichha — Full Stack Project

A mentorship marketplace connecting students with college seniors ("mentors") and working professionals for paid chat / audio / video guidance sessions — available as both a mobile app and a web platform.

This archive contains **four separate projects** that together make up the full product:

```
prichha-fullstack/
├── prichha-backend/     ← Node.js + Express + PostgreSQL API (the brain — everything talks to this)
├── prichha-app/         ← React Native (Expo) mobile app — the original UI, now wired to live data
├── prichha-web/         ← Next.js web platform — registration (₹11 fee), dashboards, chat, calling, admin
├── prichha-admin/       ← Standalone React + Vite web dashboard for managing the platform
└── LOCAL_SETUP_GUIDE.md ← Step-by-step guide to running everything locally
```

**Start with `LOCAL_SETUP_GUIDE.md`** — it walks through installing, configuring, and running all four projects in the right order, in plain terminal commands. Each sub-project also has its own `README.md` with more implementation detail specific to that piece (API endpoint reference, folder structure, etc.) — read those once you're past initial setup and want to make changes.

> **Note on the two admin experiences:** `prichha-admin` (Vite) and `prichha-web`'s `/admin` route both talk to the same admin API and do largely the same job. `prichha-admin` was built first as a lightweight standalone tool; `prichha-web`'s `/admin` came later as part of the unified web platform and additionally has the professional-profile approval queue. You can run either (or both) — they don't conflict. If you only want one going forward, `prichha-web`'s `/admin` is the more complete option.

---

## Architecture at a glance

```
                         ┌─────────────────────────┐
                         │   prichha-backend        │
                         │   Express + Prisma        │
                         │   PostgreSQL + Socket.io   │
                         │                            │
                         │   /api/auth      (JWT)     │
                         │   /api/auth/register (₹11) │
                         │   /api/mentors             │
                         │   /api/professionals       │
                         │   /api/sessions            │
                         │   /api/wallet              │
                         │   /api/payments (Razorpay) │
                         │   /api/calls    (Agora)    │
                         │   /api/profile  (self-serve)│
                         │   /api/admin               │
                         └───┬──────┬──────┬──────────┘
                             │      │      │
              HTTP (Bearer)  │      │      │  HTTP + WebSocket (cookie)
                             │      │      │
                ┌────────────▼──┐  │   ┌───▼──────────────┐
                │  prichha-app   │  │   │  prichha-web       │
                │  Expo mobile   │  │   │  Next.js platform   │
                │  (iOS/Android) │  │   │  (registration,      │
                └────────────────┘  │   │   dashboards, calls,  │
                                     │   │   chat, /admin)       │
                       HTTP (Bearer)│   └────────────────────────┘
                                     │
                              ┌──────▼───────────┐
                              │  prichha-admin      │
                              │  Vite dashboard      │
                              │  (standalone, optional)│
                              └───────────────────────┘
```

- **prichha-backend** is the single source of truth for all four other pieces. It owns the database, authentication (supporting both Bearer-token and httpOnly-cookie auth simultaneously), payment verification, call token issuance, and real-time chat (Socket.io, sharing the same HTTP server).
- **prichha-app** is your original mobile UI/UX, unchanged in layout and styling, reading and writing through the backend's REST API.
- **prichha-web** is a separate, full Next.js web platform: registration with a mandatory ₹11 Razorpay fee, role-based dashboards, Agora video/audio calling, Socket.io chat, and an admin panel.
- **prichha-admin** is the original lightweight standalone admin dashboard (still fully functional, kept for anyone who prefers a minimal separate tool).

---

## What's implemented

| Area | Mobile app | Web platform |
|---|---|---|
| **Auth** | Email/password, Bearer JWT, immediate activation (no fee) | Email/password, httpOnly cookie JWT, **mandatory ₹11 Razorpay registration fee** before activation |
| **Directory** | Browse mentors/professionals (search/sort) | Same, plus only shows **admin-approved** listings |
| **Booking** | Schedule a session, optional immediate join | On-demand booking, immediate join |
| **Chat** | REST polling (`/api/sessions/:id/messages`) | Real-time via Socket.io (same underlying `Message` table — a message sent from either client shows up on both) |
| **Calling** | Agora (native SDK, needs a custom dev client) | Agora (Web SDK, works in any modern browser) |
| **Wallet** | Add money (Razorpay), withdraw request, transaction history | Same |
| **Advisor/Professional** | — | Availability toggle, active bookings, earnings analytics, "pending approval" status |
| **Admin** | — | Full panel: stats, users, professional-profile approval queue, payment logs, withdrawal approvals |

There is no placeholder or stubbed logic anywhere in this codebase — every route does real work against the database. The three things that need **your own credentials** before they're fully live end-to-end are Razorpay (payments + the ₹11 fee), Agora (calling), and PostgreSQL (a real database connection). Every project runs and is testable without them, with those specific features returning a clear "not configured" error until you add keys — see each `.env.example`.

---

## A note on what changed vs. your original mobile app upload

- **UI/UX/layout/styling: unchanged.** Every mobile screen still uses your original components, colors, fonts, and navigation structure.
- **One file removed:** your original upload contained a `backend/` folder (an incomplete Node/Express/Prisma scaffold whose `package.json` pointed to a `src/server.js` that didn't exist anywhere in the zip — it could never run). Removed to avoid confusion with the complete, working `prichha-backend/`.
- **Dependency versions fixed** in `prichha-app/package.json` (`catalog:`/`workspace:*` monorepo-only references replaced with real, installable versions).
- **Backend schema note:** `WalletTransaction.sessionId` is no longer globally unique — a completed call now records *two* transactions (the student's debit and the expert's credit), which is what makes advisor/professional earnings analytics on the web platform possible. If you'd already run a migration against the old schema, you'll need `npx prisma migrate dev` again to pick this up (see `LOCAL_SETUP_GUIDE.md`'s reset section).

---

## Environment variables — quick reference

| Project | File | Required to run at all | Required for full functionality |
|---|---|---|---|
| `prichha-backend` | `.env` | `DATABASE_URL`, `JWT_SECRET` | + `RAZORPAY_KEY_ID/SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `AGORA_APP_ID/CERTIFICATE`, `CORS_ORIGIN` (must list web app + admin origins explicitly — not `*` — since both use cookies) |
| `prichha-app` | `.env` | `EXPO_PUBLIC_API_URL` | — |
| `prichha-web` | `.env.local` | `NEXT_PUBLIC_API_URL` | — |
| `prichha-admin` | `.env` | `VITE_API_URL` | — |

Each project's `.env.example` has inline comments explaining every value and where to get it.

---

## Folder structure reference

### `prichha-backend/`
```
src/
├── index.ts                # Express app entry point — mounts all routes + Socket.io
├── lib/
│   ├── prisma.ts             # Prisma client singleton
│   ├── razorpay.ts           # Razorpay SDK client
│   ├── agora.ts               # Agora RTC token generation
│   └── socket.ts               # Socket.io server (real-time chat)
├── middleware/
│   ├── auth.ts                 # JWT verification — accepts Bearer header OR cookie
│   └── admin.ts                # Role check (requireAdmin)
├── routes/
│   ├── auth.routes.ts           # signup / login / me / logout (mobile + shared)
│   ├── registration.routes.ts    # ₹11 web registration flow (start/verify/webhook)
│   ├── mentors.routes.ts          # public mentor listing + detail (approved only)
│   ├── professionals.routes.ts
│   ├── sessions.routes.ts          # booking
│   ├── messages.routes.ts           # chat REST endpoints (mobile)
│   ├── wallet.routes.ts              # balance, transaction history, withdrawal requests
│   ├── payments.routes.ts             # Razorpay wallet top-up: order creation, verify, webhook
│   ├── calls.routes.ts                 # Agora token issuance, call-end billing
│   ├── profile.routes.ts                # advisor/professional self-service (availability, earnings)
│   └── admin.routes.ts                   # all /api/admin/* endpoints (incl. approval queue)
└── utils/jwt.ts                            # sign/verify helpers
prisma/
├── schema.prisma      # full data model
└── seed.ts             # demo data (mentors, professionals, student, admin, one pending-approval mentor)
```

### `prichha-web/`
See `prichha-web/README.md` for the full breakdown — `app/` (pages), `components/` (Navbar, ExpertCard, BookingModal, ChatPanel, CallRoom), `lib/` (API clients, auth context, socket client).

### `prichha-app/`
Your original Expo Router structure, unchanged, plus a `lib/`/`hooks/` data layer and `app/call/[sessionId].tsx` for calling. See `prichha-app`'s own history in earlier phases for details.

### `prichha-admin/`
Standalone Vite admin dashboard — see `prichha-admin/README.md`.

---

## Getting started

👉 Open **`LOCAL_SETUP_GUIDE.md`** and follow it top to bottom — it covers installing prerequisites, database setup, seeding demo data, and running all four projects, with a checklist at the end to confirm everything's wired together correctly.
