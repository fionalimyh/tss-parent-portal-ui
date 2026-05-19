# TSS Parent Portal — Implementation Handover

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the TSS Parent Portal — a mobile-first web app where enrolled parents view their child's swim journey, payments, attendance, progress, and receive operational broadcasts.

**Architecture:** Next.js 14 App Router (TypeScript + Tailwind CSS + shadcn/ui) connected directly to the shared Supabase database. Parents authenticate via magic link email (Supabase Auth). Row Level Security enforces family-scoped data access. Supabase Realtime delivers broadcasts live.

**Tech Stack:** Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Supabase JS v2, Supabase Auth, Supabase Realtime, Supabase Storage, Vercel (deployment)

**Source of truth for data models:** `parent-portal-readme.md` Section 10 and `2026-05-11-crm-schema-design.md`. Do NOT modify `2026-05-11-crm-schema-design.md`.

---

## Project Timeline Overview

| Week | Days | Focus | Key Deliverable |
|---|---|---|---|
| **Week 0** | 1–3 | Pre-build decisions & environment setup | All blockers resolved, environments provisioned |
| **Week 1** | 4–8 | Foundation — schema + auth + shell | Magic link login works, schema live, seed data in |
| **Week 2** | 9–13 | Core modules — dashboard + all learner pages + payments | Every parent-facing page working with real data |
| **Week 3** | 14–18 | Communications + admin tools + staging | Inbox, Jamie, Support, ops admin forms, staging deployed, ops trained |
| **Week 4** | 19–23 | UAT + bug fix + production launch | Real parents tested, production live |

**Total: 23 working days (~5 weeks)**
**Critical path:** Week 0 decisions → Schema freeze (Day 6) → Seed data (Day 7) → Auth working (Day 8) → Dashboard (Day 9) → All modules (Week 2) → Staging + Ops training (Day 18) → UAT (Days 19–20) → Production (Day 22)

### Status Snapshot (Updated 2026-05-19)

- Week 0 environment setup is partially complete.
- Root `.env` has been populated with real values for `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `WATI_API_ENDPOINT`, `WATI_API_KEY`, and `ADMIN_EMAILS`.
- GitHub repository is public at `https://github.com/fionalimyh/tss-parent-portal-ui`.
- The Next.js application has not been scaffolded yet, so Week 1 build tasks have not started.
- The main remaining blockers are unresolved Week 0 product/ops decisions, SMTP setup, WATI delivery verification, Vercel setup, and the actual app scaffold.

---

## File Structure

```
src/
  app/
    (auth)/
      login/page.tsx                  ← magic link email form
      auth/callback/route.ts          ← Supabase Auth redirect handler
    (portal)/
      layout.tsx                      ← authenticated shell + bottom nav
      page.tsx                        ← redirect → /dashboard
      dashboard/page.tsx              ← family overview (learner cards)
      learners/[learnerId]/
        page.tsx                      ← learner profile
        schedule/page.tsx             ← upcoming + past classes
        attendance/page.tsx           ← attendance history
        progress/page.tsx             ← level + skills + coach notes
        credits/page.tsx              ← replacement credits
        tests/page.tsx                ← test status
      payments/page.tsx               ← invoices + PayNow QR + proof upload
      receipts/page.tsx               ← payment history
      inbox/page.tsx                  ← broadcasts list
      inbox/[broadcastId]/page.tsx    ← broadcast detail
      jamie/page.tsx                  ← FAQ chat assistant
      support/page.tsx                ← escalation form
      admin/
        broadcasts/page.tsx           ← ops: create new broadcast
        payments/page.tsx             ← ops: verify submitted payments
    api/
      support/escalate/route.ts       ← POST → Slack webhook
      payments/upload-proof/route.ts  ← POST → Supabase Storage + update payment
      payments/paynow-qr/route.ts     ← GET → signed QR URL

  components/
    layout/
      BottomNav.tsx
      Header.tsx
      PageWrapper.tsx
    dashboard/
      FamilyDashboard.tsx
      LearnerCard.tsx
      InboxBadge.tsx
    schedule/
      ScheduleList.tsx
      ClassCard.tsx
    attendance/
      AttendanceList.tsx
      AttendanceStatusBadge.tsx
    progress/
      ProgressView.tsx
      LevelBadge.tsx
      LevelProgressBar.tsx
    credits/
      CreditsList.tsx
      CreditCard.tsx
    payments/
      InvoiceCard.tsx
      PayNowQR.tsx
      ProofUpload.tsx
    receipts/
      ReceiptsList.tsx
    tests/
      TestStatusCard.tsx
    inbox/
      BroadcastList.tsx
      BroadcastCard.tsx
      BroadcastTypeIcon.tsx
    jamie/
      JamieChat.tsx
      JamieMessage.tsx
    support/
      EscalationForm.tsx
    admin/
      BroadcastForm.tsx
      PaymentVerificationList.tsx
    ui/                               ← shadcn/ui components

  lib/
    supabase/
      client.ts
      server.ts
      middleware.ts
    hooks/
      useFamily.ts
      useBroadcasts.ts
      useUnreadCount.ts
    utils/
      dates.ts
      levels.ts
      credits.ts

  types/
    database.types.ts                 ← generated from Supabase
    portal.types.ts                   ← app-level types
```

---

## WEEK 0 — Pre-Build Gate (Days 1–3)

> Nothing gets built until every item in this gate is resolved. These are blockers.

**Owner: PM + Ops + Management. Developer provisions environments only.**

### Decisions Required (Day 1–2)

- [ ] **Credit expiry policy confirmed** — when do replacement credits expire?
  - Options: end of current term / 30 days from issue date / never expire / custom
  - Decision recorded in `parent-portal-readme.md` Section 2.4
  - *Blocks: Phase 8 (Replacement Credits logic)*

- [ ] **Receipt generation method confirmed** — how are receipts produced?
  - Option A: Auto-generated from `invoice_items` (no PDF, just a portal view)
  - Option B: PDF uploaded manually by ops team and stored in Supabase Storage
  - *Blocks: Phase 14 (Receipts)*

- [ ] **PayNow QR method confirmed** — how is the QR code provided?
  - Option A: Static image uploaded per invoice by ops team (stored in `payments.paynow_qr_url`)
  - Option B: Dynamically generated by Edge Function using UEN + amount
  - *Blocks: Phase 9 (Payments) — significantly changes scope if Option B*

- [ ] **Admin broadcast creation method confirmed**
  - Option A: Simple `/admin/broadcasts` form in the portal (recommended — 3h dev work)
  - Option B: Supabase Studio direct insert using a saved query template (no dev work, higher ops error risk)
  - *Blocks: Phase 11 (Broadcasts admin) and ops go-live readiness*

- [ ] **Admin payment verification method confirmed**
  - Option A: Simple `/admin/payments` page in the portal (recommended — 3h dev work)
  - Option B: Ops runs SQL UPDATE directly in Supabase Studio
  - *Blocks: Phase 9 (Payments admin) and ops go-live readiness*

- [ ] **`portal_accounts` creation SOP confirmed** — how does a new enrolled family get portal access?
  - Option A: Ops manually inserts a row in Supabase Studio after each enrolment
  - Option B: CRM team adds a step to the enrolment workflow to trigger the insert
  - Option C: An `/admin/families` onboarding form (additional dev scope)
  - *Blocks: ops workflow post-launch*

- [ ] **PDPA consent copy approved by management/legal**
  - Draft the privacy notice text that appears on the login screen
  - Must reference: what data is collected, how it is used, contact for requests
  - *Blocks: Phase 2 (Auth) going live — portal cannot collect emails without this*

- [ ] **Test booking policy confirmed** — is test booking parent-initiated or admin-assigned only?
  - *Blocks: Phase 10 (Test Status) scope*

### Environment Setup (Day 1–3, Dev)

- [ ] **Supabase dev project provisioned**
  - Create new Supabase project named `tss-portal-dev`
  - Note Project URL, Anon Key, Service Role Key
  - Enable Email auth provider in Supabase Auth settings
  - Confirm project region: `ap-southeast-1` (Singapore)

- [ ] **Supabase production project provisioned**
  - Create new Supabase project named `tss-portal-prod`
  - Note Project URL, Anon Key, Service Role Key
  - Store all four keys securely (password manager or ops doc)
  - *Do not run any migrations on prod yet — prod migrations happen in Week 4*

- [x] **WATI API credentials obtained**
  - Log in to WATI dashboard → API settings → generate or copy API key
  - Note the WATI API endpoint URL (format: `https://live-mt-server.wati.io/{account_id}`)
  - Confirm that contact number **6582221357** is the active WATI business number
  - Confirm coach portal WATI inbox is live and coaches can receive + reply to messages
  - *Blocks: Phase 10 (Support Escalation)*

- [ ] **UAT parents identified and scheduled**
  - Identify 3–5 enrolled parents willing to test
  - Confirm availability during Week 4 (Days 19–20)
  - Collect their email addresses for staging `portal_accounts`
  - Note their device types (iPhone/Android, browser preference)

- [ ] **Seed data plan confirmed with ops**
  - 1 test family (family_name: "Test Family")
  - 2 learners (1 with Silver package on Stage 2, 1 with Platinum on Kinder)
  - 1 active enrollment per learner
  - 5 upcoming class sessions (next 4 weeks)
  - 3 past class sessions (2 Present, 1 Absent with credit generated)
  - 1 unpaid invoice (SGD 280)
  - 1 pending payment with a test PayNow QR URL
  - 1 completed payment (for Receipts page)
  - 2 replacement credits (1 Available, 1 Used)
  - Ops provides real-looking data; dev inserts it in Day 7

### External Services Setup (Complete Before Week 1)

All services below must be provisioned and credentials stored in `.env` before Day 4 development starts. See README Section 15 for full setup steps.

#### Supabase
- [ ] Create dev project `tss-portal-dev` at supabase.com
- [ ] Create prod project `tss-portal-prod` at supabase.com
- [ ] Store Project URL, Anon Key, Service Role Key for both projects in `.env` and password manager
- [ ] In both projects: create private Storage bucket named `payment-proofs`
- [ ] In both projects: Authentication → Settings → set Site URL, enable Email provider
- [ ] Configure custom SMTP with Resend (see below) — default Supabase email is rate-limited to 4/hour

#### Resend (Magic Link Email)
- [ ] Create account at resend.com (free tier)
- [ ] Add and verify domain `theswimstarter.com` — add DNS TXT + MX records at registrar
- [ ] Create API key → copy it
- [ ] In Supabase Dashboard (both projects) → Auth → SMTP → enable custom SMTP:
  - Host: `smtp.resend.com`, Port: `465`, Username: `resend`, Password: *(Resend API key)*
  - Sender: `noreply@theswimstarter.com`
- [ ] Test: trigger a magic link from Supabase Auth UI → confirm email delivered within 30 seconds

#### WATI (WhatsApp — Support Escalations)
- [x] WATI Dashboard → Settings → API → copy API Key and API Base URL
- [x] Add `WATI_API_ENDPOINT` and `WATI_API_KEY` to `.env`
- [ ] Run test curl: send a test message to a dev phone number → confirm it appears in WATI coach inbox
- [ ] Confirm WATI business number 6582221357 is active and receiving messages

#### Vercel (Hosting)
- [ ] Create account at vercel.com
- [ ] Import GitHub repo → create staging project `tss-portal-staging` with dev Supabase env vars
- [ ] Create production project `tss-portal-prod` — add prod Supabase env vars + prod WATI key (configure prod env vars in Week 4)
- [ ] Configure custom domain `portal.theswimstarter.com` in production project → add DNS CNAME at registrar

**Estimated cost at launch: $0/month** (all services on free tier). See README Section 15.5 for upgrade triggers.

---

### Week 0 Deliverable Checklist

- [ ] All 8 decision items above have a confirmed answer documented in writing
- [ ] Dev Supabase project live and accessible (URL + keys in `.env`)
- [ ] Resend domain verified and custom SMTP working end-to-end on dev project
- [ ] Prod Supabase project live and keys stored securely in password manager
- [x] WATI API key obtained and WATI endpoint URL confirmed
- [ ] WATI test: sent a test message via API → message appeared in coach portal WATI inbox
- [ ] Vercel staging project created and linked to repo
- [ ] 3–5 UAT parents confirmed with their email addresses collected
- [ ] Seed data plan written and signed off by ops
- [ ] PDPA consent copy drafted and approved

---

## WEEK 1 — Foundation (Days 4–8)

> Two parallel tracks. Dev builds the project shell. Ops/Dev sets up the database.

### Track A — Project Setup & Authentication (Dev)

#### Day 4 AM — Scaffold Project (3h)

- [ ] Run `npx create-next-app@latest tss-parent-portal --typescript --tailwind --app --src-dir --import-alias "@/*"`
- [ ] Install Supabase dependencies: `npm install @supabase/supabase-js @supabase/ssr`
- [ ] Install date libraries: `npm install date-fns date-fns-tz`
- [ ] Install lucide-react icons: `npm install lucide-react`
- [ ] Initialise shadcn/ui: `npx shadcn@latest init` (Default style, Neutral base, CSS variables: yes)
- [ ] Add shadcn components: `npx shadcn@latest add button card badge avatar sheet dialog input textarea label toast`
- [ ] Copy root `.env` into the Next.js project root — fill in real dev values for `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `WATI_API_ENDPOINT`, `WATI_API_KEY`
- [ ] Add `.env` as the very first line of `.gitignore` — verify it is not tracked by git
- [ ] Verify: `git status` does not list `.env`
- [ ] Confirm `npm run dev` starts without errors on `http://localhost:3000`
- [ ] Commit: `git commit -m "feat: scaffold Next.js project with Supabase and shadcn/ui"`

#### Day 4 PM — Supabase Client Setup (2h)

- [ ] Create `src/lib/supabase/client.ts` (browser client using anon key)
- [ ] Create `src/lib/supabase/server.ts` (server client with cookie handling for SSR)
- [ ] Create `src/middleware.ts` (session refresh on every request, redirect unauthenticated users to `/login`, redirect authenticated users away from `/login`)
- [ ] Verify middleware config matcher excludes `_next/static`, `_next/image`, `favicon.ico`
- [ ] Test middleware: visit `http://localhost:3000/dashboard` unauthenticated — confirm redirect to `/login`
- [ ] Commit: `git commit -m "feat: add Supabase browser/server clients and auth middleware"`

#### Day 5 AM — Authentication Pages (3h)

- [ ] Create `src/app/(auth)/login/page.tsx`:
  - Email input field with label
  - Submit button: "Send login link"
  - Loading state on submit: "Sending…"
  - Success state: "Check your email" screen with the entered email displayed
  - Error state: shows Supabase error message
  - PDPA consent notice (approved text from Week 0) visible below the form
- [ ] Create `src/app/(auth)/auth/callback/route.ts`:
  - Exchanges code for session using `supabase.auth.exchangeCodeForSession(code)`
  - On success: redirect to `/dashboard`
  - On failure: redirect to `/login?error=auth_failed`
- [ ] In Supabase dashboard → Authentication → URL Configuration:
  - Add `http://localhost:3000/auth/callback` to Redirect URLs
- [ ] Test auth flow end-to-end:
  - [ ] Enter a real email on login page → confirm email is received
  - [ ] Tap magic link in email → confirm redirect to `/dashboard`
  - [ ] Reload `/dashboard` → confirm session persists (no redirect to login)
  - [ ] Open in incognito → confirm `/dashboard` redirects to `/login`
  - [ ] Enter invalid email format → confirm HTML5 validation fires
- [ ] Commit: `git commit -m "feat: add magic link authentication with Supabase Auth"`

#### Day 5 PM — Layout Shell (3h)

- [ ] Create `src/app/(portal)/layout.tsx`:
  - Reads Supabase session server-side
  - Redirects to `/login` if no session
  - Renders `{children}` above `<BottomNav />`
  - Adds bottom padding (`pb-20`) so content clears the nav bar
- [ ] Create `src/components/layout/BottomNav.tsx`:
  - 5 tabs: Home (`/dashboard`), Payments (`/payments`), Inbox (`/inbox`), Support (`/support`), More (Sheet/drawer with links to Jamie, Receipts, Tests)
  - Active tab highlighted using `usePathname()`
  - Inbox tab shows `<InboxBadge />` (placeholder — real count added in Phase 11)
  - Fixed to bottom, `z-50`, background colour, safe area padding (`pb-safe`)
- [ ] Create `src/components/layout/Header.tsx`:
  - Props: `title: string`, optional `backHref?: string`
  - Back arrow appears only when `backHref` is provided
  - Sticky top, `z-40`, border-bottom
- [ ] Add `pb-safe` utility to `src/app/globals.css`:
  ```css
  .pb-safe { padding-bottom: env(safe-area-inset-bottom); }
  ```
- [ ] Create `src/app/(portal)/page.tsx` that redirects to `/dashboard`
- [ ] Create `src/components/ui/LoadingSpinner.tsx` (centred spinning ring)
- [ ] Create `src/components/ui/EmptyState.tsx` (icon + message + optional sub-message)
- [ ] Test layout manually:
  - [ ] Bottom nav renders on every portal page
  - [ ] Active tab is highlighted correctly for each route
  - [ ] Header back button appears and navigates correctly
  - [ ] Layout does not overlap content on iPhone Safari (safe area respected)
- [ ] Commit: `git commit -m "feat: add portal layout shell with bottom navigation and header"`

---

### Track B — Supabase Schema & Data Setup (Dev + Ops)

#### Day 4 — Portal Table Migrations (5h)

- [ ] Open Supabase SQL Editor in dev project
- [ ] Run migration: create `portal_accounts` table (see README 10.1)
- [ ] Run migration: create `class_sessions` table (see README 10.9)
- [ ] Run migration: create `attendance_records` table (see README 10.10)
- [ ] Run migration: create `replacement_credits` table (see README 10.11)
- [ ] Run migration: create `broadcasts` table (see README 10.15)
- [ ] Run migration: create `broadcast_reads` table (see README 10.16)
- [ ] Run migration: create `support_tickets` table (see README 10.17)
- [ ] Run migration: `ALTER TABLE payments ADD COLUMN IF NOT EXISTS payment_status_extended text, proof_url text, proof_uploaded_at timestamptz, verified_by text, verified_at timestamptz, paynow_qr_url text`
- [ ] Verify in Table Editor: all 7 new tables visible
- [ ] Verify in Table Editor: `payments` table has new columns
- [ ] Save all SQL as migration files in `supabase/migrations/` folder for reproducibility

#### Day 5 — RLS Policies (2h)

- [ ] Deploy `get_family_id()` helper function via SQL Editor
- [ ] Enable RLS on all 7 portal-owned tables
- [ ] Apply all 17 RLS policies (see README / original handover Phase 0.4)
- [ ] Verify RLS in Supabase dashboard → Authentication → Policies:
  - [ ] `portal_accounts` — SELECT policy exists
  - [ ] `families` — SELECT policy exists
  - [ ] `contacts` — SELECT policy exists
  - [ ] `learners` — SELECT policy exists
  - [ ] `enrollments` — SELECT policy exists
  - [ ] `class_sessions` — SELECT policy exists
  - [ ] `attendance_records` — SELECT policy exists
  - [ ] `replacement_credits` — SELECT policy exists
  - [ ] `invoices` — SELECT policy exists
  - [ ] `invoice_items` — SELECT policy exists
  - [ ] `payments` — SELECT + UPDATE policies exist
  - [ ] `broadcasts` — SELECT policy exists
  - [ ] `broadcast_reads` — ALL policy exists
  - [ ] `support_tickets` — ALL policy exists
- [ ] RLS test (using Supabase SQL Editor with `SET LOCAL role = 'authenticated'`):
  - [ ] Query `families` as authenticated user → returns only own family
  - [ ] Query `families` as different user → returns no rows
  - [ ] Query `broadcasts` as authenticated user → returns all active broadcasts
- [ ] Commit migration files: `git commit -m "chore: add Supabase RLS policies"`

#### Day 5 — Storage Bucket (1h)

- [ ] Create `payment-proofs` bucket in Supabase Storage:
  - Public: NO
  - File size limit: 10MB
  - Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`, `application/pdf`
- [ ] Apply storage RLS policies (upload to own family path, read own family path)
- [ ] Test storage policy:
  - [ ] Attempt upload as authenticated user to `{family_id}/test.jpg` → succeeds
  - [ ] Attempt upload to a different `family_id` path → fails with 403

#### Day 6 — TypeScript Type Generation & Schema Freeze

- [ ] Install Supabase CLI: `npm install -g supabase`
- [ ] Generate types: `supabase gen types typescript --project-id <dev-project-id> --schema public > src/types/database.types.ts`
- [ ] Verify `database.types.ts` contains:
  - [ ] `portal_accounts` table type
  - [ ] `class_sessions` table type
  - [ ] `attendance_records` table type
  - [ ] `replacement_credits` table type
  - [ ] `broadcasts` table type
  - [ ] `broadcast_reads` table type
  - [ ] `support_tickets` table type
  - [ ] `payments` table with new extended columns
- [ ] Create `src/types/portal.types.ts` for app-level derived types (LearnerWithEnrollment, InvoiceWithPayment, etc.)
- [ ] Commit: `git commit -m "chore: generate Supabase TypeScript types"`
- [ ] **SCHEMA FREEZE** — announce to team: no schema changes after this point without going through change approval. Any change requires: (1) approval, (2) new migration file, (3) regenerate types, (4) update all affected queries.

#### Day 7 — Seed Data (1.5h, Ops + Dev)

- [ ] Insert test family: `INSERT INTO families (family_name, country) VALUES ('Chan Family', 'SG')`
- [ ] Insert 2 test contacts (primary + secondary guardian)
- [ ] Insert 2 learners:
  - Learner 1: "Emma Chan", level "Stage 2", enrollment_status "Successful Enrollment"
  - Learner 2: "Ethan Chan", level "Kinder", enrollment_status "Successful Enrollment"
- [ ] Insert pools (if not already in CRM): verify Bishan, Sengkang etc. exist
- [ ] Insert 2 enrollments (one per learner):
  - Emma: Silver package, Saturday 9:00am, Bishan pool, Coach Don
  - Ethan: Platinum package, Sunday 10:00am, Sengkang pool, Coach Ben
- [ ] Insert 5 upcoming class sessions (next 4 Saturdays + Sundays)
- [ ] Insert 3 past class sessions:
  - Session 1: 2 weeks ago, Emma attended (Present)
  - Session 2: 1 week ago, Emma absent (Absent, credit generated)
  - Session 3: 1 week ago, Ethan attended (Present)
- [ ] Insert 2 attendance records for past sessions
- [ ] Insert 1 replacement credit (Available, linked to Emma's absent session)
- [ ] Insert 1 invoice: SGD 280, due next week, Unpaid
- [ ] Insert 1 pending payment (linked to invoice, `payment_status_extended = 'Pending'`, `paynow_qr_url` = placeholder image URL)
- [ ] Insert 1 completed payment (for Receipts page testing, `payment_status_extended = 'Completed'`)
- [ ] Create 2 `portal_accounts` rows:
  - Row 1: dev developer's real email → linked to Chan Family contact
  - Row 2: ops tester's real email → linked to Chan Family (secondary contact)
- [ ] Test login: developer sends magic link to own email → lands on dashboard → sees "Chan Family"

#### Week 1 Deliverable Checklist

- [ ] `npm run dev` starts without warnings or errors
- [ ] Magic link login works end-to-end (email → click link → land on /dashboard)
- [ ] Session persists across page refreshes and browser restarts
- [ ] Unauthenticated visit to any `/portal` route redirects to `/login`
- [ ] Authenticated visit to `/login` redirects to `/dashboard`
- [ ] Bottom nav renders on all portal pages with correct active highlighting
- [ ] Back button works on Header component
- [ ] Safe area padding respected on iPhone Safari (content not hidden behind nav bar)
- [ ] All 7 portal tables exist in Supabase dev with RLS enabled
- [ ] `payment-proofs` Storage bucket exists with correct privacy and size settings
- [ ] `database.types.ts` generated and committed
- [ ] Seed data inserted — Chan Family with 2 learners visible in Supabase table editor
- [ ] Developer can log in and the dashboard route resolves (even if blank — data comes in Week 2)
- [ ] Schema freeze announced — no further schema changes without approval

---

## WEEK 2 — Core Modules (Days 9–13)

> Build all parent-facing read pages. Same query-render pattern repeated. Fast delivery.

### Day 9 — Dashboard + Learner Profile (7h)

#### Dashboard (4h)

- [ ] Create `src/lib/hooks/useFamily.ts`:
  - Queries `portal_accounts` → `contacts` → `family_id`
  - Queries `families` for family name
  - Queries `learners` for all learners in family
  - Returns `{ family, learners, loading, error }`
- [ ] Create `src/components/dashboard/LearnerCard.tsx`:
  - Displays: learner name, current level, enrollment status badge
  - Status badge colours: Successful Enrollment=green, Paused=yellow, Quit=red, others=grey
  - Full card is a `<Link>` to `/learners/{learnerId}`
  - Active press state (`active:scale-95`) for touch feedback
- [ ] Create `src/lib/hooks/useUnreadCount.ts`:
  - Counts broadcasts not in `broadcast_reads` for this family
  - Returns unread count as integer
- [ ] Create `src/components/dashboard/InboxBadge.tsx`:
  - Red circle badge showing unread count
  - Hidden when count is 0
  - Shows "9+" when count exceeds 9
- [ ] Create `src/app/(portal)/dashboard/page.tsx`:
  - Server component — fetches family + learners server-side
  - Renders `<Header title="{Family Name} Family" />`
  - Renders `<LearnerCard>` for each learner, ordered alphabetically
  - Shows `<EmptyState>` if no learners found
  - Shows loading skeleton while data loads
- [ ] Dashboard manual tests:
  - [ ] Family name appears in header ("Chan Family")
  - [ ] Both learners (Emma and Ethan) appear as cards
  - [ ] Emma's card shows "Stage 2" and "Successful Enrollment" (green badge)
  - [ ] Tapping Emma's card navigates to `/learners/{emmaId}`
  - [ ] Inbox badge shows correct unread count (0 initially)

#### Learner Profile (3h)

- [ ] Create `src/lib/utils/levels.ts`:
  - `LEVEL_SEQUENCE` array: Kinder, 25m, Stage 1, Stage 2, 200m, Stage 3, 400m, Stage 4 (Bronze), 1000m, Stage 5 (Silver), 1500m, Stage 6 (Gold)
  - `getLevelIndex(level)` → returns 0-based index, -1 if not found
  - `getLevelProgressPercent(level)` → returns 0–100
  - `getNextLevel(level)` → returns next level string or null if at top
- [ ] Create `src/app/(portal)/learners/[learnerId]/page.tsx`:
  - Server component — fetches learner + active enrollment with pool/coach/package joins
  - Renders level, enrollment status, pool, coach, class day/timeslot, package name
  - Grid of 5 quick-nav links: Schedule, Attendance, Progress, Credits, Tests
  - Shows `notFound()` if learner does not belong to authenticated family
- [ ] Learner profile manual tests:
  - [ ] Emma's profile shows "Stage 2", "Silver", "Saturday 9:00am", "Bishan", "Coach Don"
  - [ ] Ethan's profile shows "Kinder", "Platinum", "Sunday 10:00am", "Sengkang", "Coach Ben"
  - [ ] Attempting to access another family's learner ID returns 404
  - [ ] All 5 nav links navigate to the correct sub-pages

- [ ] Commit: `git commit -m "feat: add family dashboard and learner profile"`

---

### Day 10 — Schedule + Attendance (7h)

#### Dates Utility (1h)

- [ ] Create `src/lib/utils/dates.ts`:
  - `formatSGT(dateStr: string, fmt: string): string` — converts ISO date to SGT display
  - `formatTimestampSGT(ts: string, fmt: string): string` — converts full timestamp to SGT
  - `isToday(dateStr: string): boolean` — checks if date is today in SGT
  - `isFuture(dateStr: string): boolean` — checks if date is in the future in SGT
- [ ] Unit test dates utility:
  - [ ] `formatSGT('2026-06-01', 'EEE d MMM yyyy')` returns "Mon 1 Jun 2026"
  - [ ] A UTC timestamp of `2026-06-01T16:00:00Z` displays as "01 Jun 2026 00:00" in SGT (UTC+8 = midnight)
  - [ ] `isToday` returns true for today's date string, false for yesterday/tomorrow

#### Schedule Page (3h)

- [ ] Create `src/app/(portal)/learners/[learnerId]/schedule/page.tsx`:
  - Server component — fetches enrollment for learner, then class sessions ordered by date
  - Splits into "Upcoming" (today and future) and "Past" (previous) sections
  - Each session card shows: date (SGT), day of week, start–end time, pool name, coach name, status badge
  - Status badge colours: Scheduled=blue, Completed=green, Cancelled=red
  - Shows cancellation reason when status is Cancelled
  - Shows `<EmptyState>` if no sessions
- [ ] Schedule manual tests:
  - [ ] Emma's schedule shows 5 upcoming Saturday sessions
  - [ ] Dates display in SGT, not UTC
  - [ ] Past sessions (3 weeks ago) appear in "Past" section
  - [ ] Cancelled session would show cancellation reason

#### Attendance Page (3h)

- [ ] Create `src/components/layout/AttendanceStatusBadge.tsx`:
  - Present = green
  - Absent = red
  - CancelledBySchool = yellow + "Cancelled by school" label
  - ReplacementUsed = blue + "Replacement class" label
- [ ] Create `src/app/(portal)/learners/[learnerId]/attendance/page.tsx`:
  - Server component — fetches `attendance_records` joined with `class_sessions` + `pools`
  - Ordered by session date descending (most recent first)
  - Each row: date (SGT), time, pool name, status badge
  - Shows "Credit issued" indicator when `replacement_credit_generated = true`
  - Shows `<EmptyState>` if no records
- [ ] Attendance manual tests:
  - [ ] Emma shows 2 past records: Present (2 weeks ago), Absent (1 week ago, "Credit issued")
  - [ ] Ethan shows 1 past record: Present (1 week ago)
  - [ ] Absent record shows "Credit issued" tag
  - [ ] Status badges use correct colours

- [ ] Commit: `git commit -m "feat: add schedule and attendance pages with SGT timezone handling"`

---

### Day 11 — Progress + Replacement Credits (7h)

#### Progress Page (3.5h)

- [ ] Create `src/components/progress/LevelProgressBar.tsx`:
  - Horizontal bar, filled proportionally based on `getLevelProgressPercent()`
  - Smooth fill animation on mount
  - Shows percentage label
- [ ] Create `src/app/(portal)/learners/[learnerId]/progress/page.tsx`:
  - Server component — fetches learner level
  - Shows current level prominently
  - Shows `<LevelProgressBar>` with percentage
  - Shows "Next milestone: {nextLevel}" if not at top
  - Shows full level path as ordered list — current level highlighted, completed levels struck-through
  - Shows "Coming soon: coach notes" placeholder (real coach notes require a future schema addition)
- [ ] Progress manual tests:
  - [ ] Emma (Stage 2): progress bar shows ~25% filled, next level = "200m"
  - [ ] Level list shows Kinder, 25m, Stage 1, Stage 2 as prior, Stage 2 highlighted, 200m onwards as upcoming
  - [ ] Ethan (Kinder): progress bar shows ~8%, next level = "25m"

#### Replacement Credits Page (3.5h)

- [ ] Create `src/lib/utils/credits.ts`:
  - `isUnlimitedPackage(packageName: string): boolean` — returns true for Platinum
  - `formatCreditStatus(status: string): { label: string, colour: string }` — returns display properties
- [ ] Create `src/app/(portal)/learners/[learnerId]/credits/page.tsx`:
  - Server component — fetches enrollment → package to determine package type
  - **If Platinum:** shows "Platinum Package — Unlimited classes, no replacement credits needed"
  - **If Silver or Gold:**
    - Shows total available credits count prominently
    - Lists all credits ordered by created_at descending
    - Each credit: missed class date, credit status badge (Available=green, Used=grey, Expired=red, Cancelled=grey)
    - "Available" credits have a distinct visual treatment
  - Shows `<EmptyState>` if no credits
- [ ] Credits manual tests:
  - [ ] Ethan (Platinum): sees "Platinum Package" message, no credit list
  - [ ] Emma (Silver): sees "1 credits available", sees 1 Available credit linked to her absent class
  - [ ] Credit date matches the session date from seed data

- [ ] Commit: `git commit -m "feat: add progress page with level bar and package-aware replacement credits"`

---

### Day 12 — Test Status + Receipts + Payments Shell (6h)

#### Test Status Page (2h)

- [ ] Create `src/app/(portal)/learners/[learnerId]/tests/page.tsx`:
  - Server component — fetches learner level and enrollment status
  - Displays current level
  - Shows "Test scheduling is managed by our team. You will be notified when a test is arranged."
  - Shows a future roadmap note: "Upcoming test details will appear here once assigned by the team."
- [ ] Test status manual tests:
  - [ ] Emma's test page shows "Stage 2"
  - [ ] Page renders without errors for both learners

#### Receipts Page (2h)

- [ ] Create `src/app/(portal)/receipts/page.tsx`:
  - Server component — fetches payments with `payment_status_extended = 'Completed'`
  - Joins with invoices for period_start/period_end
  - Ordered by payment_date descending
  - Each row: amount (SGD), payment date (SGT), term period covered
  - Shows "Paid" badge in green
  - Shows `<EmptyState message="No payment history yet." />` if no completed payments
- [ ] Receipts manual tests:
  - [ ] Shows 1 completed payment from seed data
  - [ ] Amount, date, and term period are correct
  - [ ] Empty state shows if no completed payments exist

#### Payments Page Shell (2h)

- [ ] Create `src/app/(portal)/payments/page.tsx`:
  - Server component — fetches invoices with payments joined
  - Ordered by invoice_date descending
  - Invoice card: amount, due date (SGT), invoice status badge
  - Invoice status colours: Unpaid=red, Overdue=dark red bold, Paid=green
  - Download PayNow QR button (visible only for unpaid invoices with a `paynow_qr_url`)
  - Placeholder for `<ProofUpload>` (built Day 13)
  - Shows `<EmptyState>` if no invoices
- [ ] Payments shell manual tests:
  - [ ] Unpaid invoice (SGD 280) shows with red "Unpaid" badge
  - [ ] Due date displays in SGT
  - [ ] "Download PayNow QR" button appears
  - [ ] Completed invoice shows green "Paid" badge

- [ ] Commit: `git commit -m "feat: add test status, receipts, and payments pages"`

---

### Day 13 — Payments Complete (8h)

#### PayNow QR Download (2h)

- [ ] Create `src/app/api/payments/paynow-qr/route.ts`:
  - Accepts `paymentId` query param
  - Verifies payment belongs to authenticated family (RLS check)
  - Returns the `paynow_qr_url` as a signed Supabase Storage URL (if stored in Storage) OR the raw URL (if external)
  - Returns 404 if not found, 403 if not authorised
- [ ] Create `src/components/payments/PayNowQR.tsx`:
  - "Download PayNow QR" button
  - On click: calls `/api/payments/paynow-qr?paymentId=X`
  - Opens signed URL in new tab / triggers download
  - Shows loading state during fetch
- [ ] Show PayNow instructions below the QR button:
  1. Tap Download PayNow QR
  2. Open your bank app
  3. Scan or upload the QR code
  4. Complete payment using PayNow
  5. Keep a screenshot of your confirmation
  6. Upload it below as proof

#### Proof of Payment Upload (4h)

- [ ] Create `src/components/payments/ProofUpload.tsx`:
  - Hidden file input (`accept="image/*,application/pdf"`)
  - "Upload Payment Proof" button triggers input
  - On file select:
    - Validates file size ≤ 10MB, shows error if exceeded
    - Validates file type, shows error if invalid
    - Uploads to `payment-proofs/{familyId}/{paymentId}-{timestamp}.{ext}`
    - Calls `/api/payments/upload-proof` with payment ID and Storage URL
    - Shows upload progress indicator
  - After success: shows "Proof submitted — our team will verify shortly"
  - Component hidden if `payment_status_extended` is already Submitted/Verified/Completed
- [ ] Create `src/app/api/payments/upload-proof/route.ts`:
  - Accepts `{ paymentId, proofUrl }` in request body
  - Updates `payments` row: `proof_url`, `proof_uploaded_at`, `payment_status_extended = 'Submitted'`
  - Returns `{ ok: true }` on success
  - Returns 400 if missing fields, 500 if DB error

#### End-to-End Payment Flow Test (2h)

- [ ] Test complete payment flow:
  - [ ] Log in as Chan Family
  - [ ] Navigate to Payments
  - [ ] Unpaid invoice visible with correct amount and due date
  - [ ] "Download PayNow QR" downloads/opens the QR image
  - [ ] PayNow instructions are visible and readable
  - [ ] Tap "Upload Payment Proof" → file picker opens
  - [ ] Select a valid JPG file → upload completes without error
  - [ ] Success message "Proof submitted" appears
  - [ ] Upload button disappears after submission
  - [ ] Check Supabase dashboard: `payments` row now has `proof_url` set and `payment_status_extended = 'Submitted'`
  - [ ] Select a file > 10MB → error message appears
  - [ ] Select an invalid file type (e.g. .exe) → error message appears

- [ ] Commit: `git commit -m "feat: complete payments — PayNow QR download and proof of payment upload"`

### Week 2 Deliverable Checklist

- [ ] Dashboard shows correct family name and both learners with accurate level + status
- [ ] Tapping a learner card navigates to their profile
- [ ] Learner profile shows pool, coach, class day/time, package — all matching seed data
- [ ] Schedule shows upcoming sessions ordered by date, in SGT timezone
- [ ] Attendance shows past records with correct status badges and "Credit issued" indicator
- [ ] Progress bar fills correctly based on level (Stage 2 = ~25%)
- [ ] Kinder = Kinder (not blank), Stage 4 = Stage 4 (Bronze) — verify naming matches CRM exactly
- [ ] Emma (Silver) sees credits; Ethan (Platinum) sees the unlimited message
- [ ] Payments page shows unpaid invoice with correct amount and due date
- [ ] PayNow QR button works (opens/downloads the QR)
- [ ] File upload flow works: file → Supabase Storage → status = Submitted
- [ ] File validation works: rejects files > 10MB and invalid types
- [ ] Receipts page shows completed payment
- [ ] Test status page renders without errors
- [ ] All dates and times on ALL pages display in SGT, not UTC
- [ ] All pages tested on iPhone Safari (iOS) and Chrome (Android)
- [ ] No page has a blank crash on mobile — loading and empty states render correctly

---

## WEEK 3 — Communications, Admin Tools & Staging (Days 14–18)

### Day 14 — Inbox + Supabase Realtime (6h)

#### Broadcasts Hook (2h)

- [ ] Create `src/lib/hooks/useBroadcasts.ts`:
  - Initial load: fetches all broadcasts where `expires_at IS NULL OR expires_at > now()`
  - Fetches `broadcast_reads` for this family to build a `Set<number>` of read broadcast IDs
  - Supabase Realtime: subscribes to `INSERT` events on `broadcasts` table
  - On new INSERT: prepends the new broadcast to the list
  - `markRead(broadcastId)`: inserts into `broadcast_reads`, updates local `readIds` set
  - Exposes `{ broadcasts, readIds, markRead, loading }`

#### Broadcast Components (2h)

- [ ] Create `src/components/inbox/BroadcastTypeIcon.tsx`:
  - Maps each type to its emoji/icon: Lightning=⚡, BadWeather=🌧️, CoachAbsence=👤, PoolClosure=🏊, ClassCancellation=❌, ScheduleChange=📅, Announcement=📢, TestAnnouncement=🎖️
- [ ] Create `src/components/inbox/BroadcastCard.tsx`:
  - Shows type icon, title, body text, sent timestamp (SGT)
  - Unread: highlighted border/background + blue dot indicator
  - Read: reduced opacity
  - `useEffect` calls `markRead` when card mounts (auto-mark as read on view)
- [ ] Create `src/components/inbox/BroadcastList.tsx`:
  - Client component using `useBroadcasts` hook
  - Renders list of `<BroadcastCard>` components
  - Shows `<EmptyState message="No messages yet." />` when empty

#### Inbox Page (1h)

- [ ] Create `src/app/(portal)/inbox/page.tsx`:
  - Server component — resolves `familyId` server-side
  - Passes `familyId` to `<BroadcastList>`
- [ ] Update `InboxBadge` to use real `useUnreadCount` hook

#### Realtime Test (1h)

- [ ] Open portal on device (staging or localhost via ngrok)
- [ ] In Supabase Studio: insert a new Lightning broadcast (`type='Lightning', title='⚡ Class suspended', body='Lightning detected at Bishan. Class suspended at 4pm.', is_global=true`)
- [ ] Verify: broadcast appears in portal inbox INSTANTLY without page refresh
- [ ] Verify: unread badge count increments on Inbox tab
- [ ] Tap the broadcast card: verify it moves to "read" state (dot disappears, opacity reduced)
- [ ] Verify: unread count decrements after reading

- [ ] Commit: `git commit -m "feat: add inbox with Supabase Realtime broadcast delivery"`

---

### Day 15 — Jamie + Support Escalation (7h)

#### Jamie Chat (3h)

- [ ] Create `src/components/jamie/JamieChat.tsx`:
  - Chat bubble UI: Jamie messages on left (grey bg), parent messages on right (primary bg)
  - Initial greeting message from Jamie
  - Text input + Send button at bottom (fixed above keyboard on mobile)
  - Enter key submits message
  - Auto-scrolls to bottom on new message
  - Messages stored in local state (no persistence in MVP)
- [ ] Implement FAQ keyword engine in `JamieChat`:
  - Define minimum 10 keyword → answer mappings:
    - `replacement` / `credit` / `missed class` → explains replacement credit policy
    - `paynow` / `qr` / `payment qr` → explains PayNow download steps
    - `payment status` / `paid` / `invoice` → guides to Payments page
    - `level` / `stage` / `progress` → explains level system
    - `test` / `swims safer` / `swimsafer` → explains test eligibility and admin-assigned process
    - `cancel` / `cancelled` / `weather` / `lightning` → explains school cancellation and credit policy
    - `coach` → explains what to do if coach is absent
    - `attendance` → guides to Attendance page
    - `schedule` / `class time` → guides to Schedule page
    - `contact` / `help` / `human` / `speak to someone` → triggers escalation prompt
  - Default fallback: "I'm not sure about that — let me connect you with our team. Tap Contact Support below."
  - Case-insensitive matching
- [ ] Create `src/app/(portal)/jamie/page.tsx`:
  - Full-height chat layout that respects mobile keyboard
  - Header with back button and "Ask Jamie" title
- [ ] Jamie manual tests:
  - [ ] Type "replacement" → receives correct replacement credit explanation
  - [ ] Type "how do I pay" → receives PayNow steps
  - [ ] Type "what level is Stage 4" → receives Bronze explanation
  - [ ] Type "random xyz" → receives fallback message
  - [ ] Keyboard pushes chat input up on mobile (no hidden input)
  - [ ] Chat auto-scrolls to newest message

#### Support Escalation (4h)

- [ ] Create `src/components/support/EscalationForm.tsx`:
  - Issue type selector: Payment / Replacement / Schedule / Progress / Test / Login / General (pill-style toggles)
  - Free text message field (required, min 10 characters)
  - Submit button: "Send to support team"
  - Loading state during submission
  - Success state: "Message sent — our team will get back to you on WhatsApp shortly"
  - Error state: "Something went wrong — please try again or WhatsApp us directly"
- [ ] Create `src/app/(portal)/support/page.tsx`:
  - Server component — fetches family + contact name + phone (`contacts.contact_phone`)
  - Passes `{ contactName, contactPhone, familyId }` to `<EscalationForm>`
- [ ] Create `src/app/api/support/escalate/route.ts`:
  - Saves ticket to `support_tickets` table
  - Fetches parent phone from `contacts` table via `portal_accounts → contacts.contact_phone`
  - Calls WATI API to send WhatsApp message to parent's phone:
    ```ts
    const res = await fetch(
      `${process.env.WATI_API_ENDPOINT}/api/v1/sendSessionMessage/${contactPhone}`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${process.env.WATI_API_KEY}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          messageText: `[TSS Support] ${contactName} sent a message via the parent portal:\n\nIssue: ${issueType}\nMessage: ${message}\n\nReply here to respond directly to this parent on WhatsApp.`,
        }),
      }
    )
    const watiData = await res.json()
    ```
  - On WATI success: updates `support_tickets.wati_sent = true`, stores `wati_message_id`
  - Returns `{ ok: true }` on success
- [ ] End-to-end test:
  - [ ] Submit a "Payment" escalation with message "My payment is not showing as verified"
  - [ ] WhatsApp message appears in WATI coach portal inbox within 5 seconds
  - [ ] WATI message contains correct parent name and issue details
  - [ ] Success state shows in the form
  - [ ] Supabase: `support_tickets` row exists with `wati_sent = true`
  - [ ] Try submitting with empty message → button is disabled

- [ ] Commit: `git commit -m "feat: add Jamie FAQ assistant and support escalation via WATI WhatsApp"`

---

### Day 16 — Admin Tools for Ops (7h)

> These pages are not parent-facing. They are for the ops team only. Protected behind an admin check.

#### Admin Auth Guard (1h)

- [ ] Create admin route guard in `src/app/(portal)/admin/layout.tsx`:
  - Checks authenticated user's email against an allowed admin email list stored in env var `ADMIN_EMAILS` (comma-separated)
  - Redirects to `/dashboard` if user is not in the admin list
  - Add `ADMIN_EMAILS` to `.env.local` and `.env.example`

#### Broadcast Creation Form (3h)

- [ ] Create `src/app/(portal)/admin/broadcasts/page.tsx` with `src/components/admin/BroadcastForm.tsx`:
  - Type selector (dropdown): Lightning / BadWeather / CoachAbsence / PoolClosure / ClassCancellation / ScheduleChange / Announcement / TestAnnouncement
  - Pool selector (dropdown, populated from `pools` table) — optional, leave blank for all locations
  - Title input (required, max 100 chars)
  - Body textarea (required, max 500 chars)
  - Expires at datetime picker (optional)
  - Preview: shows how the broadcast will look in the parent's inbox before submitting
  - Submit button: "Send Broadcast"
  - On success: broadcast inserted into `broadcasts` table, form resets, success toast
  - On error: shows error message
- [ ] Admin broadcast form tests:
  - [ ] Submit a Lightning broadcast → confirm it appears in parent inbox (and via Realtime if portal is open)
  - [ ] Submit without title → validation error shown
  - [ ] Pool selector shows all active pools from `pools` table
  - [ ] Broadcast with `pool_id` set → parents at other pools do not see it (verify via RLS or pool filtering)

#### Payment Verification Page (3h)

- [ ] Create `src/app/(portal)/admin/payments/page.tsx` with `src/components/admin/PaymentVerificationList.tsx`:
  - Lists all payments where `payment_status_extended = 'Submitted'`
  - Each row shows: family name, contact name, invoice amount, due date, proof image thumbnail (clickable to open full image)
  - "Verify" button: sets `payment_status_extended = 'Verified'`, `verified_by = adminEmail`, `verified_at = now()`
  - "Reject" button: sets `payment_status_extended = 'Failed'`, shows rejection note field
  - After action: row updates in-place (optimistic UI)
  - Shows "No pending verifications" when empty
- [ ] Payment verification tests:
  - [ ] Page shows the proof upload from Day 13 test (Submitted payment)
  - [ ] Click proof thumbnail → full image opens
  - [ ] Click "Verify" → status updates to Verified in Supabase, row disappears from list
  - [ ] Verify in Supabase: `verified_by` contains admin email, `verified_at` is set
  - [ ] Non-admin user visiting `/admin/*` is redirected to `/dashboard`

- [ ] Commit: `git commit -m "feat: add admin broadcast creation form and payment verification page"`

---

### Day 17 — Polish Pass (7h)

#### Loading States (2h)

- [ ] Wrap every `page.tsx` in `(portal)` in `<Suspense fallback={<LoadingSpinner />}>`
- [ ] Verify: navigating to any page never shows a blank white screen — spinner appears while data loads
- [ ] Add skeleton cards to Dashboard (2 skeleton learner cards while loading)
- [ ] Add skeleton rows to Schedule, Attendance, Credits, Receipts pages

#### Empty States (2h)

- [ ] Verify every list page has a meaningful empty state:
  - [ ] Schedule: "No upcoming classes scheduled."
  - [ ] Attendance: "No attendance records yet."
  - [ ] Credits (Silver/Gold with no credits): "No replacement credits yet."
  - [ ] Payments: "No invoices found. Contact us if this looks wrong."
  - [ ] Receipts: "No payment history yet."
  - [ ] Inbox: "No messages yet. We'll notify you of any important updates here."
  - [ ] Support: "No previous escalations."

#### Error States (1h)

- [ ] Add error boundary to portal layout (`error.tsx` in `(portal)`)
- [ ] Error page shows: "Something went wrong. Please refresh or contact support."
- [ ] Test: temporarily break a Supabase query to confirm error boundary catches it

#### Mobile Viewport Audit (2h)

- [ ] Test every page on iPhone 14 viewport (390×844, Safari):
  - [ ] No horizontal scroll
  - [ ] No content cut off by bottom nav bar
  - [ ] Input fields don't cause page zoom
  - [ ] Touch targets are at least 44px tall
- [ ] Test every page on Android viewport (360×800, Chrome):
  - [ ] Same checks as above
  - [ ] Keyboard does not obscure input fields on support form and Jamie chat

#### PWA & Meta Tags (1h)

- [ ] Update `src/app/layout.tsx` metadata: title, description, viewport (no user-scaling, viewport-fit=cover)
- [ ] Create `public/manifest.json` (name, short_name, display=standalone, start_url=/dashboard, theme_color)
- [ ] Add `<link rel="manifest">` to layout
- [ ] Add apple-mobile-web-app meta tags for iOS home screen behaviour
- [ ] Test: "Add to Home Screen" on iPhone → icon appears, opens in standalone mode

- [ ] Commit: `git commit -m "feat: polish — loading states, empty states, error boundary, PWA manifest"`

---

### Day 18 — Staging Deploy + Ops Training (5h)

#### Staging Deploy (2h)

- [ ] Push repository to GitHub
- [ ] Connect to Vercel: `npx vercel` → select the GitHub repo
- [ ] In Vercel project settings → add all 4 environment variables (dev Supabase keys + Slack webhook)
- [ ] In Supabase Auth → URL Configuration → add Vercel staging URL to Redirect URLs:
  - `https://tss-parent-portal.vercel.app/auth/callback` (or your staging domain)
- [ ] Deploy: `vercel --prod` (staging for now — production deploy happens Day 22)
- [ ] Staging smoke test:
  - [ ] Magic link login works on staging URL
  - [ ] Dashboard loads with Chan Family seed data
  - [ ] At least one complete user journey works end-to-end: login → dashboard → learner → schedule

#### Ops Training Session (1.5h)

> Block 90 minutes with the ops team. Cover all 4 workflows.

- [ ] **Workflow 1: Creating a new `portal_accounts` row when a family enrols**
  - Show ops how to find the `contact_id` in Supabase for the new family
  - Walk through the INSERT: `INSERT INTO portal_accounts (contact_id, auth_user_id, email) VALUES (?, ?, ?)`
  - Note: `auth_user_id` comes from Supabase Auth after inviting the parent via dashboard → Authentication → Users → Invite user
  - Ops team completes one practice run with a test email
  - [ ] Ops team can create a `portal_accounts` row independently

- [ ] **Workflow 2: Sending a broadcast (lightning/weather/announcement)**
  - Show ops the `/admin/broadcasts` page on staging
  - Walk through each field: type, pool (leave blank for all), title, body, expiry
  - Demo: send a test Lightning broadcast → show it appearing live in the parent inbox
  - [ ] Ops team sends their own test broadcast independently

- [ ] **Workflow 3: Verifying a parent's PayNow payment**
  - Show ops the `/admin/payments` page on staging
  - Walk through: view proof image → verify → confirm status change
  - [ ] Ops team verifies a test payment independently

- [ ] **Workflow 4: Responding to parent support messages in WATI**
  - Show ops/coaches the WATI inbox in the coach portal
  - Explain: when a parent submits a support message in the portal, it arrives as a WhatsApp message in WATI under the parent's contact number
  - Define the SLA (e.g., respond within X business hours — fill in per ops policy)
  - Review what fields appear in the WATI message (parent name, issue type, message)
  - [ ] Ops/coach confirms they can see the message in WATI and send a WhatsApp reply

#### Ops Self-Test (1h)

- [ ] Ops team independently:
  - [ ] Sends a broadcast via admin form → confirms it appears in their own parent-view inbox on staging
  - [ ] Uploads a test payment proof as a parent → logs into admin → verifies it
  - [ ] Submits a support escalation as a parent → confirms WATI WhatsApp message appears in coach portal inbox
  - [ ] Signs off: "Staging workflows confirmed by ops team" (written sign-off in chat/email)

### Week 3 Deliverable Checklist

- [ ] Inbox receives new broadcasts within 2 seconds via Supabase Realtime (tested on actual mobile device)
- [ ] Unread badge count is accurate and decrements correctly as broadcasts are read
- [ ] Jamie answers all 10 FAQ keyword categories correctly
- [ ] Fallback response guides parent to contact support
- [ ] WATI WhatsApp message appears in coach portal inbox within 5 seconds, with correct parent name and issue info
- [ ] `support_tickets` row created in Supabase with `wati_sent = true`
- [ ] Admin broadcast form creates a broadcast that parents see in real-time
- [ ] Admin payment page shows Submitted payments, Verify action updates status correctly
- [ ] Admin pages return 302 redirect to `/dashboard` for non-admin users
- [ ] Loading spinners appear on all pages during data fetch
- [ ] Empty states present on every list page
- [ ] Error boundary catches and displays a user-friendly message
- [ ] All pages pass mobile viewport audit (no overflow, no hidden inputs, 44px touch targets)
- [ ] Staging URL deployed and accessible
- [ ] Magic link login works on staging URL
- [ ] Ops team has completed all 4 workflow self-tests and provided written sign-off
- [ ] Ops team can operate all admin workflows independently without dev support

---

## WEEK 4 — UAT + Bug Fix + Production Launch (Days 19–23)

### Days 19–20 — User Acceptance Testing (UAT)

#### UAT Setup (Day 19 AM, 1.5h)

- [ ] For each UAT parent (3–5 parents):
  - [ ] Insert their `portal_accounts` row in staging Supabase (linked to their real CRM `contact_id`)
  - [ ] In Supabase dashboard → Authentication → Users → Invite user → enter their email
  - [ ] Copy the `auth_user_id` from the created auth user and insert into `portal_accounts`
  - [ ] Verify their family has real data: at least 1 learner, 1 upcoming class, 1 invoice
- [ ] Prepare and send UAT instructions to each parent via WhatsApp:
  ```
  Hi [Name], we're testing a new parent portal for The Swim Starter.
  Please tap this link and follow the test checklist:
  [staging URL]
  
  Test checklist:
  1. Log in using the magic link sent to your email
  2. Can you see your child's name and level?
  3. Can you find their upcoming schedule?
  4. Does the attendance history look correct?
  5. Can you see your invoice and download the PayNow QR?
  6. Go to Inbox — send yourself a test message (ops will send one)
  7. Try asking Jamie: "how do replacement classes work?"
  8. Send a support message from the Support page
  9. Reply to us with: what worked, what was confusing, what was wrong
  ```

#### UAT Monitoring (Day 19–20, Ongoing)

- [ ] Monitor Supabase logs for errors during UAT window
- [ ] Monitor Slack escalation channel — each UAT escalation confirms the flow works
- [ ] Collect feedback from each UAT parent (WhatsApp or form)
- [ ] Log all issues in a bug tracker (Notion, Linear, or simple spreadsheet) with:
  - Issue description
  - Device and browser
  - Steps to reproduce
  - Severity: Critical (blocks use) / High (significant friction) / Low (cosmetic)

#### UAT Triage (Day 20 EOD, 2h)

- [ ] Review all reported issues
- [ ] Classify each as Critical / High / Low
- [ ] Confirm: any Critical issues must be resolved before production deploy
- [ ] Confirm: any High issues to be resolved before production deploy (or noted as known issues)
- [ ] Low issues deferred to post-launch patch

---

### Day 21 — Bug Fix Sprint (Up to 8h)

- [ ] Fix all Critical severity UAT bugs
- [ ] Fix all High severity UAT bugs
- [ ] For each fix: write a brief description of what changed
- [ ] Re-test fixed items:
  - [ ] At least 1 UAT parent re-tests Critical fixes on their device
  - [ ] Developer self-tests all High fixes on iPhone Safari and Android Chrome
- [ ] If any Critical bug cannot be fixed within Day 21: assess whether to delay production deploy (PM decision)
- [ ] Commit all fixes: `git commit -m "fix: UAT feedback — [brief description]"`

---

### Day 22 — Production Deploy (5h)

#### Production Supabase Setup (2h)

- [ ] Run all table creation migrations on production Supabase (same SQL as dev Day 4)
- [ ] Run `ALTER TABLE payments` migration on production
- [ ] Deploy `get_family_id()` function on production
- [ ] Apply all 17 RLS policies on production
- [ ] Create `payment-proofs` Storage bucket on production with same settings as dev
- [ ] Verify: all tables visible, RLS enabled, Storage bucket exists
- [ ] Do NOT insert seed data on production

#### Real Family Data Import (1.5h, Ops + Dev)

- [ ] Ops team provides list of all currently enrolled families with:
  - Family name
  - Primary contact name
  - Contact email address
  - `contact_id` from CRM
- [ ] For each family:
  - [ ] In production Supabase Auth → Users → Invite user → enter email
  - [ ] Copy the `auth_user_id`
  - [ ] Insert row: `INSERT INTO portal_accounts (contact_id, auth_user_id, email) VALUES (...)`
- [ ] Verify: count of `portal_accounts` rows matches number of families being onboarded
- [ ] Spot-check: 3 random families — confirm their learners, enrollments, and invoices exist in the shared production DB

#### Vercel Production Deploy (1h)

- [ ] Create a separate Vercel project for production (or promote staging to prod)
- [ ] Set environment variables to PRODUCTION Supabase keys (not dev)
- [ ] In production Supabase Auth → URL Configuration → add production Vercel URL to Redirect URLs
- [ ] Deploy: `vercel --prod`
- [ ] Production smoke test (use one of the real family accounts):
  - [ ] Magic link login works on production URL
  - [ ] Family dashboard loads with correct learner(s)
  - [ ] Schedule loads with upcoming classes
  - [ ] No console errors in browser dev tools
  - [ ] No errors in Supabase logs

#### Pre-Launch Checklist

- [ ] All Critical and High UAT bugs are fixed and confirmed
- [ ] Production environment variables point to PRODUCTION Supabase (not dev)
- [ ] Production Supabase Auth redirect URL is correct
- [ ] PDPA consent text on login page is approved and correct
- [ ] Admin email list is set correctly in `ADMIN_EMAILS` env var
- [ ] `WATI_API_ENDPOINT` and `WATI_API_KEY` in production env are correct production WATI credentials
- [ ] Ops team has been given the production `/admin` URL and confirmed it works

---

### Day 23 — Soft Launch (Ongoing)

#### Launch (AM)

- [ ] Ops team sends first WhatsApp broadcast to parent cohort:
  ```
  Hi [Name]! The Swim Starter parent portal is now live.
  Tap the link below to log in and view [child's name]'s class schedule,
  progress, and upcoming payments.
  [portal URL]
  Your login link will be sent to: [email]
  ```
- [ ] Ops team sends Supabase Auth magic link invitations to all onboarded parents
- [ ] Confirm: all parents have received their invitation emails

#### Monitoring (Day 23, All Day)

- [ ] Watch Supabase logs for any 5xx errors — investigate immediately
- [ ] Watch WATI coach portal inbox — respond to every parent support message within the SLA
- [ ] Check Supabase Storage: confirm payment proof uploads are arriving correctly
- [ ] Check Realtime: send a test broadcast via admin form → confirm it arrives on at least 1 live parent device
- [ ] After 2 hours: tally escalation count by issue type — any pattern indicates a bug or UX issue

#### Post-Launch Log (Ongoing)

- [ ] Create a post-launch bug log document
- [ ] Any parent-reported issue is triaged same-day
- [ ] Plan first patch release for Day 25–26 based on launch feedback

### Week 4 Deliverable Checklist

- [ ] 3+ UAT parents confirmed portal works correctly on their personal devices
- [ ] Zero Critical bugs remaining before production deploy
- [ ] All High bugs resolved (or accepted as known with workaround documented)
- [ ] Production Supabase is separate from dev — confirmed by checking project IDs
- [ ] All enrolled families have `portal_accounts` rows in production
- [ ] Production deploy is live on correct URL
- [ ] Magic link works end-to-end on production URL (tested with real parent email)
- [ ] Ops team has sent WhatsApp invitations to all enrolled parents
- [ ] Ops team / coaches are monitoring WATI inbox actively and responding to parent support messages
- [ ] Post-launch bug log document created and shared with team

---

## Ongoing — Post-Launch Operations

### Recurring Ops Workflow: New Family Enrolment

Every time a new family enrols, ops must:

1. [ ] Confirm family exists in CRM (`families` table) and get their `family_id`
2. [ ] Confirm primary contact exists (`contacts` table) and get `contact_id`
3. [ ] In Supabase dashboard → Authentication → Users → Invite user → enter parent email
4. [ ] Note the `auth_user_id` from the created auth user record
5. [ ] Run: `INSERT INTO portal_accounts (contact_id, auth_user_id, email) VALUES (<contact_id>, '<auth_user_id>', '<email>')`
6. [ ] Confirm parent receives the Supabase Auth invitation email
7. [ ] Inform parent: "You can now access the parent portal. Check your email for a login link."

### Recurring Ops Workflow: Sending a Broadcast

1. [ ] Go to `[portal-url]/admin/broadcasts`
2. [ ] Select broadcast type (Lightning / BadWeather / etc.)
3. [ ] Select pool (leave blank to send to all locations)
4. [ ] Write title (short, under 100 chars)
5. [ ] Write body (clear and actionable — what happened, what parent should do)
6. [ ] Set expiry if relevant (e.g. lightning alert expires in 4 hours)
7. [ ] Review preview
8. [ ] Tap "Send Broadcast" — confirm it appears in your own inbox

### Recurring Ops Workflow: Verifying a Payment

1. [ ] Go to `[portal-url]/admin/payments`
2. [ ] See list of Submitted payments
3. [ ] Tap proof thumbnail to view the uploaded screenshot
4. [ ] Verify it matches the invoice amount and the TSS UEN
5. [ ] Tap "Verify" to confirm, or "Reject" with a reason if invalid
6. [ ] Parent's payment status updates to Verified automatically

---

## Key Decisions Locked In

| Decision | Value |
|---|---|
| Database | Supabase (shared with CRM — no separate database) |
| Authentication | Magic link via email → `portal_accounts` bridge table |
| ID type | `bigint` (matches CRM schema) |
| Timezone | UTC stored, Asia/Singapore (UTC+8) displayed |
| Multi-child UX | Family overview dashboard with learner cards |
| Credit logic | Package-aware: Platinum = none, Silver/Gold = track credits |
| Payments | `payment_status_extended` + `proof_url` columns extend CRM `payments` table |
| Broadcasts | Supabase Realtime + `broadcast_reads` junction for per-family unread state (no external pub/sub needed) |
| Support escalation | WATI API (WhatsApp) — message sent to parent's phone, coach responds from WATI/coach portal |
| Entity naming | `learners` (not children), `contacts` (not parents), `family_id` as root |
| Admin tools | `/admin/broadcasts` and `/admin/payments` pages, protected by email allowlist |
| Deployment | Vercel (staging + production as separate projects) |
| Schema freeze | End of Day 6 — no schema changes without approval after this point |

## Open Questions (Resolve in Week 0)

| Question | Who decides | Blocks |
|---|---|---|
| Credit expiry policy | Ops + Management | Phase 8 |
| Receipt generation: auto view vs PDF upload | Ops | Phase 14 |
| PayNow QR: static image vs dynamic generation | Ops + Dev | Phase 9 |
| Admin broadcast creation: portal form (recommended) vs Studio insert | Ops + PM | Phase 11, Day 16 |
| Admin payment verification: portal page (recommended) vs Studio SQL | Ops + PM | Phase 9, Day 16 |
| `portal_accounts` creation SOP: manual Studio vs automated trigger | Ops + CRM team | Post-launch ops |
| PDPA consent copy | Management + Legal | Phase 2 (auth go-live) |
| Test booking: parent-initiated vs admin-assigned | Ops | Phase 10 scope |
