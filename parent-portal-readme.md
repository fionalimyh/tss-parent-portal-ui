# The Swim Starter Parent Portal — README

## 1. Project Overview

The Swim Starter Parent Portal is a **mobile web application** designed for parents to manage their child’s swim journey in one place.

The portal should allow parents to view class schedules, attendance, swim progress, test levels, replacement credits, payment status, receipts, PayNow QR payment instructions, and support options.

This project should be designed as a **mobile-first parent experience**, optimised for parents who mainly access links from WhatsApp, SMS, email, or browser.

---

## 2. Confirmed Product Decisions

### 2.1 Platform

The Parent Portal will be built as a **mobile web application first**.

Native iOS and Android apps are **not part of the MVP**.

Reason:
- Faster to build and launch
- Easier for parents to access without downloading an app
- Can be opened directly from WhatsApp links
- Easier to test with current customers
- Can later evolve into a Progressive Web App or native app if adoption is strong

---

### 2.2 Backend — Confirmed: Supabase

The backend is **Supabase**. This is confirmed and no longer a placeholder.

The parent portal reads and writes to the **same Supabase database as the CRM**. It does not have a separate database. All CRM tables (`families`, `contacts`, `learners`, `pools`, `coaches`, `enrollments`, `invoices`, `payments`, etc.) are shared.

Key Supabase capabilities used by the parent portal:

| Capability | Usage |
|---|---|
| **Supabase Auth** | Magic link via email — parents log in with a one-click link sent to their email |
| **Row Level Security (RLS)** | All tables restrict reads to `family_id = get_family_id()` — parents only see their own data |
| **Supabase Realtime** | Broadcasts/Inbox — parents receive new alerts instantly without refreshing |
| **Supabase Storage** | Payment proof uploads and PayNow QR downloads via signed URLs |
| **Edge Functions** | Slack escalation webhook, PayNow QR generation |

A **`portal_accounts` bridge table** links Supabase Auth users to CRM `contacts` records, because the CRM `contacts` table is phone-first and has no email field.

---

### 2.3 Payment Flow

Parents do **not** need to make direct in-app card payments in the MVP.

Instead, the MVP payment flow should support:

1. Parent views outstanding payment status
2. Parent taps **Download PayNow QR**
3. Parent scans the PayNow UEN QR using their bank app
4. Parent may upload proof of payment if required
5. Admin/backend verifies payment
6. Portal updates payment status

The system should support the following payment statuses:

| Status | Meaning |
|---|---|
| Pending | Payment not yet made or not detected |
| Submitted | Parent has submitted proof of payment |
| Verified | Admin/backend has confirmed payment |
| Paid | Payment completed |
| Overdue | Payment deadline has passed |
| Failed / Rejected | Payment proof or transaction could not be verified |

---

### 2.4 Replacement Credit Logic

Parents receive replacement credits when they miss an eligible class.

**Credit eligibility depends on the learner's package type:**

| Package | Credit behaviour |
|---|---|
| **Silver** | 1 credit deducted per week regardless of attendance. Replacement credit generated when class is **cancelled by the school** (weather, coach absence, pool closure) — not for parent-initiated absences. |
| **Gold** | Credit only deducted when child attends. If child misses, no credit is lost — but a replacement credit is still generated if the learner wishes to make up the missed class. |
| **Platinum** | Unlimited classes — no credit tracking at all. No replacement credits applicable. |

The replacement credits module in the portal must be **package-aware**. Platinum learners should see no replacement credits section. The ops team assigns the package via `enrollments.package_id`.

There is currently **no fixed maximum or minimum** number of replacement credits per 8-week term for Silver and Gold packages.

A replacement credit should be created when a missed class is eligible for replacement.

Suggested fields:

| Field | Purpose |
|---|---|
| replacement_credit_id | Unique ID for the credit |
| child_id | Links the credit to the child |
| parent_id | Links the credit to the parent account |
| missed_class_date | Original class date that was missed |
| missed_class_id | Original class session ID |
| credit_status | Available, Used, Expired, Cancelled |
| replacement_class_id | Filled when the credit is used |
| replacement_class_date | Date of the booked replacement class |
| reason | Optional reason for missed class |
| created_at | Credit creation timestamp |
| updated_at | Last updated timestamp |

Suggested replacement credit statuses:

| Status | Meaning |
|---|---|
| Available | Parent can use this credit |
| Used | Credit has been used for a replacement class |
| Expired | Credit is no longer valid |
| Cancelled | Credit was manually removed or voided |

---

### 2.5 Jamie AI Assistant

Jamie should **not connect to live parent, class, payment, or progression data from day one**.

For the MVP, Jamie should begin as a controlled FAQ and support assistant.

Jamie can answer general questions such as:
- How replacement classes work
- How to download PayNow QR
- How to check payment status
- How to read swim progress
- What the different test levels mean
- How to escalate to a human
- Where to find class schedule information

Jamie should **not answer live account-specific questions** until the backend source of truth is confirmed and reliable APIs are available.

Examples of live-data questions to defer until later:
- “How many replacement credits do I have?”
- “Has my payment been verified?”
- “When is my child’s next class?”
- “What level is my child currently at?”
- “Can I book a replacement class for this Saturday?”

Recommended MVP behaviour:

> Jamie can guide parents to the correct portal section, explain policies, and escalate to the team when account-specific data is needed.

---

### 2.6 Escalation Workflow

Escalated parent support tickets are sent to the **WATI WhatsApp Business inbox** (contact number 6582221357), which is integrated with the coach portal. Coaches see the message in WATI and can reply directly — the parent receives the reply on their personal WhatsApp.

The escalation is sent via the WATI API from the portal backend (server-side only — API key never exposed to the browser).

Escalation message sent to parent’s WhatsApp via WATI:

```text
[TSS Support] {{parent_name}} sent a message via the parent portal:

Issue: {{issue_type}}
Message: {{message}}
Page: {{portal_page}}
Urgency: {{urgency}}

Reply here to respond directly to this parent on WhatsApp.
```

The parent’s WhatsApp number is sourced from `contacts.contact_phone` (resolved via `portal_accounts → contacts`).

---

## 3. MVP Goal

The MVP goal is to give parents a simple mobile-friendly portal where they can self-serve the most common post-enrolment information without needing to message customer service for every question.

The MVP should reduce manual parent enquiries related to:
- Class schedule
- Attendance
- Replacement credits
- Payment status
- Receipts
- Progress updates
- Test level status
- Basic FAQs

---

## 4. Primary Users

### 4.1 Parent

Parents use the portal to:
- View their child’s swim journey
- Check upcoming classes
- View attendance
- Track progress
- View replacement credits
- Check payment status
- Download PayNow QR
- View receipts
- Ask Jamie for help
- Escalate issues to the team

### 4.2 Admin / Operations Team

The internal team uses backend tools to:
- Maintain parent and child records
- Update schedules
- Verify payments
- Manage replacement credits
- Track test progression
- Respond to escalated Slack tickets

### 4.3 Coach

Coaches may indirectly provide data through the backend system, such as:
- Attendance
- Skill progress
- Test readiness
- Progress notes

Coach-facing features are **not required in the parent portal MVP** unless later added as a separate module.

---

## 5. Core MVP Modules

## 5.1 Authentication / Parent Access

Parents log in via **magic link email**. This is confirmed. Supabase Auth handles the full flow.

### Login Flow

1. Parent enters their email address on the portal login screen
2. Supabase sends a one-click magic link to their inbox
3. Parent taps the link — opens portal in browser, session established
4. Portal looks up `portal_accounts` using `auth.uid()` to resolve `contact_id` and `family_id`
5. Parent lands on the family dashboard

### Bridge Table: `portal_accounts`

The CRM `contacts` table is phone/WhatsApp-first and has no email field. A bridge table connects Supabase Auth to the CRM:

| Field | Description |
|---|---|
| id | Portal account ID |
| contact_id | Links to CRM `contacts.contact_id` |
| auth_user_id | Supabase Auth `auth.users.id` (UUID) |
| email | Email used for magic link login |
| created_at | Account creation timestamp |

The ops team creates a `portal_accounts` row for each parent at enrolment using their registered email.

### Session Behaviour

- Magic link tokens expire after **1 hour**
- Session persists on device for **7 days** (Supabase Auth default)
- Parents stay logged in between visits
- Expired WhatsApp/email link → redirect to login screen to request a new link

---

## 5.2 Parent Dashboard

The dashboard should be the first screen after login.

It should show:
- Child name
- Current level
- Next class date/time/location
- Attendance summary
- Replacement credit count
- Payment status
- Upcoming test status, if any
- Quick actions

Suggested quick actions:
- View Schedule
- View Progress
- View Replacement Credits
- Download PayNow QR
- Ask Jamie
- Contact Support

---

## 5.3 Learner Profile

Each family may have one or more learners (swimmers). The CRM calls them **learners** — the portal uses the same term.

The learner profile should show:
- Learner name
- Age (calculated from date of birth)
- Current swim level
- Current pool/location
- Class day and timeslot
- Coach name
- Enrolment status
- Progress summary

Learner fields from CRM (`learners` table):

| Field | Description |
|---|---|
| learner_id | CRM `learners.learner_id` (bigint) |
| learner_name | Learner’s name |
| date_of_birth | Used to calculate age |
| learner_level | Current level — see full level sequence in Section 5.6 |
| enrollment_status | From CRM — see values below |

Enrollment status values (must match CRM `learners.enrollment_status` exactly):

| Status | Meaning |
|---|---|
| Yet to Enroll | Learner has not started a class |
| Pending Enrollment | Enrollment in progress |
| Successful Enrollment | Actively enrolled and attending |
| Paused | Enrolment temporarily paused |
| Failed Enrollment | Enrollment did not proceed |
| Future Enrollment | Committed to enrol at a future date |
| Quit | Learner has left |

Class details come from the `enrollments` table:

| Field | Source |
|---|---|
| pool_id | `enrollments.pool_id` → `pools.pool_name` |
| coach_id | `enrollments.coach_id` → `coaches.coach_name` |
| class_day | `enrollments.class_day` |
| class_timeslot | `enrollments.class_timeslot` |
| package | `enrollments.package_id` → `packages.package_name` (Silver / Gold / Platinum) |

---

## 5.4 Class Schedule

Parents should be able to view upcoming classes.

The schedule page should show:
- Date
- Day
- Time
- Location
- Coach
- Class status

Suggested class statuses:

| Status | Meaning |
|---|---|
| Upcoming | Class is scheduled |
| Completed | Class has ended |
| Missed | Child did not attend |
| Cancelled | Class was cancelled |
| Replacement Booked | This is a replacement class |

---

## 5.5 Attendance

Parents should be able to view past attendance.

Attendance records should show:
- Date
- Class time
- Location
- Attendance status
- Replacement credit generated, if any

Suggested attendance statuses:

| Status | Meaning |
|---|---|
| Present | Child attended |
| Absent | Child missed class |
| Cancelled | Class cancelled by company/weather/other reason |
| Replacement Used | Child attended using a replacement credit |

---

## 5.6 Progress Tracking

Parents should be able to see their child’s swim progress.

Progress should be shown in a parent-friendly way, not as an internal coach-only technical checklist.

Suggested progress sections:
- Current level
- Skills completed
- Skills in progress
- Coach notes
- Next milestone
- Test readiness

Test levels:

| Type | Levels |
|---|---|
| Internal Test | 25m, 200m, 400m, 1000m, 1500m |
| SwimSafer National Test | Stage 1, Stage 2, Stage 3, Stage 4 (Bronze), Stage 5 (Silver), Stage 6 (Gold) |

Full level sequence (matches CRM `learners.learner_level` exactly):

1. **Kinder** ← entry level for young beginners
2. 25m
3. Stage 1
4. Stage 2
5. 200m
6. Stage 3
7. 400m
8. Stage 4 (Bronze)
9. 1000m
10. Stage 5 (Silver)
11. 1500m
12. Stage 6 (Gold)

---

## 5.7 Replacement Credits

Parents should be able to view available, used, expired, and cancelled replacement credits.

The page should show:
- Total available credits
- Missed class date
- Credit status
- Replacement booking status
- Expiry date, if applicable
- Used replacement class date, if applicable

For MVP, parents may only view credits. Booking replacement classes can be added later depending on operational readiness.

Suggested MVP scope:

| Feature | MVP? |
|---|---|
| View available credits | Yes |
| View used credits | Yes |
| View missed class linked to credit | Yes |
| Book replacement class directly | Later phase |
| Cancel replacement class | Later phase |

---

## 5.8 Payment Status and PayNow QR

Parents should be able to view payment status and download PayNow QR.

The page should show:
- Outstanding amount
- Due date
- Payment status
- Invoice/reference number
- Download PayNow QR button
- Payment instructions
- Receipt history

Suggested PayNow instruction copy:

```text
1. Tap Download PayNow QR.
2. Open your bank app.
3. Scan or upload the QR code.
4. Complete payment using PayNow.
5. Keep a screenshot of your payment confirmation.
```

Optional MVP feature:
- Upload proof of payment

---

## 5.9 Receipts / Payment History

Parents should be able to view past payment records.

Receipt fields:

| Field | Description |
|---|---|
| receipt_id | Unique receipt ID |
| invoice_id | Linked invoice |
| amount_paid | Amount paid |
| payment_date | Date paid |
| payment_method | PayNow, bank transfer, other |
| payment_status | Paid, Verified, Refunded, Failed |
| receipt_url | Downloadable receipt link |

---

## 5.10 Test Booking / Test Status

Parents should be able to view test information.

MVP test functions:
- View current test eligibility
- View upcoming test, if assigned
- View test level
- View test status
- View test result

Suggested test statuses:

| Status | Meaning |
|---|---|
| Not Eligible | Child is not ready yet |
| Eligible | Child is ready to be considered for test |
| Pending Booking | Test not booked yet |
| Booked | Test has been booked |
| Completed | Test has been completed |
| Passed | Child passed |
| Needs Retry | Child did not pass yet |

---

## 5.11 Jamie Help Assistant

Jamie should appear as a help assistant inside the portal.

MVP Jamie functions:
- Answer general FAQ
- Explain portal pages
- Explain policies
- Guide parent to correct action
- Escalate via WATI (WhatsApp) when needed

Jamie should avoid:
- Guessing live account data
- Confirming payment unless connected to verified source
- Confirming class changes unless connected to schedule source
- Giving progression/test updates unless connected to official progress records

Fallback response for live data:

```text
I can help explain how this works, but I’m unable to confirm live account details yet. I’ll escalate this to our team so they can check for you.
```

---

## 5.12 Support Escalation

Parents should be able to escalate issues from any major page.

Escalation categories:
- Payment issue
- Replacement class issue
- Schedule issue
- Progress/test issue
- Login/access issue
- General enquiry

Escalation destination:

> WATI WhatsApp Business inbox (6582221357) — integrated with coach portal. Coaches reply directly from WATI; parent receives reply on their personal WhatsApp.

---

## 5.13 Inbox / Broadcasts

Parents should be able to receive and view operational broadcasts sent by the admin team.

This is a **read-only inbox** — parents view messages, they do not reply through this module.

The inbox should be accessible from the dashboard (with an unread badge) and as a dedicated page in the navigation.

### Broadcast Types

| Type | Icon | Examples |
|---|---|---|
| Lightning Alert | ⚡ | Lightning detected, class suspended per NEA/Singapore Sports Council policy |
| Bad Weather | 🌧️ | Heavy rain, outdoor pool unsafe, class cancelled or relocated |
| Coach Absence | 👤 | Assigned coach unavailable, replacement coach confirmed or class cancelled |
| Pool Closure | 🏊 | Pool under maintenance, venue closed by facility operator |
| Class Cancellation | ❌ | Specific class cancelled, reason attached |
| Schedule Change | 📅 | Class time or location change for upcoming session(s) |
| General Announcement | 📢 | Holiday schedule, term dates, promotions, general notices |
| Test Announcement | 🎖️ | Upcoming test dates, test schedule broadcast |

### Inbox Display

Each broadcast message should show:
- Broadcast type and icon
- Title
- Message body
- Date and time sent
- Affected children or classes, if applicable
- Read / Unread status

### Dashboard Badge

The parent dashboard should show an unread count badge on the Inbox quick action if there are unread broadcasts.

### Suggested Broadcast Fields

| Field | Description |
|---|---|
| broadcast_id | Unique broadcast ID |
| type | Lightning, BadWeather, CoachAbsence, PoolClosure, ClassCancellation, ScheduleChange, Announcement, TestAnnouncement |
| title | Short broadcast title |
| body | Full message body |
| affected_class_ids | List of affected class session IDs, if applicable |
| affected_child_ids | List of affected child IDs, if targeted |
| is_global | True if sent to all parents, false if targeted |
| sent_at | Timestamp when broadcast was sent |
| created_by | Admin user who created the broadcast |
| expires_at | Optional expiry timestamp |

### Suggested Broadcast Statuses (Parent View)

| Status | Meaning |
|---|---|
| Unread | Parent has not opened the message |
| Read | Parent has opened the message |
| Expired | Broadcast is past its expiry date |

### MVP Scope

| Feature | MVP? |
|---|---|
| View inbox with all broadcasts | Yes |
| Unread badge on dashboard | Yes |
| Mark as read on open | Yes |
| Filter by broadcast type | Later phase |
| Push notification to device | Later phase |
| WhatsApp broadcast integration | Later phase |
| Admin broadcast creation UI | Later phase (admin tool) |

### Suggested API Endpoint

```text
GET /api/broadcasts
GET /api/broadcasts/:broadcast_id
PATCH /api/broadcasts/:broadcast_id/read
```

---

## 6. Suggested Information Architecture

```text
Parent Portal
├── Login / Magic Link
├── Dashboard (with Inbox unread badge)
├── Child Profile
├── Class Schedule
├── Attendance
├── Progress
│   ├── Current Level
│   ├── Skills Completed
│   ├── Skills In Progress
│   └── Test Readiness
├── Replacement Credits
├── Payments
│   ├── Payment Status
│   ├── Download PayNow QR
│   └── Receipts
├── Tests
│   ├── Eligibility
│   ├── Booking Status
│   └── Results
├── Inbox / Broadcasts
│   ├── Lightning Alerts
│   ├── Bad Weather
│   ├── Coach Absence
│   ├── Pool Closure
│   ├── Class Cancellations
│   ├── Schedule Changes
│   └── General Announcements
├── Jamie Assistant
└── Support Escalation
```

---

## 7. Recommended User Flow

### 7.1 Parent Login Flow

```text
Parent receives portal link via WhatsApp
→ Parent taps link
→ Parent verifies identity through magic link / OTP
→ Parent lands on dashboard
```

### 7.2 Payment Flow

```text
Parent opens Payments page
→ Views outstanding payment
→ Downloads PayNow QR
→ Opens bank app
→ Scans QR
→ Makes payment
→ Uploads proof if required
→ Payment status becomes Submitted / Verified / Paid
```

### 7.3 Replacement Credit Flow

```text
Child misses eligible class
→ Replacement credit is created
→ Parent sees credit in portal
→ Credit status shows Available
→ Credit is used later when replacement class is booked
→ Credit status becomes Used
```

### 7.4 Jamie Escalation Flow

```text
Parent asks Jamie a question
→ Jamie answers if it is general FAQ
→ Jamie escalates if live account data is required
→ Slack message is created for internal team
→ Team follows up with parent
```

---

## 8. Design Principles

The portal should feel:
- Mobile-first
- Clean
- Friendly
- Parent-friendly
- Fast to load
- Easy to navigate
- Low-friction
- Clear on next action

Design should prioritise:
- Large tap targets
- Clear cards
- Minimal text overload
- Simple labels
- Clear statuses
- WhatsApp-friendly entry points
- Parent-friendly language

---

## 9. Technical Direction

### 9.1 Frontend

Confirmed tech stack:

| Layer | Technology |
|---|---|
| Framework | **Next.js 14+** (App Router) |
| Language | **TypeScript** |
| Styling | **Tailwind CSS** |
| Components | **shadcn/ui** |
| Data fetching | **Supabase JS** (server components + client hooks) |
| State | React built-ins — no external state library needed for MVP |

Structure:
- **Server Components** for initial page data (SSR — fast first load, good for WhatsApp link entry)
- **Client Components** for interactive UI (form inputs, realtime subscriptions)
- **Route Groups**: `(auth)` for login pages, `(portal)` for all authenticated pages

### 9.2 Backend — Supabase

The backend is **Supabase** (confirmed). The portal reads the same database as the CRM.

| Capability | How it is used |
|---|---|
| **Auth (magic link)** | `supabase.auth.signInWithOtp({ email })` — sends magic link |
| **RLS** | `get_family_id()` helper function enforces per-family data isolation |
| **Realtime** | `.channel('broadcasts').on('postgres_changes', ...)` — live inbox updates |
| **Storage** | `payment-proofs` bucket (private, signed URLs) for PayNow upload |
| **Edge Functions** | `send-wati-escalation`, `generate-paynow-qr` |

**RLS helper (deployed once to Supabase):**
```sql
CREATE OR REPLACE FUNCTION get_family_id()
RETURNS bigint LANGUAGE sql STABLE SECURITY DEFINER AS $$
  SELECT c.family_id
  FROM portal_accounts pa
  JOIN contacts c ON c.contact_id = pa.contact_id
  WHERE pa.auth_user_id = auth.uid()
$$;
```

**Timezone:** All `timestamptz` stored in UTC. All displayed times converted to **Asia/Singapore (UTC+8)**.

### 9.3 API Design

The portal uses **Supabase JS directly** from Next.js — no custom REST API layer. Server Components query Supabase server-side using the service role client. Client Components use the anon key client for real-time subscriptions and mutations.

Next.js API routes (`/api/...`) are used only for server-side operations that must not expose credentials to the browser:



```text
POST /api/support/escalate        ← calls WATI API to send WhatsApp to parent (server-side, hides WATI key)
POST /api/payments/upload-proof   ← uploads to Supabase Storage, updates payment record
GET  /api/payments/paynow-qr      ← generates or retrieves PayNow QR signed URL
```

All other data is fetched directly from Supabase using the JS client in Server or Client Components.

---

## 10. Data Models

All IDs are `bigint` (matching CRM schema). All timestamps are `timestamptz` stored in UTC, displayed in Asia/Singapore (UTC+8). Tables marked **[CRM — read only]** exist in the shared Supabase database and are managed by the CRM or ops team. Tables marked **[Portal — owned]** are net-new tables created for the parent portal.

---

### 10.1 portal_accounts [Portal — owned]

Bridge between Supabase Auth and CRM contacts. Created by ops team at enrolment.

```json
{
  "id": "bigint PK",
  "contact_id": "bigint FK → contacts.contact_id",
  "auth_user_id": "uuid FK → auth.users.id",
  "email": "text UNIQUE",
  "created_at": "timestamptz"
}
```

---

### 10.2 families [CRM — read only]

Root grouping entity. One family per household.

```json
{
  "family_id": "bigint PK",
  "family_name": "text",
  "country": "text (SG / MY / ID)",
  "pipeline_status": "text",
  "created_at": "timestamptz"
}
```

---

### 10.3 contacts [CRM — read only]

Parent or guardian record. One family may have multiple contacts. `is_primary_contact = true` identifies the main communication contact.

```json
{
  "contact_id": "bigint PK",
  "family_id": "bigint FK → families",
  "is_primary_contact": "boolean",
  "contact_name": "text",
  "phone_number": "text",
  "country": "text"
}
```

---

### 10.4 learners [CRM — read only]

One record per swimmer. Belongs to a family, not a specific contact.

```json
{
  "learner_id": "bigint PK",
  "family_id": "bigint FK → families",
  "learner_name": "text",
  "date_of_birth": "date",
  "learner_level": "text (Kinder / 25m / Stage 1 / Stage 2 / 200m / Stage 3 / 400m / Stage 4 (Bronze) / 1000m / Stage 5 (Silver) / 1500m / Stage 6 (Gold))",
  "enrollment_status": "text (Yet to Enroll / Pending Enrollment / Successful Enrollment / Paused / Failed Enrollment / Future Enrollment / Quit)",
  "updated_at": "timestamptz"
}
```

---

### 10.5 pools [CRM — read only]

Swimming pool / venue lookup table.

```json
{
  "pool_id": "bigint PK",
  "pool_name": "text (Bishan / Sengkang / Pasir Ris / Jurong West / Clementi / Bukit Canberra)",
  "country": "text",
  "active": "boolean"
}
```

---

### 10.6 coaches [CRM — read only]

Coach lookup table.

```json
{
  "coach_id": "bigint PK",
  "coach_name": "text (Don / Ben / Sky / Derek / Fabian / Fredy)",
  "country": "text",
  "active": "boolean"
}
```

---

### 10.7 packages [CRM — read only]

Package definitions. Drive replacement credit logic.

```json
{
  "package_id": "bigint PK",
  "package_name": "text (Silver / Gold / Platinum)",
  "credits_per_week": "integer (null for Platinum)",
  "deduct_always": "boolean (true = Silver, false = Gold)",
  "is_unlimited": "boolean (true = Platinum)",
  "active": "boolean"
}
```

---

### 10.8 enrollments [CRM — read only]

One record per learner per enrolment period. Links learner to class schedule, pool, coach, and package.

```json
{
  "enrollment_id": "bigint PK",
  "learner_id": "bigint FK → learners",
  "family_id": "bigint FK → families",
  "enrollment_status": "text",
  "package_id": "bigint FK → packages",
  "class_day": "text",
  "class_timeslot": "text",
  "pool_id": "bigint FK → pools",
  "coach_id": "bigint FK → coaches",
  "enrollment_start_date": "date",
  "enrollment_end_date": "date"
}
```

---

### 10.9 class_sessions [Portal — owned]

Individual class instances. A class session is a **group event** — multiple learners attend it. Not linked to a single child.

```json
{
  "class_session_id": "bigint PK",
  "enrollment_id": "bigint FK → enrollments",
  "pool_id": "bigint FK → pools",
  "coach_id": "bigint FK → coaches",
  "session_date": "date",
  "start_time": "time",
  "end_time": "time",
  "status": "text (Scheduled / Completed / Cancelled)",
  "cancellation_reason": "text",
  "created_at": "timestamptz",
  "updated_at": "timestamptz"
}
```

---

### 10.10 attendance_records [Portal — owned]

Per-learner attendance for each class session.

```json
{
  "attendance_id": "bigint PK",
  "class_session_id": "bigint FK → class_sessions",
  "learner_id": "bigint FK → learners",
  "family_id": "bigint FK → families",
  "status": "text (Present / Absent / CancelledBySchool / ReplacementUsed)",
  "replacement_credit_generated": "boolean",
  "created_at": "timestamptz"
}
```

---

### 10.11 replacement_credits [Portal — owned]

One credit per eligible missed or school-cancelled class. Package-aware — only Silver and Gold learners have credits.

```json
{
  "credit_id": "bigint PK",
  "learner_id": "bigint FK → learners",
  "family_id": "bigint FK → families",
  "source_attendance_id": "bigint FK → attendance_records",
  "status": "text (Available / Used / Expired / Cancelled)",
  "expires_at": "date",
  "used_attendance_id": "bigint FK → attendance_records (null until used)",
  "created_at": "timestamptz",
  "updated_at": "timestamptz"
}
```

---

### 10.12 invoices [CRM — read only]

Family-level billing. One per billing event.

```json
{
  "invoice_id": "bigint PK",
  "family_id": "bigint FK → families",
  "invoice_date": "timestamptz",
  "due_date": "date",
  "period_start": "date",
  "period_end": "date",
  "total_amount": "numeric",
  "currency": "text (SGD)",
  "invoice_status": "text (Unpaid / Paid / Overdue)",
  "notes": "text"
}
```

---

### 10.13 invoice_items [CRM — read only]

Line items per invoice. Each item is attributable to a specific learner.

```json
{
  "item_id": "bigint PK",
  "invoice_id": "bigint FK → invoices",
  "learner_id": "bigint FK → learners",
  "item_type": "text (Package / Equipment / Swim Test / Other)",
  "description": "text",
  "quantity": "integer",
  "unit_price": "numeric",
  "total_price": "numeric"
}
```

---

### 10.14 payments [CRM — extended by portal]

Payment records against invoices. Extended from CRM with `proof_url`, `Submitted`, and `Verified` statuses for the PayNow upload flow.

```json
{
  "payment_id": "bigint PK",
  "invoice_id": "bigint FK → invoices",
  "payment_date": "timestamptz",
  "amount": "numeric",
  "payment_method": "text (PayNow / Bank Transfer / Card)",
  "payment_status": "text (Pending / Submitted / Verified / Completed / Failed)",
  "proof_url": "text (Supabase Storage signed URL — set when parent uploads screenshot)",
  "proof_uploaded_at": "timestamptz",
  "verified_by": "text (admin name)",
  "verified_at": "timestamptz",
  "paynow_qr_url": "text (Supabase Storage URL)",
  "transaction_ref": "text"
}
```

Payment status flow: `Pending` → `Submitted` (proof uploaded) → `Verified` (admin confirms) → `Completed`

---

### 10.15 broadcasts [Portal — owned]

Operational alerts sent by admin. Scoped by pool location or global.

```json
{
  "broadcast_id": "bigint PK",
  "type": "text (Lightning / BadWeather / CoachAbsence / PoolClosure / ClassCancellation / ScheduleChange / Announcement / TestAnnouncement)",
  "title": "text",
  "body": "text",
  "pool_id": "bigint FK → pools (null = all locations)",
  "is_global": "boolean",
  "sent_at": "timestamptz",
  "expires_at": "timestamptz",
  "created_by": "text"
}
```

---

### 10.16 broadcast_reads [Portal — owned]

Junction table tracking which families have read each broadcast.

```json
{
  "id": "bigint PK",
  "broadcast_id": "bigint FK → broadcasts",
  "family_id": "bigint FK → families",
  "read_at": "timestamptz"
}
```

---

### 10.17 support_tickets [Portal — owned]

Escalation records. Each ticket triggers a WATI WhatsApp message to the parent's phone, visible in the coach portal WATI inbox.

```json
{
  "ticket_id": "bigint PK",
  "family_id": "bigint FK → families",
  "learner_id": "bigint FK → learners (optional)",
  "issue_type": "text (Payment / Replacement / Schedule / Progress / Test / Login / General)",
  "message": "text",
  "portal_page": "text",
  "urgency": "text (Normal / High / Critical)",
  "wati_sent": "boolean",
  "wati_message_id": "text (WATI message ID, for reference)",
  "created_at": "timestamptz"
}
```

---

## 11. Non-Goals for MVP

The following are not required for MVP:

- Native iOS app
- Native Android app
- Full in-app card payment gateway
- Live Jamie access to all parent data
- Direct replacement class booking
- Coach-facing app
- Admin dashboard rebuild
- Full CRM replacement
- Complex push notification engine (in-portal inbox is MVP; device push notifications are later phase)
- Advanced analytics dashboard

---

## 12. Future Phase Ideas

Possible future enhancements:
- Progressive Web App installation
- Push notifications
- Live replacement class booking
- Real-time payment reconciliation
- Jamie connected to verified backend data
- Coach progress update interface
- Parent-coach conference booking
- Test registration flow
- Multi-child family dashboard
- Automated WhatsApp reminders
- Admin support dashboard
- CRM-integrated parent timeline

---

## 13. Open Questions to Revisit Later

| Question | Current Status |
|---|---|
| ~~Final backend source of truth~~ | **Resolved — Supabase** |
| ~~Whether to use Airtable, Supabase, Splash, WATI, or CRM~~ | **Resolved — Supabase (shared CRM database)** |
| ~~Auth method~~ | **Resolved — magic link via email** |
| ~~Multi-child UX~~ | **Resolved — family overview dashboard (child cards)** |
| Whether parents can book replacement classes directly | Later phase |
| Whether Jamie can access live data | Later phase |
| Whether payment proof upload is required | **Operationally required for PayNow — should be MVP, not optional** |
| Whether receipts are auto-generated or uploaded manually | To confirm — likely auto-generated from `invoice_items` |
| Whether test booking is parent-initiated or admin-assigned | To confirm — current assumption: admin-assigned |
| Credit expiry policy — when do replacement credits expire? | To confirm — end of term? 30 days? |
| PDPA consent — how is consent collected at first login? | To confirm with legal/ops — required before collecting personal data |
| Admin broadcast creation flow for MVP | To confirm — direct Supabase Studio insert, or simple admin form? |

---

## 14. Build Priority

Recommended MVP build order:

1. Mobile web shell and navigation
2. Parent login / access
3. Dashboard
4. Child profile
5. Schedule
6. Attendance
7. Progress
8. Replacement credits
9. Payment status and PayNow QR download
10. Receipts
11. Test status
12. Inbox / Broadcasts (lightning, bad weather, coach absence, pool closure, announcements)
13. Jamie FAQ assistant
14. WATI WhatsApp escalation

---

## 15. External Services Setup

Everything the parent portal depends on that lives **outside the codebase**. All must be provisioned before Week 1 development begins.

---

### 15.1 Supabase (Database, Auth, Realtime, Storage)

**What it does:** Hosts the shared database, handles magic-link email authentication, delivers real-time broadcasts via WebSocket, and stores payment-proof images.

**Free tier limits:** 500 MB database, 50,000 monthly active users, 1 GB Storage, 2 active Edge Functions, 200 concurrent Realtime connections. Sufficient for MVP.

**Setup steps:**
1. Create account at [supabase.com](https://supabase.com)
2. Create **dev** project: `tss-portal-dev` — note Project URL, Anon Key, Service Role Key
3. Create **prod** project: `tss-portal-prod` — store keys in password manager
4. In each project: Supabase Dashboard → Storage → create private bucket named `payment-proofs`
5. In each project: Authentication → Settings → enable "Email" provider, set Site URL to the portal URL
6. In each project: Authentication → Email Templates → customise magic-link email with TSS branding
7. Realtime is enabled by default — no extra config needed

**Cost:** Free tier recommended at launch. Upgrade to Pro ($25 USD/month) if the team exceeds 50,000 MAU or 500 MB storage.

---

### 15.2 Resend (Transactional Email for Magic Links)

**What it does:** Sends magic-link login emails to parents. Supabase's built-in email is rate-limited (4 emails/hour on free tier) — too slow for a parent launch. Resend provides reliable delivery from a custom domain.

**Free tier limits:** 100 emails/day, 3,000/month. Sufficient for a swim school of under 200 families.

**Setup steps:**
1. Create account at [resend.com](https://resend.com)
2. Add and verify domain: `theswimstarter.com` (add DNS TXT + MX records as instructed)
3. Create an API key → copy it
4. In Supabase Dashboard → Authentication → SMTP Settings → enable custom SMTP:
   - Host: `smtp.resend.com`
   - Port: `465`
   - Username: `resend`
   - Password: *(Resend API key)*
   - Sender email: `noreply@theswimstarter.com`
5. Test: send a magic link from Supabase Auth → confirm email is received

**Cost:** Free tier sufficient at launch. Paid plans from $20 USD/month if volume exceeds free limits.

---

### 15.3 WATI (WhatsApp Business API — Support Escalations)

**What it does:** Receives parent support messages from the portal and delivers them as WhatsApp messages into the WATI inbox, where coaches respond directly. No Slack required. Parent receives the reply on their personal WhatsApp.

**Already in use:** The coach portal is already connected to WATI under number **6582221357**. No new WATI account is needed — only API credentials.

**Setup steps:**
1. Log in to WATI Dashboard
2. Navigate to: Settings → API → copy **API Key** (Bearer token)
3. Note the **API Base URL** (format: `https://live-mt-server.wati.io/{account_id}`)
4. Add both values to `.env`: `WATI_API_ENDPOINT` and `WATI_API_KEY`
5. Test via curl or Postman:
   ```bash
   curl -X POST "{WATI_API_ENDPOINT}/api/v1/sendSessionMessage/{test_phone_number}" \
     -H "Authorization: Bearer {WATI_API_KEY}" \
     -H "Content-Type: application/json" \
     -d '{"messageText": "Test from parent portal setup"}'
   ```
6. Confirm message appears in WATI coach portal inbox

**Cost:** Already subscribed. No additional cost.

---

### 15.4 Vercel (Hosting)

**What it does:** Hosts the Next.js application. Provides automatic deployments from git, preview deployments for PRs, and separate staging/production environments.

**Free tier limits:** 100 GB bandwidth/month, unlimited deployments, 1 concurrent build. Sufficient for a small swim school.

**Setup steps:**
1. Create account at [vercel.com](https://vercel.com)
2. Import the GitHub repository
3. Create **staging** project: `tss-portal-staging` → add dev Supabase env vars
4. Create **production** project: `tss-portal-prod` → add prod Supabase env vars + prod WATI key
5. Configure custom domain in production project settings (e.g., `portal.theswimstarter.com`)
6. Add DNS CNAME record at domain registrar pointing to Vercel

**Cost:** Hobby (free) tier sufficient at launch. Pro ($20 USD/month) only needed if team needs more concurrent builds or advanced analytics.

---

### 15.5 Summary — Monthly Cost at Launch

| Service | Tier | Monthly Cost |
|---|---|---|
| Supabase | Free | $0 |
| Resend | Free | $0 |
| WATI | Already subscribed | $0 additional |
| Vercel | Hobby (Free) | $0 |
| **Total** | | **$0/month at launch** |

Upgrade triggers:
- Supabase Pro ($25/mo): if DB exceeds 500 MB or MAU exceeds 50,000
- Resend paid ($20/mo): if email volume exceeds 3,000/month
- Vercel Pro ($20/mo): if build concurrency or bandwidth limits are hit

---

### 15.6 Pub/Sub — How Real-Time Works (No Extra Service Needed)

Supabase Realtime is the pub/sub layer for broadcasts. It uses PostgreSQL's `LISTEN/NOTIFY` and WebSockets — no separate Redis, RabbitMQ, or Pub/Sub service is needed.

**How it works:**
1. Admin inserts a row into the `broadcasts` table
2. Supabase Realtime detects the INSERT via Postgres change data capture
3. It publishes the event to all subscribed WebSocket clients (parents with the portal open)
4. The parent's browser receives the new broadcast instantly — no page refresh

**Portal code (client-side subscription):**
```ts
supabase
  .channel('broadcasts')
  .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'broadcasts' }, (payload) => {
    // prepend new broadcast to list
  })
  .subscribe()
```

No external pub/sub account or configuration required — Supabase Realtime is included in all Supabase tiers.

---

## 16. Summary

The Parent Portal should be built as a **mobile web-first MVP** that gives parents easy access to the most important post-enrolment information.

The portal should focus on clarity, self-service, and operational efficiency. It should avoid overbuilding full app/payment/live AI infrastructure too early.

Confirmed MVP direction:

- Mobile web first (Next.js 14, TypeScript, Tailwind CSS, shadcn/ui)
- Backend: **Supabase** — shared database with CRM, no separate database
- Auth: **Magic link via email** via Supabase Auth + `portal_accounts` bridge table
- Family overview dashboard with learner cards (not single-child view)
- Entity naming aligned to CRM: **learners** (not children), **contacts** (not parents), **family_id** as root
- Level sequence: Kinder → 25m → Stage 1 → Stage 2 → 200m → Stage 3 → 400m → Stage 4 (Bronze) → 1000m → Stage 5 (Silver) → 1500m → Stage 6 (Gold)
- Enrollment statuses: match CRM exactly (Successful Enrollment, Paused, Quit, etc.)
- Replacement credits are **package-aware**: Platinum = no credits; Silver/Gold = credits apply
- PayNow UEN QR download + **proof of payment upload** (operationally required)
- Payments: `proof_url` field and `Submitted`/`Verified` statuses extend the CRM `payments` table
- Inbox/Broadcasts: Supabase Realtime for live delivery, `broadcast_reads` junction for unread tracking, scoped by `pool_id`
- No fixed cap on replacement credits for Silver/Gold
- Jamie starts as FAQ/support assistant, not live-data assistant
- Escalations go to WATI WhatsApp (6582221357) via Next.js API route — coach responds from WATI/coach portal
- PDPA consent notice at first login (required — to be confirmed with ops)

