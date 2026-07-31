# Running Prichha Locally — Full Setup Guide

This walks you through running all four pieces on your own machine:

1. **prichha-backend** — the API server (Node + Express + PostgreSQL + Socket.io)
2. **prichha-app** — the mobile app (Expo)
3. **prichha-web** — the Next.js web platform (registration, dashboards, calling, chat, admin)
4. **prichha-admin** — the standalone admin dashboard (optional — `prichha-web` has its own `/admin` now too)

Do them in this order — the backend has to be running before the others will work.

---

## 0. Install the prerequisites (one-time)

You need these installed on your machine before starting:

| Tool | Check if installed | Install if missing |
|---|---|---|
| **Node.js** (v18+) | `node -v` | https://nodejs.org (download the LTS version) |
| **npm** (comes with Node) | `npm -v` | — |
| **PostgreSQL** (v14+) | `psql --version` | https://www.postgresql.org/download/ (or see "easy option" below) |
| **Expo Go app** on your phone | — | Search "Expo Go" on the App Store / Play Store |

**Easy option for PostgreSQL:** if you don't want to install Postgres locally, create a free instance at https://neon.tech or https://supabase.com in about 2 minutes and copy the connection string it gives you — you'll paste that into `.env` in Step 1. Skip straight to Step 1 and use that URL instead of a local one.

Unzip all four project folders somewhere convenient, e.g.:
```
~/prichha/prichha-backend
~/prichha/prichha-app
~/prichha/prichha-web
~/prichha/prichha-admin
```

---

## 1. Backend Server

Open a terminal.

```bash
cd ~/prichha/prichha-backend
npm install
```

### 1a. Create the database

If you installed Postgres locally, create an empty database:
```bash
psql -U postgres -c "CREATE DATABASE prichha;"
```
(If it asks for a password, use whatever you set when installing Postgres.)

If you used Neon/Supabase, skip this — the database already exists.

### 1b. Configure environment variables

```bash
cp .env.example .env
```
Open `.env` in a text editor and fill in:

```bash
DATABASE_URL="postgresql://postgres:YOUR_PASSWORD@localhost:5432/prichha?schema=public"
# ^ if using Neon/Supabase, paste their connection string here instead

JWT_SECRET="any-long-random-string-you-make-up"

# Required if you're running prichha-web or prichha-admin (both use browser cookies):
CORS_ORIGIN="http://localhost:3000,http://localhost:5173"
```

Leave `RAZORPAY_*` and `AGORA_*` blank for now if you only want to test the mobile app — it works without them, you just won't be able to test payments/calling until you add real keys (see the bottom of this guide). **If you're setting up `prichha-web`, though, you'll need real Razorpay test-mode keys from the start** — the ₹11 registration fee is mandatory on that platform, so you can't even create a web account without them. Jump to "Adding real Razorpay / Agora keys later" at the bottom of this guide before continuing with Step 4 below.

### 1c. Create the database tables

```bash
npx prisma migrate dev --name init
```
You'll see it create a bunch of tables. That's expected.

### 1d. Seed test data

```bash
npm run prisma:seed
```
This fills the database with sample mentors, professionals, a demo student, and a demo admin. You'll see this in your terminal when it's done:
```
Seed complete. Demo logins:
  Student: student@example.com / password123
  Admin:   admin@example.com / password123
```

### 1e. Start the server

```bash
npm run dev
```
You should see:
```
Prichha API listening on http://localhost:4000
```

✅ **Leave this terminal window open and running.** Everything else in this guide needs it running in the background.

**Quick check:** open http://localhost:4000/health in a browser — you should see `{"ok":true}`.

---

## 2. Admin Web Dashboard

Open a **new** terminal window (keep the backend one running).

```bash
cd ~/prichha/prichha-admin
npm install
cp .env.example .env
```
The default `.env` already points at `http://localhost:4000`, which is correct since this runs in your browser on the same machine as the backend — no changes needed.

```bash
npm run dev
```
You'll see something like:
```
Local:   http://localhost:5173/
```
Open that URL in your browser. Log in with:
- Email: `admin@example.com`
- Password: `password123`

✅ You should now see the Dashboard with stats, Users, Transactions, etc.

---

## 3. Mobile App

Open a **third** terminal window (keep the other two running).

```bash
cd ~/prichha/prichha-app
npm install
```
> If `npm install` complains about `react-native-agora`, run `npx expo install react-native-agora` afterward to fix its version automatically.

### 3a. Connect the app to your backend — the important part

The mobile app can't use `localhost` to reach your backend, because on a phone or emulator, "localhost" means *the phone itself*, not your computer. You need your computer's **local network IP address**.

**Find your computer's IP address:**
- **Mac:** System Settings → Wi-Fi → click the "i" next to your network (or run `ipconfig getifaddr en0` in Terminal)
- **Windows:** open Command Prompt, run `ipconfig`, look for "IPv4 Address" (something like `192.168.1.23`)
- **Linux:** run `hostname -I` or `ip addr`

It'll look like `192.168.1.23` or `10.0.0.5`.

```bash
cp .env.example .env
```
Open `.env` and set:
```bash
EXPO_PUBLIC_API_URL=http://192.168.1.23:4000
```
(replace with **your** actual IP — not this example one)

> ⚠️ Your phone and your computer must be on the **same Wi-Fi network** for this to work.

### 3b. Start the app

```bash
npm run start
```
This opens Expo's developer tools and shows a QR code in the terminal.

- **On your phone:** open the Expo Go app and scan the QR code (iOS: use the Camera app instead, then tap the notification).
- **On a simulator:** press `i` (iOS simulator, Mac only) or `a` (Android emulator) in the terminal.

The app should load, showing the Prichha splash screen.

### 3c. Log in

Tap through onboarding, then log in with:
- Email: `student@example.com`
- Password: `password123`

You should land on the home screen with real mentors/professionals loaded from your backend.

> **Note on video/audio calling:** the calling feature (Agora) uses a native module that Expo Go cannot run. Everything else in the app (browsing, chat, wallet, booking) works fine in Expo Go. To test actual calls, you'd need to build a custom dev client (`npx expo prebuild` + `eas build`) — ask me if you want a walkthrough of that when you're ready.

---

## 4. Web Platform (prichha-web)

Open a **fourth** terminal window (keep the other three running).

```bash
cd ~/prichha/prichha-web
npm install
cp .env.local.example .env.local
```
The default `.env.local` already points at `http://localhost:4000`, which is correct.

```bash
npm run dev
```
Open **http://localhost:3000** in your browser.

### 4a. Try registering

Click "Sign up", fill in the form, pick a role, and submit. A Razorpay Checkout modal should pop up asking for **₹11**. In Razorpay's **test mode**, use their test card:
- Card number: `4111 1111 1111 1111`
- Any future expiry date, any CVV, any name

Once payment succeeds, you're automatically logged in and redirected to your dashboard.

> If clicking "Sign up" gives an error instead of opening the payment modal, double-check `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET` are set in the **backend's** `.env` (see the note at the end of Step 1b).

### 4b. Try the admin panel

Log out, then log in at `http://localhost:3000/login` with `admin@example.com` / `password123`. You'll land on `/admin` automatically. Check the **Approvals** tab — the seed script includes one mentor (`pending.mentor@example.com`) sitting there waiting for approval, so you can test that flow immediately.

---

## How the pieces connect (recap)

```
┌─────────────────┐         ┌──────────────────────┐
│   Mobile App     │ ──────► │                       │
│ (your phone,     │         │   Backend Server      │
│  via LAN IP)     │         │  localhost:4000        │
└─────────────────┘         │  (PostgreSQL + Socket.io)│
                              │                        │
┌─────────────────┐         │                        │
│  Web Platform     │ ─────► │                        │
│ (browser,          │         │                        │
│  localhost:3000)   │         │                        │
└─────────────────┘         │                        │
                              │                        │
┌─────────────────┐         │                        │
│  Admin Dashboard  │ ─────► │                        │
│ (browser,          │         └──────────────────────┘
│  localhost:5173)   │
└─────────────────┘
```
- The **backend** must always be running first — it's what all three other apps talk to.
- The **web platform** and **admin dashboard** both talk to it via `localhost:4000` (fine, since your browser is on the same machine) — and both need to be listed in the backend's `CORS_ORIGIN`, since they use cookie-based auth.
- The **mobile app** talks to it via your computer's **LAN IP** (`192.168.x.x:4000`), because it's a separate device — CORS doesn't apply to it (CORS is a browser-only mechanism).

---

## 5. Resetting / re-seeding the database

**To wipe all data and start fresh** (back in the `prichha-backend` terminal — stop the server first with `Ctrl+C`):

```bash
npx prisma migrate reset
```
This drops all tables, recreates them, and **automatically re-runs the seed script** too — so you'll get the same demo mentors/professionals/logins back immediately. Confirm the prompt with `y`.

Then restart the server:
```bash
npm run dev
```

**To just re-add seed data without wiping** (e.g. if you deleted some listings via the admin dashboard):
```bash
npm run prisma:seed
```
This is safe to run repeatedly — it won't create duplicates of the same demo accounts.

**To inspect the database visually** (handy for debugging):
```bash
npm run prisma:studio
```
Opens a browser UI at `http://localhost:5555` where you can view/edit every table directly.

---

## 6. Quick end-to-end test checklist

Once all four are running:

1. ✅ Backend: `http://localhost:4000/health` shows `{"ok":true}`
2. ✅ Admin dashboard (`:5173`): log in, see stats on the Dashboard page
3. ✅ Mobile app: log in as `student@example.com`, see mentors load on the home screen
4. ✅ Mobile app: open a mentor profile, tap "Message" → send a chat message
5. ✅ Mobile app: go to Wallet tab → see balance ₹2,450 and transaction history
6. ✅ Web platform (`:3000`): register a new account, complete the ₹11 test payment, land on your dashboard
7. ✅ Web platform: log in as `student@example.com`, browse mentors, book a chat session, send a message (this should also appear if you open the same session's chat via the mobile app — both read/write the same messages table)
8. ✅ Web platform `/admin`: log in as `admin@example.com`, go to Approvals, approve `pending.mentor@example.com`'s listing
9. ✅ Web platform: refresh the student dashboard's college-seniors list and confirm the newly-approved mentor now appears

If all of those work, your full stack is wired up correctly.

---

## Troubleshooting

| Problem | Likely fix |
|---|---|
| `npm install` fails on `prichha-backend`, `prichha-web`, or `prichha-admin` | Make sure Node.js is v18+: `node -v` |
| `prisma migrate dev` fails to connect | Double-check `DATABASE_URL` in `.env` — wrong password/port is the usual cause |
| Mobile app shows "Network request failed" | Your `EXPO_PUBLIC_API_URL` IP is wrong, or your phone isn't on the same Wi-Fi as your computer, or a firewall is blocking port 4000 |
| Web platform or admin dashboard shows a CORS error in the browser console | Make sure the backend is running and `CORS_ORIGIN` in the backend's `.env` explicitly lists `http://localhost:3000` and/or `http://localhost:5173` (a wildcard `*` will NOT work here, since both use cookie-based auth) |
| Web platform: "Sign up" doesn't open a payment popup | `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET` aren't set in the backend's `.env` — the registration flow can't create an order without them |
| Web platform: chat messages don't appear in real time | Check the browser console for a Socket.io connection error — usually the same CORS/cookie issue as above |
| Login fails with "Invalid email or password" | Make sure you ran `npm run prisma:seed` in the backend, and you're using `student@example.com` / `password123` exactly |
| Changed `.env` but nothing happened | Restart the relevant server (`Ctrl+C` then re-run the start command) — `.env` is only read on startup |

---

## Adding real Razorpay / Agora keys later

When you're ready to test real payments or calls (not required for the checklist above):
- **Razorpay:** get test-mode keys from https://dashboard.razorpay.com → Settings → API Keys, add to the backend's `.env` as `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET`. For webhook testing locally, use `ngrok http 4000` to get a public URL and register it in Razorpay's webhook settings.
- **Agora:** get an App ID + App Certificate from https://console.agora.io, add to `.env` as `AGORA_APP_ID` / `AGORA_APP_CERTIFICATE`. Remember calling itself needs a custom Expo dev client, not Expo Go.

Restart the backend (`Ctrl+C`, then `npm run dev`) after editing `.env` for either of these.
