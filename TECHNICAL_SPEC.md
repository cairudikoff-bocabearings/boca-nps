# Boca NPS — Technical Specification

This is the full technical spec generated directly from the Lovable project, covering exact tech stack, file structure, database schema, server functions, and route behavior. Use this alongside README.md for the actual migration work — README.md covers what's done/pending/decisions-needed, this document covers exact implementation detail.

---

## 1. Product Overview

A small internal web app that lets Boca Bearings staff:
- Sign in with a Microsoft 365 account.
- Upload a CSV of customers.
- Send each customer a branded one-click NPS email survey (1–10 buttons).
- Collect responses with an optional comment via a public landing page.
- Show a "thanks / please share" page after a response, with social share buttons (only encouraged for promoters/passives).
- Email an internal alert to the owner whenever a detractor (score ≤ 6) responds.
- View an NPS trends dashboard with score distribution and an average-score line chart over multiple timeframes.

Not a multi-tenant SaaS — single Boca Bearings team. Anonymous sign-up disabled.

---

## 2. Tech Stack

- **Framework:** TanStack Start v1 (React 19, Vite 7, file-based routing under `src/routes/`)
- **Language:** TypeScript, strict mode
- **Styling:** Tailwind CSS v4 (via `src/styles.css`, no `tailwind.config.js`), shadcn/ui, tw-animate-css
- **Charts:** recharts
- **Forms/validation:** zod
- **Data fetching:** @tanstack/react-query
- **Backend (Lovable Cloud / Supabase):** Postgres + Auth + RLS
- **Email:** Resend, via Lovable connector gateway (`https://connector-gateway.lovable.dev/resend/emails`). No verified custom domain yet — sender is `onboarding@resend.dev`
- **Auth provider:** Microsoft (Azure AD) via Lovable OAuth broker (`lovable.auth.signInWithOAuth("microsoft", ...)`)
- **Runtime:** Cloudflare Workers (nodejs_compat). All server logic uses TanStack `createServerFn` or server routes — no Supabase Edge Functions.

### Required environment variables

**Server-only (Lovable Cloud injects automatically):**
- `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SERVICE_ROLE_KEY`
- `RESEND_API_KEY` (managed by Resend connector)
- `LOVABLE_API_KEY`

**Client (Vite):**
- `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY`

---

## 3. Branding & Design System

- **Logo:** `src/assets/boca-bearings-logo.png` (re-exported as `logoAsset` JSON; reference `logoAsset.url`)
- **Palette** (dark navy + royal/steel blue), OKLCH tokens in `src/styles.css`:
  - `--background: oklch(0.2 0.03 235)` (dark navy)
  - `--card: oklch(0.27 0.05 235)` (slightly lighter navy)
  - `--primary: oklch(0.62 0.13 232)` (steel/royal blue)
  - `--accent: oklch(0.45 0.17 264)` (deeper royal blue) — main CTAs
  - `--destructive: oklch(0.6 0.22 27)`
  - `--secondary: oklch(0.36 0.07 235)`, `--muted: oklch(0.32 0.05 235)`
- **Font:** Inter (system fallback): `--font-display: "Inter", system-ui, sans-serif`
- **Radius:** 0.5rem base
- **Email palette** (hard-coded in HTML templates): Navy #0f1b2e, Navy soft #1a2942, Accent #2952a3, Steel #3b6fa0, Text #1e293b, Muted #64748b
- **NPS button colors:** green #16a34a (9–10), amber #f59e0b (7–8), red #dc2626 (1–6)
- **Owner alert recipient (hard-coded):** jason@bocabearings.com
- **Components:** standard shadcn/ui set (Button, Card, Input, Textarea, Table, Select, Sonner toaster, etc.)

---

## 4. Database Schema (Supabase / Postgres, public schema)

Two tables, both with RLS enabled.

### `survey_batches`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK default gen_random_uuid() | |
| sent_by | uuid (nullable) | auth.uid() of the staff member who sent the batch |
| sent_at | timestamptz not null default now() | |
| recipient_count | integer not null default 0 | |

Grants: `SELECT, INSERT, UPDATE, DELETE` to `authenticated`; `ALL` to `service_role`.
Policies: INSERT/SELECT (authenticated) both require `auth.uid() = sent_by`. UPDATE/DELETE denied.

### `nps_responses`

| Column | Type | Notes |
|---|---|---|
| id | uuid PK default gen_random_uuid() | |
| batch_id | uuid (nullable) FK → survey_batches(id) | |
| email | text not null | max length 320 |
| customer_name | text | |
| company | text | seed data uses `'__SEED__'` to flag samples |
| area_of_interest | text | |
| score | integer not null | 1–10 |
| comment | text | max length 2000 |
| created_at | timestamptz not null default now() | |

Grants: `SELECT, INSERT` to `anon, authenticated`; `ALL` to `service_role`.

Policies:
- INSERT (anon + authenticated): `batch_id IS NOT NULL AND EXISTS (SELECT 1 FROM survey_batches b WHERE b.id = nps_responses.batch_id) AND score BETWEEN 1 AND 10 AND email IS NOT NULL AND length(email) <= 320 AND (comment IS NULL OR length(comment) <= 2000)`
- SELECT (authenticated): only the batch owner can read — `EXISTS (SELECT 1 FROM survey_batches b WHERE b.id = nps_responses.batch_id AND b.sent_by = auth.uid())`
- UPDATE/DELETE denied.

---

## 5. Authentication

- Single provider: Microsoft (Azure AD) via Lovable OAuth broker. Email/password and Google are off.
- Anonymous sign-ups disabled. No `profiles` or `roles` table — no per-user data beyond `survey_batches.sent_by`.
- Sign-in: `lovable.auth.signInWithOAuth("microsoft", { redirect_uri: window.location.origin })`
- Session lives in Supabase's client (localStorage); `attachSupabaseAuth` registered as a global `functionMiddleware` in `src/start.ts` so every server-fn call carries the bearer token automatically.

### Route protection
- **Public:** `/auth`, `/respond`, `/share`, `/api/public/low-score-alert`
- **Authenticated:** everything under `src/routes/_authenticated/` (dashboard at `/`, trends at `/trends`)
- `src/routes/_authenticated/route.tsx` is `ssr: false`, calls `supabase.auth.getUser()` in `beforeLoad`; unauthenticated users redirect to `/auth`.

---

## 6. Routes & Pages

**`/__root.tsx`** — Root layout. `QueryClient` via `createRootRouteWithContext`, `HeadContent`/`Scripts`, default 404 and error boundary, pulls in `styles.css`.

**`/auth`** (public) — Centered card, logo, single "Sign in with Microsoft" button (inline SVG). Already-signed-in users redirect to `/` on mount. Errors render below button.

**`/_authenticated/` (Dashboard)** — Header: logo, "Boca Bearings / NPS Survey Console", Sign out button. Title "Send NPS Surveys". CSV upload card (parser splits `\r?\n`, header row required, case-insensitive column matching by substring, simple comma split with quote stripping — no embedded-comma support, rows missing email dropped). Preview card with row count and "Send N Surveys" button. Result card: ✓ N sent successfully · ✗ M failed.

**`/_authenticated/trends`** — Title "NPS Trends". Timeframe `<Select>`: Last 24 hours / 7 days / 30 days / 90 days / 12 months / All time / Custom range (custom reveals two date pickers). Five StatCards: Total responses, Avg score, Promoters % (green), Passives % (amber), Detractors % (red). Recharts LineChart of average score per bucket (Y axis fixed 0–10), tooltip shows value (n=count). Bucket granularity: hour (≤2 days), day (≤45 days), week (≤180 days), month (otherwise). Empty state: "No responses in this timeframe."

**`/respond`** (public) — Zod-validated search params: `email`, `score` (1–10), `batch` (uuid). Missing/invalid → "Invalid link" card. Headline by score band (≥9 / 7–8 / ≤6). Optional textarea (max 2000 chars). Submit inserts into `nps_responses` via anon client. If score ≤6, fire-and-forget POST to `/api/public/low-score-alert`. On success, navigate to `/share?score={score}`.

**`/share`** (public) — Optional `score` param. Detractor (≤6): plain thank-you, no share buttons. Promoter (≥9) or Passive (7–8): Facebook button (`facebook.com/bocabearings`, #1877F2), LinkedIn share (`bocabearings.com`), "Copy suggested post" (fixed text, see below).

**`/api/public/low-score-alert`** (server route, POST) — Validates score 1–6, email required. Looks up most recent `customer_name`/`company` for that email via `supabaseAdmin`. Sends HTML email via Resend gateway to `jason@bocabearings.com`. Returns `{ ok: true }`.

---

## 7. Server Functions

### `src/lib/surveys.functions.ts` — `sendSurveys`
`createServerFn({ method: "POST" })` with `.middleware([requireSupabaseAuth])`. Input: `{ contacts: Contact[] }`. Reads `RESEND_API_KEY`/`LOVABLE_API_KEY`, throws if missing. Inserts a `survey_batches` row (`sent_by = context.userId`). POSTs each contact to the Resend gateway with `Authorization: Bearer ${LOVABLE_API_KEY}` and `X-Connection-Api-Key: ${RESEND_API_KEY}`. From: `Boca Bearings <onboarding@resend.dev>`. Email HTML: centered card, navy header, 1–10 colored button grid (5×2), each linking to `${appUrl}/respond?email=...&score=N&batch=${batchId}`. Returns `{ sent, failed, batchId }`.

### `src/lib/trends.functions.ts` — `getTrends`
`createServerFn({ method: "GET" })`, no auth middleware (aggregate stats only). Input: `{ timeframe, start?, end? }`. Timeframes: day/week/month/quarter/year/all/custom. Query selects `score, created_at` filtered by range, buckets by `bucketKey(date, granularity)` (UTC), computes avg score per bucket plus totals and promoter/passive/detractor %. Returns `TrendsData`. NPS bands: Detractor 1–6, Passive 7–8, Promoter 9–10.

---

## 8. Email Templates (HTML, inline-styled)

**Survey email:** 600px white card, navy header (#0f1b2e) with logo and 4px accent (#2952a3) border. Greeting + "On a scale of 1 to 10..." Score grid: 10 buttons, two rows of 5, 44×44px, colored by band. Footer: navy-soft (#1a2942), "bocabearings.com · ISO 9001:2015 Certified".

**Low-score alert:** Subject "Low NPS score: {score}/10 from {customerName}". White card, navy header. Score (red, bold), customer + company, email, comment in left-bordered blockquote.

**Fixed share post text:**
> I just had a great experience with Boca Bearings and wanted to share it. If you need precision bearings for any application these are the people to call. Check them out at bocabearings.com.

---

## 9. File Tree

```
src/
  assets/boca-bearings-logo.png          (+ generated .asset.json)
  components/ui/                          shadcn components: button, card, input,
                                          textarea, table, select, sonner
  integrations/
    lovable/index.ts                     (Lovable OAuth helper)
    supabase/
      client.ts                          (auto-gen)
      client.server.ts                   (auto-gen — service role)
      auth-middleware.ts                 (auto-gen)
      auth-attacher.ts                   (auto-gen)
      types.ts                           (auto-gen)
  lib/
    surveys.functions.ts
    trends.functions.ts
    utils.ts
    error-capture.ts
    error-page.ts
    lovable-error-reporting.ts
  routes/
    __root.tsx
    auth.tsx
    respond.tsx
    share.tsx
    _authenticated/
      route.tsx                          (integration-managed auth gate)
      index.tsx                          (dashboard / send page)
      trends.tsx
    api/public/
      low-score-alert.ts
  router.tsx
  start.ts                               (registers attachSupabaseAuth)
  styles.css                             (Tailwind v4 + OKLCH tokens)
supabase/migrations/                     (table + RLS migration)
```

---

## 10. Migration SQL (canonical)

```sql
-- survey_batches
CREATE TABLE public.survey_batches (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sent_by uuid,
  sent_at timestamptz NOT NULL DEFAULT now(),
  recipient_count integer NOT NULL DEFAULT 0
);
GRANT SELECT, INSERT, UPDATE, DELETE ON public.survey_batches TO authenticated;
GRANT ALL ON public.survey_batches TO service_role;
ALTER TABLE public.survey_batches ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Authenticated can insert batches"
  ON public.survey_batches FOR INSERT TO authenticated
  WITH CHECK (auth.uid() = sent_by);

CREATE POLICY "Owners can view their batches"
  ON public.survey_batches FOR SELECT TO authenticated
  USING (auth.uid() = sent_by);

-- nps_responses
CREATE TABLE public.nps_responses (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  batch_id uuid REFERENCES public.survey_batches(id),
  email text NOT NULL,
  customer_name text,
  company text,
  area_of_interest text,
  score integer NOT NULL,
  comment text,
  created_at timestamptz NOT NULL DEFAULT now()
);
GRANT SELECT, INSERT ON public.nps_responses TO anon, authenticated;
GRANT ALL ON public.nps_responses TO service_role;
ALTER TABLE public.nps_responses ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can submit a response to a valid batch"
  ON public.nps_responses FOR INSERT TO anon, authenticated
  WITH CHECK (
    batch_id IS NOT NULL
    AND EXISTS (SELECT 1 FROM public.survey_batches b WHERE b.id = nps_responses.batch_id)
    AND score BETWEEN 1 AND 10
    AND email IS NOT NULL AND length(email) <= 320
    AND (comment IS NULL OR length(comment) <= 2000)
  );

CREATE POLICY "Batch owners can view responses"
  ON public.nps_responses FOR SELECT TO authenticated
  USING (
    EXISTS (SELECT 1 FROM public.survey_batches b
            WHERE b.id = nps_responses.batch_id
              AND b.sent_by = auth.uid())
  );
```

---

## 11. End-to-End Flow

1. Staff signs in at `/auth` with Microsoft → redirected to `/`.
2. Uploads CSV → preview table → clicks "Send N Surveys".
3. `sendSurveys` inserts a `survey_batches` row and sends one Resend email per contact with embedded 1–10 buttons linking to `/respond?email=...&score=N&batch=...`.
4. Customer clicks a score → `/respond` renders thank-you copy + optional comment box → submit inserts an `nps_responses` row (anon, RLS-checked).
5. If score ≤6, browser also POSTs `/api/public/low-score-alert`, which emails jason@bocabearings.com.
6. Customer redirected to `/share?score=N` — promoters/passives see social share buttons + copyable post; detractors see a plain thank-you.
7. Staff visits `/trends` to view aggregate stats and the average-score line chart.

---

## 12. Out of Scope / Explicitly Not Built

- Business Central API integration to auto-pull the monthly customer list
- Scheduled (monthly cron) survey sends
- Filtering trends by customer type or area of interest *(built, then removed at Jason's request in favor of date-range filtering — see README.md)*
- Verified custom email sender domain (Resend uses shared `onboarding@resend.dev`)
- Per-user roles / admin UI / multi-tenant features
- Sharing the dashboard outside the signed-in team *(open decision — see README.md)*
