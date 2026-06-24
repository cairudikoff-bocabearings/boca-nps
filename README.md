# boca-nps
# Boca Bearings — NPS Survey System

A password-protected web app for sending NPS (Net Promoter Score) surveys to customers, collecting scores/comments, alerting management on detractors, and encouraging promoters to share publicly.

**Status: Phase 1 prototype, built in Lovable, exported to GitHub for migration to a standalone codebase (Claude Code + Vercel).**

---

## What's already built and working

- **Dashboard (`/`)** — auth-gated. Upload a CSV (columns: `CUSTOMER NAME`, `COMPANY`, `EMAIL`, `AREA OF INTEREST`), preview contacts, send surveys via Resend.
- **Sign-in (`/auth`)** — Microsoft 365 SSO via Lovable's managed OAuth (`lovable.auth.signInWithOAuth("microsoft", ...)`). No username/password form.
- **Survey email** — branded template with 10 numbered score buttons (1–10), sent via Resend.
- **Response page (`/respond`)** — reads `email`, `score`, and `batch_id` from the URL. Shows a score-tiered thank-you message (sympathetic for low scores, enthusiastic for high). Optional comment box. Submits to `nps_responses`. Redirects to `/share`.
- **Detractor alert** — backend function (`/api/public/low-score-alert`). Any score ≤6 triggers an automatic email to `jason@bocabearings.com` with customer name, score, and comment.
- **Share page (`/share`)** — score 9–10: strong encouragement + Facebook/LinkedIn buttons + copy-pre-written-post button. Score 7–8: softer ask, same buttons. Score 1–6: no share ask, just an appreciation message.
- **Trends page (`/trends`)** — score-over-time line chart, summary stats (total responses, average score, % promoter/passive/detractor), filterable by timeframe (24 hours, 7 days, 30 days, 90 days, 12 months, all time, or a custom date range). An area-of-interest filter was built earlier but removed at Jason's request in favor of date-based filtering. Currently seeded with sample/placeholder data for demo purposes, tagged with `company = '__SEED__'` — **search for and remove these rows once real data exists.**
- **Database** — two tables: `survey_batches` (batch history: id, sent_by, sent_at, recipient_count) and `nps_responses` (id, batch_id, email, customer_name, company, area_of_interest, score, comment, created_at). Row Level Security is enabled — responses/batches are currently scoped so only the user who sent a batch can view its data (see "Open decision" below).
- **Email sending** — Resend, connected and functional.

---

## What is NOT done yet — required before this goes live

### 1. Resend domain verification
The sending domain (`bocabearings.com`) is **not yet verified** with Resend. Until it is:
- All emails send from Resend's shared test address (`onboarding@resend.dev`), not `nps@bocabearings.com`.
- Resend will **only deliver to the email address tied to the Resend account itself** — it silently rejects sends to any other, unverified recipient. This is a hard Resend platform restriction, not a bug in this app.

**To fix:** log in to the Resend dashboard → Domains → add DNS records for `bocabearings.com` at the domain registrar → wait for verification (5–30 min after records propagate) → update the `FROM` constant (currently pointing at the test address) in the email-sending function to use `nps@bocabearings.com`.

**Note on the API key:** the current `RESEND_API_KEY` is managed through Lovable's connector system. Once `sendSurveys` is rewritten to call Resend directly (see item 4 below), it's worth generating a **fresh** API key directly from the Resend dashboard rather than assuming the Lovable-managed one transfers cleanly — cheap to regenerate, and avoids any ambiguity about where the original key is actually stored once Lovable's connector layer is gone.

### 2. Microsoft / Azure app registration
Real Microsoft 365 SSO login has never been tested end-to-end. Someone with Microsoft 365 admin access on the Boca Bearings tenant — confirm with Jason whether that's him directly or someone else on the IT/admin side — needs to:
1. Go to portal.azure.com → App registrations → New registration.
2. Name: `Boca NPS Survey`. Supported account types: this organization only.
3. Copy the **Application (client) ID** and **Directory (tenant) ID**.
4. Certificates & secrets → New client secret → copy the **Value** immediately (shown once).
5. Hand these three values (Client ID, Tenant ID, Client Secret) to whoever is configuring auth in the new codebase.

**Important:** this Lovable build used Lovable's own managed OAuth system, which does NOT take these values as plain environment variables — it handled Microsoft auth internally, through Lovable's own settings, not through code. **Whoever rebuilds this outside Lovable will need to wire up Microsoft OAuth from scratch** using a standard library (e.g. NextAuth.js, Auth.js, or Microsoft's MSAL libraries) and these three values directly. This is genuinely new implementation work, not a copy-paste of what existed in Lovable.

### 3. Database migration
This project currently runs on **Lovable Cloud's internal database**, not a standalone Supabase project. Lovable Cloud:
- Is not accessible via the normal Supabase dashboard or SQL editor.
- Cannot be exported via `pg_dump` — only CSV export per table.
- Does not carry RLS policies, auth setup, or secrets over automatically to an external Supabase project.

A real, separate Supabase project (`boca-nps`) was created early in this process but has not been used — it is currently empty.

**Before exporting, clean up test artifacts** — run this in Lovable Cloud's database viewer:

```sql
DELETE FROM nps_responses WHERE company = '__SEED__';
DELETE FROM nps_responses WHERE batch_id IN (SELECT id FROM survey_batches WHERE sent_by IS NULL);
DELETE FROM survey_batches WHERE sent_by IS NULL;
```
This removes the fake `/trends` demo data and any rows created during bypass testing (which violate the real `auth.uid() = sent_by` policy and shouldn't exist under normal operation).

**Then, in order:**
1. Recreate the schema (see SQL below) in the real `boca-nps` Supabase project.
2. Export `survey_batches` from Lovable Cloud as CSV (Cloud → Database → table → Download CSV) and import into `boca-nps` **first**.
3. Export `nps_responses` and import it **second** — `nps_responses.batch_id` is a foreign key referencing `survey_batches.id`, so importing out of order will fail.
4. Recreate RLS policies (see "Open decision" below — current policies scope by `sent_by = auth.uid()`).
5. Update environment variables to point at the new Supabase project (`SUPABASE_URL`, `SUPABASE_ANON_KEY`).

```sql
CREATE TABLE survey_batches (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  sent_by TEXT,
  sent_at TIMESTAMPTZ DEFAULT NOW(),
  recipient_count INTEGER
);

CREATE TABLE nps_responses (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  batch_id UUID REFERENCES survey_batches(id),
  email TEXT,
  customer_name TEXT,
  company TEXT,
  area_of_interest TEXT,
  score INTEGER,
  comment TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4. Remove the Lovable connector gateway dependency

`sendSurveys` (in `src/lib/surveys.functions.ts`) currently sends email by POSTing to `https://connector-gateway.lovable.dev/resend/emails`, authenticated with **two** keys: `LOVABLE_API_KEY` (Lovable's own gateway auth) and `RESEND_API_KEY` (passed through as a header). Once outside Lovable, `LOVABLE_API_KEY` and the gateway itself are gone — **this function needs to be rewritten to call Resend's API directly**, not just have an env var swapped out. This is real code, not config:

- Replace the gateway POST with a direct call using Resend's own SDK (`npm install resend`) or a direct `fetch` to `https://api.resend.com/emails`, authenticated with just `RESEND_API_KEY`.
- The email HTML template itself (the 600px branded card, score button grid) doesn't need to change — just how it's sent.
- Same applies to the low-score alert function (`/api/public/low-score-alert`) — it likely uses the same gateway pattern.

### 5. Deployment target — Cloudflare Workers vs. Vercel

This project's current runtime is **Cloudflare Workers** (`nodejs_compat` mode) — not a default assumption Vercel makes. Before deploying to Vercel:
- Confirm whether TanStack Start's Vercel adapter is being used, or whether the Cloudflare-specific server code (in `start.ts` and the server functions) needs adjusting for Vercel's runtime.
- This is worth checking early — it could affect how `createServerFn` and the server routes behave, not just a deployment config tweak.

### 6. Document version

This README and the accompanying TECHNICAL_SPEC.md reflect the state of the project as generated directly from Lovable on **June 22, 2026**. If picking this up later, sanity-check against the actual exported code rather than assuming nothing has drifted.

---

## Open decision — needs a human answer, not a default

**Who can see survey responses?** Currently, RLS scopes visibility so only the person who sent a batch can see its responses — i.e., per-sender, not shared across the team. The original build brief never specified this either way. Confirm with Jason (or whoever owns this project) whether visibility should be:
- **Per-sender** (current behavior), or
- **Shared** — any authenticated team member can see all responses, regardless of who sent the batch

This determines the RLS policy to write when the schema is recreated in the real Supabase project.

---

## Known temporary states from testing — check before assuming bugs

During Phase 1 testing, several security checks were **temporarily bypassed** to allow full end-to-end testing without real Microsoft login:
- Auth check on the survey-send function (`sendSurveys` in `src/lib/surveys.functions.ts`)
- Link/batch validation on `/respond`
- Insert permission on `nps_responses` (policy: "Anyone can submit a response to a valid batch")
- Insert permission on `survey_batches` (policy: "Authenticated can insert batches")

As of this export, these were intended to be reverted — but **verify directly in the code and the RLS policies** (see canonical migration SQL below) rather than assuming. The canonical policies require a valid `batch_id` plus score/email/comment bounds on `nps_responses`, and `auth.uid() = sent_by` on `survey_batches` for both insert and select — if test data exists with `sent_by IS NULL`, that's a sign the bypass was active when it was created, not a sign of a current bug.

If any bypass is still active, do not deploy to production until restored — they were only intended for one local test session.

The `/trends` page also contains **seeded fake sample data**, tagged with `company = '__SEED__'` in `nps_responses`. Find and remove these rows before going live with real customer data:

```sql
DELETE FROM nps_responses WHERE company = '__SEED__';
```

---

## Quick test data, if useful

A 3-row test CSV with safe test emails was used throughout development:

```csv
CUSTOMER NAME,COMPANY,EMAIL,AREA OF INTEREST
Cai Test One,Acme Robotics,cai.rudikoff.bocabearings@gmail.com,Ceramic Bearings
Cai Test Two,Speedway Motorsports,cai.rudikoff.bocabearings@gmail.com,Hybrid Bearings
Cai Test Three,Coastal Drone Co,cai.rudikoff.bocabearings@gmail.com,Stainless Steel Bearings
```

---

## Contacts / questions

Direct build questions to Jason (jason@bocabearings.com). This README reflects the state of the project as of the Lovable → GitHub export; check commit history for anything more recent.
