# TSS Parent Portal — Project Rules

## Session Start (REQUIRED every time)

Before doing anything else at the start of every session on this project:

1. Read `handover.md` in full
2. Read `parent-portal-readme.md` in full
3. Report to the user:
   - Where we are in the build plan (last completed phase/task)
   - What the next step is
   - Any blockers or open questions that need a decision before proceeding

Do not skip this even if the user jumps straight to a task. Read first, then confirm the next step.

---

## Plan Review Rule

`handover.md` is the single source of truth for the build plan.

- Before starting any work, check whether the task is covered in `handover.md`
- If new information, decisions, or scope changes affect the plan, **pause and propose the revision** to the user before making any changes to `handover.md`
- Any change to `handover.md` requires explicit user approval — do not edit it without asking first
- After approval, update `handover.md` to reflect the change, then proceed

---

## Data Model Rule

`parent-portal-readme.md` Section 10 is the source of truth for all data models.

- Do not invent table names, column names, or status values — check the README first
- Entity naming: `learners` (not children), `contacts` (not parents), `family_id` as root grouping key
- IDs are `bigint` (not UUID, not string)
- Do NOT modify `2026-05-11-crm-schema-design.md` — it is a read-only reference document

---

## Approval Gates

These actions require the user to explicitly say "yes" or "approved" before proceeding:

- Any edit to `handover.md`
- Any edit to `parent-portal-readme.md`
- Dropping or altering existing Supabase tables
- Pushing to any remote repository
- Deploying to Vercel or any other host

---

## Secrets & Security Rule

All secrets, tokens, API keys, and credentials must be managed as follows:

- **Store in `.env` only.** Never hardcode any secret value in any source file, markdown document, or config file.
- **`.env` must never be committed.** It is NOT auto-ignored by Next.js — `.env` must be the first entry in `.gitignore` before `git init` is run.
- **`.env` is the single secrets file.** It lives at the repo root. Update it whenever a new environment variable is added, and keep the placeholder structure intact so it also serves as the template.
- **`SUPABASE_SERVICE_ROLE_KEY` is server-side only.** It must never appear in any file imported by a client component or exposed via `NEXT_PUBLIC_*`.
- **Before generating any code** that references a credential, use the variable name from `.env.example` — never invent a value.
- **Before committing or reviewing diffs**, scan for patterns that indicate leaked secrets: `eyJ` (JWT), `supabase.co` (project URL), `hooks.slack.com` (webhook), any string that looks like a key or token.
- **If a secret is ever found hardcoded**, treat it as compromised: rotate it immediately, then remove it from all files.

Current variables (see `.env.example` for full list):
- `NEXT_PUBLIC_SUPABASE_URL` — Supabase project URL
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — Supabase anon/public key (safe for browser)
- `SUPABASE_SERVICE_ROLE_KEY` — Supabase service role key (server only)
- `WATI_API_ENDPOINT` — WATI API base URL for WhatsApp support escalations (server only)
- `WATI_API_KEY` — WATI Bearer token (server only)
- `ADMIN_EMAILS` — comma-separated admin email list for `/admin/*` route protection

**`.gitignore` reminder:** When initialising git, ensure `.env` is the very first line of `.gitignore` before any commit is made.
