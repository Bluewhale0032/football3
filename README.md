# ⚽ Northgate United FC — Team Management Platform (Demo)

A premium, interactive **frontend demo** of a football club management platform:
squad roster with player photos, training/match calendar, match centre,
training statistics, reminders/notifications and **role-based access control**
— including a secure, RBAC-gated medical records tab.

> **Demo notice:** all club names, people, statistics and medical records are
> **fictional demo data**. No real personal or medical information exists in
> this project. The demo password is intentionally public.

---

## Tech stack

| Layer | Technology |
|---|---|
| Framework | **Next.js 16** (App Router) + **React 19** + TypeScript 5 |
| Styling | **Tailwind CSS 4** + shadcn/ui (New York) + Lucide icons |
| Charts | **Recharts** (via shadcn ChartContainer) |
| Data layer | **TanStack Query** + **Zustand** |
| API | Next.js Route Handlers (Node runtime) with **Zod** validation |
| Auth | Demo login endpoint (simulated session) |
| Forms | react-hook-form + @hookform/resolvers + Zod |

> The original brief mentioned Vite + Express; this sandbox standardises on
> Next.js 16 (same React code, deploy-ready on Vercel, with a Node.js API
> layer built in). All business endpoints are real REST APIs with validation
> and RBAC — the dataset itself is an in-memory demo dataset.

## Quick start

```bash
bun install          # or npm install / pnpm install
bun run dev          # starts Next.js on http://localhost:3000
```

Open the app, pick any demo role card (or use the credentials below) and sign in.

### Demo accounts

| Role | Email | Password |
|---|---|---|
| Head Coach | `coach@northgate.fc` | `demo1234` |
| Administrator | `admin@northgate.fc` | `demo1234` |
| Club Doctor | `doctor@northgate.fc` | `demo1234` |
| Player | `player@northgate.fc` | `demo1234` |

You can also switch identity live via the **“Demo role”** menu in the top bar.

## Environment variables

No secrets are required to run the demo — there are **no API keys, no
database credentials, no secrets in source code**. For the production build
(described below) you would add:

```env
# .env.local  (NOT committed)
DATABASE_URL="postgresql://user:password@host:5432/football"
NEXTAUTH_URL="https://your-domain.com"
NEXTAUTH_SECRET="<openssl rand -base64 32>"
```

## Feature map

| Feature | Where | Notes |
|---|---|---|
| Login + role selection | `src/components/football/login-view.tsx` | Demo session via `POST /api/auth/login` |
| Dashboard (KPIs, form, countdown, charts) | `dashboard-view.tsx` | Live countdown to next fixture |
| Roster with photos, filters | `roster-view.tsx` | Search, position & status filters, add player (coach/admin) |
| Player profiles + **gated medical tab** | `player-detail-dialog.tsx` | Medical data served only to doctor/admin (HTTP 403 otherwise) with server-side audit log |
| Calendar (month grid, event detail, create) | `calendar-view.tsx`, `event-form-dialog.tsx` | Colour-coded by type, reminders, validated create form |
| Match centre (fixtures/results, timeline, formation) | `matches-view.tsx`, `formation-pitch.tsx` | Add fixture (coach/admin), SVG pitch with starting XI |
| Training statistics | `training-view.tsx` | Attendance trend, session load, drill completion, player radar |
| Notifications / reminders | `topbar.tsx` | Bell dropdown, unread badge, mark-as-read |
| Sticky footer, dark mode, responsive shell | `app-root.tsx`, `footer.tsx` | Mobile nav drawer, `min-h-screen flex` + `mt-auto` footer |

## API reference (demo)

| Method | Route | Access | Purpose |
|---|---|---|---|
| POST | `/api/auth/login` | public | Demo sign-in (email+password or role) |
| GET | `/api/dashboard` | any role | Aggregated KPIs, fixtures, events, scorers |
| GET | `/api/players` | any role | Squad list (no medical fields) |
| POST | `/api/players` | coach, admin | Add player (Zod-validated, unique shirt number) |
| GET | `/api/players/:id/medical` | **doctor, admin only** | Confidential record + audit log entry |
| GET | `/api/events` | any role | Calendar events |
| POST | `/api/events` | coach, admin | Create event (validated) |
| GET | `/api/matches` | any role | Fixtures + results |
| POST | `/api/matches` | coach, admin | Schedule fixture (validated) |
| GET | `/api/stats` | any role | Training analytics + per-player radar |
| GET/POST | `/api/notifications` | any role | List reminders / mark as read |

RBAC is enforced **server-side**: the client attaches an `x-demo-role` header
after login and privileged endpoints verify it. In this demo the header is
trusted because there is no real identity system and no real data —
production replaces it with signed sessions (see below).

## Production roadmap

The demo is structured so each boundary maps 1:1 to a production system:

1. **Database — PostgreSQL + Prisma**

   ```prisma
   model User      { id String @id @default(cuid()) email String @unique name String passwordHash String role Role teamId String? }
   model Player    { id String @id @default(cuid()) teamId String firstName String lastName String shirtNumber Int @unique position String dob DateTime nationality String photoUrl String? medicalRecord MedicalRecord? }
   model Event     { id String @id @default(cuid()) teamId String type String title String startsAt DateTime location String reminderMinutes Int? createdByUserId String }
   model Match     { id String @id @default(cuid()) teamId String opponent String startsAt DateTime venue String competition String status String homeScore Int? awayScore Int? }
   model MatchStat { id String @id @default(cuid()) matchId String @unique possession Int shots Int shotsOnTarget Int corners Int fouls Int }
   model Attendance{ id String @id @default(cuid()) sessionId String playerId String status String }
   model MedicalRecord { id String @id @default(cuid()) playerId String @unique lastCheckup DateTime fitnessStatus String conditions Json allergies Json medications Json notes String clearedByUserId String }
   enum Role { ADMIN COACH DOCTOR PLAYER }
   ```

2. **Authentication** — NextAuth credentials provider, bcrypt password
   hashing, httpOnly session cookies, middleware-protected routes. Replace
   `getDemoRole()` in `src/lib/football/rbac.server.ts` with session lookup.
3. **Medical data security** — column-level encryption (pgcrypto or
   application-level with KMS), access audit table, short-lived access tokens,
   IP + role checks, and the audit log already modelled in
   `medical-records.server.ts`.
4. **Notifications** — scheduled jobs (e.g. Vercel Cron) sending email/push
   reminders ahead of events; the reminder UI is already wired.
5. **File uploads** — player photos via Vercel Blob/S3 with signed URLs.

## Scripts

```bash
bun run dev        # development server (port 3000)
bun run lint       # eslint
bun run db:push    # push Prisma schema (production DB setup)
node scripts/generate-assets.mjs   # regenerate demo images (idempotent)
```

## Accessibility & UX

- Semantic landmarks (`header`, `main`, `nav`, `footer`), labelled controls, `aria-live` regions for dynamic counts.
- Keyboard-visible focus rings, 44px+ touch targets, mobile-first responsive layout.
- Loading skeletons, empty states, error states with retry, toast feedback (Sonner), inline form validation.
