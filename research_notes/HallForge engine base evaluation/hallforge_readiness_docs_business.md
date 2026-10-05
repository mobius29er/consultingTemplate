# HallForge — readiness, docs and business context

**Scope:** `/home/user/hallforge` read-only (shallow clone of `mobius29er/hallForge`). Evidence is cited as `path:Lx-Ly`. No git commands, no web calls. Date of review: 2026-10-05. The newest dated document in the repo is 2026-09-02; the newest migration is `060_council_money_buckets.sql`, so the code is a little ahead of the docs and the gaps are called out where that matters.

**One-paragraph verdict.** HallForge is a well-documented, well-tested multi-tenant Next.js 16 / Supabase / Cloudflare Workers product whose *Free tier* ("presence": starter site, editor with live preview, calendar, enquiries, media on R2, appearance, navigation) is shipped and verified in production; whose *billing* (Stripe hosted Checkout + portal + webhook + plan tables) is coded and tested but, as of the last runbook, **not switched on** (Stripe account never activated, Worker secrets absent); and whose *money-taking* features (event registration, ticket checkout, store, custom domains) are either a deliberate 503 stub, "built but unreachable" (no entry point, no scheduler), or unbuilt. The repo's own words: "Pre-launch, no customers" (`README.md:L104`), "All four production councils are on Free" (`docs/decisions/2026-09-02-plan-tiers-presence-communication-fundraising.md:L83`). The founder's roadmap is honest and unusually precise about what is and is not done; the risk for an engine reuse is not hidden rot but KofC vocabulary and a long tail of council-only modules that would be dead weight for a storefront.

---

## 0. What is in the repo (inventory)

| Area | Contents | Cited |
| --- | --- | --- |
| `README.md` | Purpose, stack, tenancy model, block pipeline, status paragraph | `README.md:L1-L110` |
| `docs/` (~69 files) | 33 decision records (`docs/decisions/`, July 15 → Sept 2 2026), 6 launch docs (`docs/launch/`), product contract, quality/test contract, 7 loose notes (`ideas.md`, `tools.md`, `uknight.md`, `page-builder-block-set.md`, `publication-pipeline-risks.md`, two concept notes), 1 HTML build ledger + README, 17 council newsletter PDFs + 1 `.htm`, 1 screenshot PNG | `find docs` listing |
| `deliverables/` | July 20 2026 pilot deck (`.pptx`), speaker notes `.md`, contact sheet, 20 slide SVG/PNG previews | `presentation/README.md:L12-L19` |
| `presentation/` | pptxgenjs generator for that deck (own `package.json`, Node 24) | `presentation/package.json` |
| `analysis/` | Two RoastMyPage reports on the incumbent uKnight (score 26/100) | `analysis/roast-uknight.org-CouncilSite--CNO-15 (1).md:L1-L12` |
| `waitlist/` | Separate Cloudflare Worker + D1 + static HTML signup app (see §3) | `waitlist/wrangler.jsonc:L1-L20` |
| `src/` | 719 files; 158,979 lines of TS/TSX incl. tests and the 6,896-line generated `types/database.ts`; 112,014 lines non-test | `wc -l` |
| `supabase/` | 60 numbered migrations (32,442 lines SQL), 39 pgTAP suites, 13 auth email templates, `legacy-reference/` and `rebaseline/` folders | `ls supabase/migrations`, `supabase/tests/database/` |
| `.github/` | `ci.yml` (lint, typecheck, build, OpenNext build; deploy only on `workflow_dispatch`, with a live smoke check) + 9 agent persona files | `.github/workflows/ci.yml` |
| `files/`, `files.zip` | Superseded January 2026 planning corpus (flagged stale by the audit) | `docs/launch/2026-08-18-launch-readiness-audit.md:L67` |

---

## 1. What is shipped and working vs planned, per module

### Takeaway
By the repo's own module ledger (Aug 30 / Sept 1) and the migrations that landed after it, **foundation + presence + content + media + calendar + members roster + volunteers + news + design studio + newsletter-as-print + operator surface + billing plumbing are built**; **registration, Connect payments, store, custom domains, newsletter email, programs ledger, compliance, MFA enforcement and e2e tests are not**. TODO/FIXME markers are not a useful signal here: the codebase has essentially none (the founder records gaps in docs and in long "why" comments instead).

### Findings (cited)

**Module ledger as recorded by the founder** (`docs/launch/2026-08-20-module-build-plan.md:L13-L33`, revised Sept 1):

| Module | Recorded state | Code corroboration |
| --- | --- | --- |
| `identity` | Built; *gap: MFA not enforced* (L15) | `src/server/auth/identity.ts`, `src/app/auth/{callback,confirm}/route.ts` |
| `tenancy` | Built and reachable after two production-only defects (L16) | `src/middleware.ts:L49,L156,L440-L441` rewrites to `/site`; e2e record `docs/launch/2026-08-20-first-end-to-end-verification.md:L14-L30,L55-L66` |
| `access` | Built (L17); roles/sign-ins became self-serve in migration 058 | `supabase/migrations/058_who_can_sign_in.sql:L1-L25` (1,436 lines), `src/app/(dashboard)/dashboard/access/` |
| `entitlements` | "Mechanism built, nothing enforced" (L18) — **now stale**: SQL-level gates in 046, tier map in 055, and `hasFeature` checks in actions | `supabase/migrations/046_plan_gates_honest.sql:L1-L20`; `src/app/actions/manage-events.ts:L98`, `manage-members.ts:L93`, `import-members.ts:L112`, `src/app/api/media/stock-search/route.ts:L56`, `src/app/(print)/dashboard/newsletters/[id]/print/page.tsx:L112`. `featureProcedure` in `src/server/trpc/index.ts:L191-L197` still has **no callers** because the tRPC router is empty (see §5) |
| `orgtype` | Built, two seeded verticals (L19) | `supabase/migrations/010_orgtype_vocabulary.sql:L73-L109` (`kofc` + `generic` rows); `src/modules/orgtype/contracts.ts:L99-L116` |
| `publishing` | Built — items, revisions, review, conflicts, snapshots, renderer (L20) | `src/server/content/compiler.ts` (1,456 lines), `src/components/public/published-page-renderer.tsx` (1,756), `src/modules/content/blocks-v1/schema.ts` (893) |
| `media` | Built on R2, no public bucket URL; gallery; stock search; *gap: storage cap not enforced* (L21) | `wrangler.jsonc:L25-L43`; `src/app/api/media/{upload,library,stock-search,stock-import}/route.ts`; browser-side re-encoding `src/components/media/prepare-image.ts`. Note migration `041_storage_quota_enforced.sql` exists and post-dates that gap note |
| `messaging` | Foundation only: Resend SMTP, 13 auth templates, enquiry notification; *remaining: digests, member email, newsletters* (L22) | `src/server/email/notify-enquiry.ts:L30` reads `RESEND_API_KEY`; `src/server/email/notify-council-invitation.ts` |
| `payments` | **Built but unreachable** — pipeline and workers exist; no entry point, no scheduler (L23) | `src/server/payments/orchestration/checkout.ts:L175` `preparePaidCheckout` has zero callers; `src/server/payments/processing/README.md:L22-L25` "No scheduler is installed by this module"; `wrangler.jsonc` declares no `triggers.crons` |
| `presence` | Built — starter site, header/footer identity, colours and badge (L24) | `src/modules/presence/starter/template.ts`, `src/app/(dashboard)/dashboard/settings/appearance/page.tsx` |
| `onboarding` | Built — ten derived setup tasks (L25) | `src/modules/onboarding/checklist.ts` |
| `calendar` | Built — public calendar, event pages with JSON-LD, admin (L26) | `src/app/(public)/site/events/`, `src/modules/calendar/` |
| `members` | Built — roster and CSV import (L27) | `src/app/(dashboard)/dashboard/members/{import,new,[id]}/page.tsx`, `src/modules/members/import/` |
| `registration` | **Schema only — endpoint is a deliberate 503** (L28) | `src/app/api/registrations/create/route.ts:L14-L27` returns `REGISTRATION_REBUILD_IN_PROGRESS` 503 |
| `volunteers` | Built; *gap: no public sign-up* (L29) | `src/modules/people/volunteers/`, migration 026 |
| `newsletter` | "Nothing" on Sept 1 (L30) — **superseded**: issues as design documents rendered to paged HTML and printed to PDF; **no email send** | `supabase/migrations/054_newsletter_issues.sql:L1-L15`; `src/modules/newsletter/render/NewsletterRenderer.tsx` (1,156 lines); `grep -il mjml|resend src/modules/newsletter` → none; decision defers MJML/email, canvas editor, contributor workflow, archive (`docs/decisions/2026-09-01-newsletter-issue-from-records.md:L13`) |
| `store` | Registered, unbuilt — dials only (L31) | migration `036_store_features.sql`; no store routes under `src/app` |
| `programs` | "Nothing" (L33) — **superseded**: first slice (Faith-in-Action table) shipped in 050; ledger columns (hours, dollars, Form 1728 line codes) still to come | `supabase/migrations/050_programs_faith_in_action.sql`; `docs/decisions/2026-09-01-newsletter-issue-from-records.md:L29-L35` |
| `compliance` | Nothing (L33) | no module directory under `src/modules/` |

**Built after the ledger, documented only in migration headers** (the docs have not caught up):
- Design studio (flyer, banner, custom) on react-konva — `supabase/migrations/042_design_studio.sql:L1-L11`, `049_design_kinds.sql`, `src/components/dashboard/designs/custom-editor.tsx` (860 lines).
- News posts — `045_news_posts.sql`, `src/app/(dashboard)/dashboard/news/`, public `src/app/(public)/site/news/page.tsx`.
- Recognitions, prayer intentions, office rosters — `051`, `052`, `053`.
- Billing subscriptions — `048_billing_subscriptions.sql:L1-L26` ("organizations.plan_key gains its first writer").
- Council money report from the council's own read-only Stripe key, sealed with a master key — `056_council_stripe_reporting.sql:L1-L25`, `057`, `060_council_money_buckets.sql:L1-L25`; `src/server/finance/stripe-report/`, `src/server/credentials/sealed-credential.ts`.
- Officer changeover (July 1 slate transition) — `059_officer_changeover.sql:L1-L25` (2,231 lines), `src/app/(dashboard)/dashboard/officers/changeover/`.

**Dashboard surface actually present** (`find src/app -name page.tsx`): dashboard home, enquiries, events, programs, members, officers (+changeover), volunteers, prayer, recognitions, money (+print), pages (+preview, conflicts), news, navigation, pictures, designs, newsletters (+print), access, settings (profile, public-profile, appearance, billing), platform operator page. Sidebar entries match (`src/components/dashboard/navigation.ts:L129-L344`).

**TODO/FIXME/XXX/stub count** (`grep -rniE 'TODO|FIXME|XXX|not implemented|stub' src`): **18 matches total**, 13 of them the identifier `stub` in `src/modules/people/prayer/prayer.test.ts`, 1 in `src/test/next-font-stub.ts`, and three comments (`src/server/media/import-stock-photo.ts:L218`, `src/app/demo/admin/pages/[id]/page.tsx:L32`, `src/modules/recognitions/data/rpc.ts:L14`). **Zero `TODO`/`FIXME`/`XXX` in non-test source.** Incompleteness is instead expressed as fail-closed code (the 503 registration route, `HALLFORGE_*` flags defaulting to off per `wrangler.jsonc:L45-L49`) and in decision records.

**Stale README status paragraph.** `README.md:L104-L110` says "there is still no checkout" and "Revenue, custom domains and member management are not built". Checkout (`src/app/(dashboard)/dashboard/settings/billing/actions.ts:L20-L62` `createBillingCheckoutAction`), the billing webhook (`src/server/billing/webhook.ts`, 702 lines, `src/app/api/webhooks/stripe-billing/route.ts`) and the member roster are built; custom domains are not. The roadmap's "Last revised: September 1" (`docs/launch/2026-08-19-phase-roadmap.md:L3`) likewise predates migrations 048–060.

### Gaps
- No single current "state of the product" document: the ledger (Sept 1), the roadmap (Sept 1), the README and migration headers (through 060) disagree in the direction of the code being further along than the prose.
- Nothing records whether migrations 056–060 were applied to production; the last runbook that states production migration state says "001–034 aligned" (`docs/launch/2026-09-01-ops-runbook-two-manual-steps.md:L6-L14`) and the Sept 2 runbook says 055 "pushed neither to GitHub nor to the database" (`docs/launch/2026-09-02-billing-go-live.md:L17-L18`).
- The product contract's named test scripts (`test:integration`, `test:rls`, `test:e2e`, `test:a11y`) do not exist; `package.json:L9-L22` has only `test`/`test:run` (`docs/launch/2026-08-19-phase-roadmap.md:L376-L379`).

---

## 2. Recorded decisions on hosting, database and cost

### Takeaway
Hosting is decided and executed: **Cloudflare Workers via `@opennextjs/cloudflare`, Vercel retired** (DNS for `www`/apex still points at the old Vercel deployment and is intercepted by Worker Routes). Database is decided and executed: **Supabase Postgres + Auth as system of record, media on R2 from day one (Path D)**, with a written exit path to Neon + Better Auth and explicit triggers. Fixed infrastructure floor is recorded at **≈$31/month** today.

### Findings (cited)

**Hosting — Cloudflare over Vercel** (`docs/decisions/2026-08-18-cloudflare-workers-hosting.md`):
- Decision and four reasons in weight order: custom-domain fit via Cloudflare for SaaS, owner already runs ~20 Workers projects with R2, Cron Triggers for the payment workers, flat cost shape (L17-L24).
- Compatibility evidence: adapter supports Next 16.x; `proxy.ts` originally unsupported, resolved by Next 16.2 Build Adapters API; standing constraint "Node.js APIs are not available inside `proxy.ts`" (L26-L31).
- Vercel rejected on "fit and cost" — weaker platform-customer-domain story, per-seat pricing, separate vendor for cron/queues (L58). Pages and self-hosted VPS also rejected (L59-L60).
- Required work list included replacing `@vercel/analytics` and the `VERCEL_URL` fallback (L36-L37); today `grep -ri vercel src package.json` returns nothing, and `vercel.json:L1-L5` only sets `git.deploymentEnabled: false`.
- Roadmap Phase B: "Cloudflare consolidation, Vercel retired, www/apex/app/wildcard routing" (`docs/launch/2026-08-19-phase-roadmap.md:L19`).
- `wrangler.jsonc:L7-L11` records that `www` and apex DNS "still point at the retired Vercel deployment, and a Route intercepts the request before it reaches that origin"; routes at L12-L20; R2 incremental cache per `open-next.config.ts:L1-L8`.
- The two production-only defects that CI could not see (underscore route folder dropped by Next; `node:net` in Edge middleware) are the reason the deploy job runs a live smoke check (`docs/launch/2026-08-20-first-end-to-end-verification.md:L14-L30`; `.github/workflows/ci.yml` "Smoke check the deployed tenant path").

**Database — Supabase retained** (`docs/decisions/2026-07-15-database-platform.md`):
- Short answer: PostgreSQL, retain managed Supabase for pilot and first councils; Neon not a product-level improvement today; decision explicitly not permanent (L15-L21).
- "Use Supabase as infrastructure, not as the application architecture": browser use limited to auth and signed uploads, all business access through server services, provider clients behind adapters, forward-only SQL migrations, provider-independent RLS tests (L129-L141).
- Reconsideration triggers listed (L164-L177).

**Cost paths — Path D chosen** (`docs/decisions/2026-08-18-platform-cost-paths.md`):
- Paths A–D defined (L17-L22); monthly model Pilot/Growth/National: A ~$45/$961/$24,100; B ~$40/$357/$8,771; C ~$78/$737/$25,900; **D ~$40/$358/$8,783**, 2.5–3.5 engineer-weeks delta (L26-L33).
- Three decisive findings: R2 deletes media egress (A is 2.7× B nationally); Clerk MRO pricing disqualifies Path C (~$19k/mo auth at 17k orgs); the repo is "deliberately less Supabase-coupled" — identity via `getClaims()` behind an interface, 13 `supabase.rpc()` sites, portable SQL (L37-L39).
- Exit amendment: if triggers fire, **Neon + Better Auth**, not Clerk (L45). Staged sequencing: Supabase Pro $25 now, PITR/Large compute at ~250 councils, Team $599 at national (L49-L51). Triggers L53-L60. Researched vendor prices L64-L69.

**Cost notes as built** (`docs/decisions/2026-08-18-pricing-model.md:L59-L89`): fixed floor ≈ **$31/mo** (Supabase Pro $25, Workers Paid $5, domain ~$1, Resend free, R2 pennies), ≈ $51 once Resend Pro is needed; marginal cost Free ≈ $0.03–0.05/mo, Starter ≈ $0.70, Full ≈ $0.80; break-even 2 Full or 4–5 Starter subscriptions; at 250 councils infra ≈ 19% of revenue. Go/no-go: "a live pilot is ~2–3 weeks away at ~$40/month infrastructure" (`docs/decisions/2026-08-18-go-no-go-verdict.md:L13`).

**Runtime constraints that matter on Workers** (relevant to any engine reuse):
- Node 24 only (`package.json:L5-L8`, `.nvmrc`, `.node-version`); pnpm 11 with reviewed `allowBuilds` — `esbuild`, `workerd` allowed, `sharp` and `unrs-resolver` denied (`pnpm-workspace.yaml:L1-L5`; rationale `docs/decisions/2026-07-15-lts-runtime-and-dependencies.md:L36-L44`). Consequence: no server-side image processing; images are re-encoded in the browser (`src/components/media/prepare-image.ts`).
- `nodejs_compat` flag, compatibility date 2026-08-01 (`wrangler.jsonc:L4-L5`).
- Stripe webhook verification must use `constructEventAsync` on Workers (`docs/decisions/2026-09-01-stripe-billing-current-state.md:L22-L25`).
- Middleware/proxy must stay edge-safe (`docs/decisions/2026-08-18-cloudflare-workers-hosting.md:L31`).
- R2 bucket names are hard-coded into a CHECK constraint in migration 002, so they are "not free to change" (`wrangler.jsonc:L27-L31`).
- No Cron Trigger is configured, so the payment workers and scheduled publication have no scheduler (`wrangler.jsonc`; `src/server/payments/processing/README.md:L22-L25`).

### Gaps
- `.env.example` still lists `INNGEST_EVENT_KEY`/`INNGEST_SIGNING_KEY`, but `package.json` has no Inngest dependency and the hosting decision chose Cron Triggers instead — leftover.
- `next.config.ts:L31-L41` keeps `images.remotePatterns` for Supabase Storage public objects and Unsplash, although media is on R2 and Unsplash was ruled out (`docs/launch/2026-08-19-phase-roadmap.md:L132-L135`).
- No cost actuals (invoices) are in the repo; all figures are modelled.

---

## 3. Commercial model: pricing, plans, who pays, councils live / on waitlist

### Takeaway
Three plans — **Free / Starter $9/mo or $90/yr / Full $29/mo or $290/yr** — named Presence / Communication / Fundraising, with every one of 29 feature keys placed in exactly one tier by migration 055. The council pays (annual preferred; councils buy by motion and vote). **Four production councils, all on Free; the pilot council 16492 is to be comped to Full by the operator.** The repo records no waitlist count (the waitlist lives in a D1 database with a CSV admin export); the go/no-go gate is "25+ councils on a waitlist or pre-paying".

### Findings (cited)

**Prices and plans**
- Pricing decision (Aug 18): Free $0; Starter $9/mo, $90/yr; Full $29/mo, $290/yr; "pay for 10 months, get 12" (`docs/decisions/2026-08-18-pricing-model.md:L15-L23`). Market anchors L29-L30; unit economics L34-L38; annual anchoring fits check-paying councils L42-L44; risks L48-L50; Stripe Billing on the platform account, separate from Connect L54.
- Tier contents (Sept 2): Free = presence (subdomain, editor, rich text, pages, live preview, layouts, navigation, colours, logo, officers directory, picture library capped 1 GB, stock search, enquiries) — `docs/decisions/2026-09-02-plan-tiers-presence-communication-fundraising.md:L31-L47`; Starter adds custom domain, roster + import, calendar, volunteers (+public signup), news, programs, recognitions, prayer, newsletter issues, design studio — L49-L64; Full adds registration, card payments, store catalog/orders — L66-L73. Dashboard shows an upgrade panel rather than hiding tabs (L75).
- The in-product plan copy matches: `src/app/(dashboard)/dashboard/settings/billing/plans.ts:L37-L54` ("$90/yr", "$9/mo", "$290/yr", "$29/mo"; "one ticketed event pays for the website").
- Earlier `ideas.md:L35-L37` says Starter carries dues/donations and Full starts at "$288" — superseded by 055 ($290; payments.checkout is Full-only).
- Stripe fee arithmetic verified Sept 1: card $90/yr → $3.54; $290/yr → $10.74; ACH would halve it (`docs/decisions/2026-09-01-stripe-billing-current-state.md:L65-L83`). Live catalog: two products, four prices, portal configuration, one webhook endpoint (`docs/launch/2026-09-02-billing-go-live.md:L7-L12`).

**Who pays and how**
- Councils buy by parliamentary procedure; nothing is proposed and approved in the same meeting; $90 is within discretionary range, $288 "will require more of a budget meeting"; selling window May–June; trials must span 60–90 days; Free must need zero approval (`docs/decisions/2026-08-20-council-operations-ground-truth.md:L15-L28`).
- Founder's own answers: annual invoice "most likely would be better"; councils would "probably" pay for the newsletter alone; day-one hook is recruiting information plus a calendar (`docs/decisions/marketdecisions.md:L94-L110, L206-L207`).
- Councils are merchant of record for ticket money via Stripe Connect direct charges; **no platform application fee** in v1 (`docs/decisions/2026-08-18-pricing-model.md:L30, L49`; `docs/ideas.md:L15`). The platform home page sells "$0 Platform Fees" with a $20 fish-fry ticket worked example (`src/app/page.tsx:L103, L237-L283`) and hides pricing during early access (L374-L378).
- Operator can set a plan by hand with a written reason; the pilot council "does not pay through Stripe" (`docs/launch/2026-09-02-billing-go-live.md:L186-L191`).

**Councils live and on the waitlist**
- "All four production councils are on Free. Only the founder's pilot council (16492) has anything recorded in a feature outside Free: three studio designs" (`docs/decisions/2026-09-02-plan-tiers-presence-communication-fundraising.md:L83`).
- Waitlist page copy: "HallForge is being built right now, with two founding councils helping shape it. Signup isn't open yet" (`waitlist/public/index.html`, "Honest status" block in the L60-L140 region).
- Go/no-go falsification gate: "25+ councils on a waitlist or pre-paying for the $29 tier without founder-led sales" (`docs/decisions/2026-08-18-go-no-go-verdict.md:L43`). No document reports progress against it.
- Business framing: base case "high-margin lifestyle/mission business in the low-to-mid six figures ARR" (`go-no-go-verdict.md:L46`); ~$92k ARR at 1,000 adopters, ~$1.56M at full saturation (`pricing-model.md:L38`); platform thesis reframes KofC as beachhead and targets $500k/yr at ~3,300 councils across verticals (`docs/decisions/2026-08-20-platform-thesis-and-vertical-packs.md:L78-L82`). Incumbent uKnight estimated at $150–250k/yr, 1–3 people (`docs/uknight.md:L7-L30`).
- Trademark condition: obtain a trademark attorney's opinion on the neutral-host posture **before charging money** (`go-no-go-verdict.md:L39`). Nothing in the repo records that this happened.

**Waitlist app — what it is**
- A **signup app, not a marketing site**: a Cloudflare Worker (`waitlist/src/index.js`, 248 lines) with a D1 database bound as `DB` (`waitlist/wrangler.jsonc:L12-L18`), static HTML served as assets (L11), custom domain `waitlist.hallforge.com` (L6), Turnstile (L8-L9).
- Endpoints: `POST /api/join`, `GET /api/config`, `/admin` (HTML) and `/admin.csv` (`waitlist/src/index.js:L37-L49`); honeypot (L70-L73), Turnstile verification with hostname check (L80-L110), per-IP 5/hour rate limit (L112-L118), upsert on email (L125-L138); admin protected by `ADMIN_KEY` with timing-safe compare and attempt logging (L170-L234, `schema.sql:L18-L23`).
- Schema captures name, email, council number/name, city/state, current website, interests JSON, heard-about, notes, referrer, UTM, hashed IP, UA (`waitlist/schema.sql:L1-L17`). Interest checkboxes: website, events, member management, newsletters/email, online payments, moving off an old site (`waitlist/public/index.html`).
- The main app's landing page posts to it cross-origin (`src/components/waitlist-section.tsx:L6, L95`), with CORS allowlisted in the Worker (`waitlist/src/index.js:L7-L19`).

**What was promised to Council 16492** (`deliverables/`, `presentation/`, pilot decision)
- Ask: approve 16492 as first pilot, hire Jeremy under "a written milestone-based scope and not-to-exceed budget", evaluate HallForge as a repeatable platform (`docs/decisions/2026-07-15-council-16492-hallforge-pilot.md:L10-L14`; `deliverables/Council-16492-HallForge-Pilot-speaker-notes.md:L5, L77`). **No dollar figure for the builder's fee appears anywhere in the repo**; price is set in discovery (speaker notes L69).
- Timeline: discovery → public MVP in parallel with live Wix/WordPress → operations only after a security/payment gate → "90-day pilot should include at least one real event" → continue / revise / Wix fallback / stop with export (speaker notes L69).
- Honesty boundaries: no live payments, no real member data, no endorsement claim, no card storage; "HallForge is not production-ready today" (pilot decision L104-L113; speaker notes L61).
- Current-state costs quoted: Bluehost $612.21 / 3 yrs, Wix $1,116 / 3 yrs (speaker notes L17).
- Recorded risk: the deck "promises breadth (migration, CRM, payments) that is a multi-month program; the accepted July scope is the pages-CMS slice first, Stripe last" (`docs/launch/2026-08-18-launch-readiness-audit.md:L68`).

### Gaps
- No waitlist metrics, no conversion data, no signed pilot agreement or fee in the repo.
- Billing not live: Stripe account not activated, Worker lacks `STRIPE_SECRET_KEY` / `STRIPE_BILLING_WEBHOOK_SECRET` (`docs/launch/2026-09-02-billing-go-live.md:L14-L18`); ACH deliberately off because `checkout.session.async_payment_failed` is unhandled (L282-L290, L317-L319).
- The billing page copy was flagged as describing the old Free contents (`plan-tiers…md:L89`); `plans.ts` now matches 055, so this specific gap appears closed, but the `plans` table descriptions seeded by 028 are noted as stale and unused.

---

## 4. The founder's roadmap and what "finishing HallForge" means

### Takeaway
The roadmap explicitly refuses time estimates ("solo-founder week estimates proved meaningless", `docs/launch/2026-08-19-phase-roadmap.md:L7`). Its own summary: **9 of 17 phases shipped plus six unplanned features; 8 phases to contract-complete; the binding gap is that "nothing collects money"** (L146-L157, L397-L424). Recommended order: billing (7) → tier gating (8) → custom domains (6) → member management (9) (L416-L419). The module plan's "what is left, in order" is: rest of messaging → registration → programs then compliance → newsletter → entitlement enforcement (`docs/launch/2026-08-20-module-build-plan.md:L37-L47`). Where the docs give sizes, they come from the August 18 audit and the cost-path analysis, not the roadmap.

### Findings — remaining items with the docs' own size signals

| # | Item | Status per docs | Size signal in docs |
| --- | --- | --- | --- |
| 7 | **Billing go-live** | Code built (048, checkout, portal, webhook, operator plan control); not switched on | An 8-step runbook done "in one sitting" (`billing-go-live.md:L3-L5`); step 8 costs ~$9 real-card test (L210-L213). Open: ACH bounce handling, page self-refresh, double-customer cleanup (L307-L326) |
| 8 | **Tier gating / enforcement** | 055 places all 29 keys; SQL gates in 046; calendar/volunteers panels "in the same release as the migration" (`plan-tiers…md:L87`) | Roadmap: "deciding which routes consult it and turning them on one at a time" (L421-L423); storage byte-counting still noted (`module-build-plan.md:L110-L112`) |
| 6 | **Custom domains** (Starter headline) | Not built; no code references Cloudflare for SaaS | "Firm"; manual-with-detection "a legitimate v1" (`phase-roadmap.md:L197-L201`); automation ladder in `cloudflare-workers-hosting.md:L47-L54`; national hostname cost ~$1,690/mo (`platform-cost-paths.md:L51`) |
| 9 | **Member management → submissions** | Roster + CSV import built; the paid value (generate Supreme submissions, Form 1728A elimination, Star Council board) not built | "Highest-value phase, most discovery-dependent" (`phase-roadmap.md:L285-L289`); compliance engine design in `docs/decisions/2026-08-20-star-council-compliance-engine.md:L13-L44`; no filing API exists (L58-L60) |
| 10 | **Events with registration** | 503 stub | "Firm"; free registration first, capacity, check-in, reporting, confirmation email (`phase-roadmap.md:L295-L299`); audit sized it 2–3 weeks with events admin (`launch-readiness-audit.md:L51`) |
| 11 | **Payments (Connect ticketing)** | Orchestration + workers coded and tested; no entry point, no `/payment-status`, no scheduler, no Connect onboarding, no refunds admin | "Locked"; ten-gate checklist before live mode (`phase-roadmap.md:L301-L305`); audit sized Phase 4 at 3–4 weeks (`launch-readiness-audit.md:L53`) |
| 11b | **Council store** | Dials only | Depends on 11; products/variants, Checkout on connected account, orders, pickup fulfilment (`phase-roadmap.md:L307-L320`) |
| 12 | **Newsletter (email)** | Print/PDF issue exists; list management, consent, send pipeline, MJML, contributor workflow, archive deferred | "Fluid"; email now exists so "what remains … is the newsletter itself" (`phase-roadmap.md:L322-L331`); `newsletter-issue-from-records.md:L13` |
| — | **Programs ledger + compliance pack** | Programs table only; compliance nothing | "programs, then compliance" (`module-build-plan.md:L42-L43`); M8/M9 scope at L87-L88 |
| 15 | **Operator surface** | Roster, verification, cross-tenant list shipped; transfer, suspend-from-UI, disputes table manual | "correct at pilot volume" (`phase-roadmap.md:L363-L371`) |
| 16 | **Test harness** | 1,137 app tests / 806 pgTAP at Aug 30 (now larger); no browser e2e; contract scripts missing | (`phase-roadmap.md:L373-L389`) |
| 17 | **Human release gates** | Not started; "cannot be compressed" | 10 testers incl. 6 aged 60+, NVDA/VoiceOver, restore drill, domain rollback drill, two MFA owners (`phase-roadmap.md:L391-L393, L426-L430`); audit Phase 5 2–3 weeks calendar-bound (`launch-readiness-audit.md:L55`) |
| — | **MFA enforcement** | "The largest security gap" | passkey `aal1` caveat (`module-build-plan.md:L56-L58`) |
| — | **Browser signup verified once** | `bootstrap_organization` exercised directly, form not | (`module-build-plan.md:L54-L57`) |
| — | **Five demo screens** | Broken outside local dev since the rebaseline | separate work (`module-build-plan.md:L60-L62`) |
| — | **Fourth Degree assemblies** | Cannot onboard; bootstrap passes `organizationTypeKey: null` | "cheaper than it looks" (`phase-roadmap.md:L94-L113`) |
| — | **Members-only area, dues renewal** | Parity gaps vs incumbents, not on roadmap | (`phase-roadmap.md:L86-L92`) |
| — | **WordPress Migration Center** | 1,193-line design (`docs/decisions/2026-07-15-wordpress-migration-center.md`), **deferred indefinitely** | "team-sized programs the ARPU cannot fund" (`go-no-go-verdict.md:L38`); staged WXR import "when migration becomes the sales blocker", concierge first (`docs/decisions/2026-09-01-quiz-import-onboarding-concepts.md:L21-L28`) |
| — | **Direct-manipulation v2 editor + design studio maturity** | Sidebar-forms editor is "a stepping stone"; Puck noted as candidate; studio order banner → flyer → brochure → newsletter | (`docs/decisions/2026-09-01-direct-manipulation-editors.md:L13-L26, L30-L52`) |

**Historic whole-product estimates** (for calibration only; the founder later disowned week estimates): minimal live pilot 2–3 weeks, contract-complete launch 3–4 months (`launch-readiness-audit.md:L11-L12`); Phases 0–3 "~6–8 weeks" (`go-no-go-verdict.md:L38`). The first of those was met (Free tier live by Aug 20–30); the second has not been, consistent with the roadmap's L7 remark.

**What "finishing" means in the founder's own ordering** (`phase-roadmap.md:L416-L424`): make a tier mean something (billing live), then turn gates on, then custom domains, then the member/compliance wedge; everything else is "blocked on evidence rather than engineering" (L410-L414).

### Gaps
- No burn-down or dated changelog after Sept 2; the HTML build ledger (`docs/reports/2026-08-29-build-ledger.html`) is the last cross-cutting status artifact and already says "Built on Supabase Storage; R2 outstanding", which the roadmap corrected on Aug 30 (`phase-roadmap.md:L345-L348`).
- Programs/compliance (the paid wedge per the platform thesis) have no sizing at all beyond the schema sketch.

---

## 5. Code health signals

### Takeaway
Healthy for its age: strict TypeScript, Zod at boundaries, 200 vitest files with axe accessibility gates, 39 pgTAP suites, security-conscious SQL (FORCE RLS, SECURITY DEFINER-only access), current dependency line (Next 16.2.10, React 19.2.7, Tailwind 4.3.2, tRPC 11.18, Stripe 22.3.2) with a written upgrade policy. Two structural oddities for an engine fork: **tRPC is installed but the router is deliberately empty** (everything is Server Actions + route handlers), and **~9,000 lines of legacy/demo code remain** (demo pages, a retired `[orgSlug]` public tree with `as any` table reads, an old block renderer and block editor).

### Findings (cited)

**Size and shape**
- Files per top folder: `src/app` 228, `src/components` 171, `src/modules` 156, `src/server` 120, `src/lib` 17, `src/test` 8.
- Modules by file count: content 39, people 21, newsletter 15, presence 13, tenancy 10, calendar 9, members 8, media 7, design 7, orgtype 5, onboarding 5, entitlements 5, recognitions 4, programs 4, organizations 2, events 2. Server: content 34, payments 22, finance 20, organizations 6, http 6, billing 6, media 4, auth 4, email 3, credentials 3.
- Largest non-test files: `src/types/database.ts` 6,896 (generated), `src/components/dashboard/pages/block-fields.tsx` 2,667, `src/components/public/published-page-renderer.tsx` 1,756, `src/server/content/compiler.ts` 1,456, `src/server/content/admin/repository.ts` 1,328, `src/components/public/block-renderer.tsx` 1,328 (legacy), `src/modules/newsletter/render/NewsletterRenderer.tsx` 1,156, `src/app/demo/page-old.tsx` 1,126 and `demo-page-reference.tsx` 1,126 (legacy), `src/components/dashboard/pages/page-editor.tsx` 1,061, `src/components/page-editor/block-editor.tsx` 1,047 (legacy), `src/lib/mock-data.ts` 1,013 (legacy).
- Legacy weight: `src/app/demo/` 6,419 lines; `src/app/(public)/[orgSlug]/` 1,361 lines reading tables through `(supabase as any).from("organizations")` (`src/app/(public)/[orgSlug]/[[...slug]]/page.tsx:L20-L23`); `block-renderer.tsx` and `page-editor/` are imported only by `src/app/demo/page.tsx`. The audit listed deleting the `[orgSlug]` tree as a quick win that "permanently removes the dormant stored-XSS `dangerouslySetInnerHTML` paths" (`launch-readiness-audit.md:L61`); it is still present.
- Migrations: 60 files, 32,442 lines; the largest recent ones are 059 (2,231), 060 (1,766), 058 (1,436), 056 (716).

**Tests**
- 200 `*.test.ts(x)` files; ~1,963 `it(`/`test(` occurrences. Distribution: `app/(dashboard)` 42, `components/dashboard` 24, `modules/content` 21, `server/content` 14, `server/finance` 9, `components/public` 7, `server/payments` 6, `modules/newsletter` 6, `app/actions` 5, then 1–4 each across tenancy, people, organizations, presence, design, supabase lib, api routes, print pages, auth, billing, credentials, email, domains, media, trpc.
- Cross-cutting gates: `src/test/accessibility.test.ts`, `src/components/dashboard/pages/page-editor.a11y.test.tsx`, `src/components/public/published-page.a11y.test.tsx` (axe; serious/critical fail the build per `README.md:L31-L35`), `src/test/production-boundary-safety.test.ts`, `src/test/public-indexing-safety.test.ts`, `src/middleware.test.ts`. Timeout raised to 120 s for the axe suites (`vitest.config.ts:L9-L23`); `server-only` and `next/font` stubbed (L26-L33).
- pgTAP: 39 suites under `supabase/tests/database/` (001–007, 010–015, 021–028, 031, 039, 041, 045–046, 048–060); `plan()` counts sum to 1,575 assertions; includes a `pg_proc` scan proving every function is revoked/granted correctly (`README.md:L48-L55`).
- Not present: browser e2e (Playwright/Cypress) — none in `package.json:L59-L76`; the contract's named scripts (`phase-roadmap.md:L376-L379`).

**Dependencies and freshness** (`package.json:L23-L76`; lockfile resolutions)
- `next ^16.2.10` → 16.2.10 (`pnpm-lock.yaml:L4251`); `react`/`react-dom ^19.2.7` → 19.2.7 (L4567, L4513); `tailwindcss ^4.3.2` → 4.3.2 (L4861); `@trpc/* ^11.18.0`; `stripe 22.3.2` **exact** (L4817) — Stripe docs note 22.6.0 is latest, same major, upgrade "advisable, not required" (`stripe-billing-current-state.md:L13-L21`); `@opennextjs/cloudflare ^1.20.2` → 1.20.2 (L1491); `wrangler ^4.123.0` → 4.123.0 (L5204); `@supabase/supabase-js` → 2.110.5, `@supabase/ssr` → 0.12.3 (L2270-L2279); `konva ^10.3.2`, `react-konva ^19.2.5`; `@tiptap/* ^3.27.1`; `zod ^3.25.76` (Zod 4 deliberately deferred, `lts-runtime…md:L69`); `vitest ^4.1.10`, `vite ^8.1.4`, `jsdom ^29.1.1`.
- Policy: Node 24 LTS only, pnpm 11 pinned (`packageManager pnpm@11.22.0`, engines `>=11.13.0 <12`), 24-hour release-age rule, quarterly review, deferred majors list (`lts-runtime…md:L10-L16, L63-L84`). Security overrides for minimatch/brace-expansion/picomatch/flatted in `pnpm-workspace.yaml:L9-L17`.
- UI kit: shadcn-style Radix primitives (`components.json`), lucide icons, class-variance-authority, tailwind-merge.

**Architecture quirks relevant to reuse**
- **tRPC is plumbing without procedures**: `src/server/root.ts:L1-L19` — "Empty, deliberately… The v1 replacements are server components and server actions calling SECURITY DEFINER RPCs directly". 30 files declare `"use server"`. The `@trpc/*` and `@tanstack/react-query` dependencies, the `/api/trpc` route and `src/lib/trpc/client.tsx` remain.
- All data access is via `supabase.rpc()` to SECURITY DEFINER functions; tables have no browser privileges (`README.md:L48-L61`, `058_who_can_sign_in.sql:L17-L25`). Any engine built on this inherits a SQL-first, RPC-only style — a strength for tenancy, a cost for every new feature (migration + pgTAP + RPC adapter + action).
- Adding a block type "touches nine files" with `assertNever` guards (`README.md:L63-L74`).
- Publication pipeline has recorded, accepted risks (retry-forever on unknown SQLSTATE, fixed in 019; others left) in `docs/publication-pipeline-risks.md:L1-L30`.
- Secrets: all production secrets are Worker secrets; a master key seals council-entrusted credentials with versioned rotation (`.env.example` `HALLFORGE_CREDENTIAL_KEY_CURRENT` block; `src/server/credentials/sealed-credential.ts`). The repo commits the public Supabase URL and publishable key in `wrangler.jsonc:L63-L64` (public by design) and a Turnstile site key in `waitlist/wrangler.jsonc:L8`.

**Cloudflare-specific complications for running elsewhere / for a fork**
- Middleware must avoid Node built-ins (the `node:net` crash) — `first-end-to-end-verification.md:L22-L27`.
- No `sharp`; image variants are produced client-side; migration 047 "re-encoded variants trued up".
- R2 bindings (`MEDIA_ORIGINALS`, `MEDIA_VARIANTS`, `NEXT_INC_CACHE_R2_BUCKET`) are accessed through Worker bindings, so local `next dev` cannot serve media without `cf:preview`.
- No scheduler of any kind is configured; anything periodic (payment workers, scheduled publication, digests) still needs a Cron Trigger plus authenticated internal routes (`cloudflare-workers-hosting.md:L41`).
- Platform hosts list includes a `workers.dev` hostname (`wrangler.jsonc:L54`), and `www`/apex DNS still target Vercel behind Routes (L7-L11) — a deliberate but fragile arrangement.

### Gaps
- No lint rule enforces the module-boundary rules the architecture decision asks for (`docs/decisions/2026-08-20-modular-tool-architecture.md:L75-L81` "enforceable by lint and review"); `eslint.config.mjs` is the stock Next config.
- No dependency-update automation configured (`.github/` has only `ci.yml`), despite the policy asking for Dependabot/Renovate (`lts-runtime…md:L79`).
- No analytics or error-monitoring dependency in `package.json`; the audit asked for "error monitoring + uptime probes" in Phase 1 (`launch-readiness-audit.md:L47`); only Cloudflare `observability` is enabled (`wrangler.jsonc:L44`).

---

## Generic vs Knights-of-Columbus-specific

**Generic (organization-agnostic by design, confirmed in code)**
- Tenancy: host classification, wildcard subdomains, reserved slugs, `resolve_public_domain`, `/site` rewrite (`src/middleware.ts`, `src/modules/tenancy/`).
- Identity, sessions, auth callbacks, role presets and capability resolver (001, 058).
- `orgtype` vocabulary layer with a seeded `generic` row proving nothing is hard-coded (`010_orgtype_vocabulary.sql:L91-L99`; `src/modules/orgtype/contracts.ts:L6`).
- Content: `blocks_v1` schema, compiler, published renderer, autosave, review/conflicts, layouts, templates merge, navigation editor, appearance (three colours + badge with contrast arithmetic), sitemap/robots/`llms.txt` (`src/app/(public)/site/llms.txt/route.ts`), JSON-LD.
- Media: validate-then-store upload, R2 gated delivery, variants, gallery, rights attestation, Pexels/Pixabay stock import.
- Calendar/events with timezone-correct rendering and Event JSON-LD.
- Enquiries form (anonymous write path with Turnstile) and email notification.
- Entitlements (plans / features / overrides) and Stripe Billing (customer, subscription mirror, hosted Checkout, portal, webhook).
- Payments rail (Stripe Connect direct charges, idempotent webhook intake, leased workers) — generic but unreachable.
- Design studio (flyer/banner/custom on react-konva) and the "document with typed slots" model.
- Money report from a tenant's own restricted Stripe key, sealed credentials, buckets/ledger (056–060).
- Platform operator surface, verification states, suspension.
- Onboarding checklist (derived from records).
- Waitlist Worker (D1 + Turnstile + CSV admin).
- Build/CI/test harness, a11y gates, pgTAP tenancy proofs.

**KofC-specific (vocabulary, content or whole modules)**
- Default org type is `kofc`; `council_identifier` naming; claims by council number (`src/modules/orgtype/contracts.ts:L99-L116`; `009_council_claim_identity.sql`; `docs/decisions/2026-08-19-council-claims-and-disputes.md:L27-L38`).
- Starter template prose ("Through the Knights of Columbus our council is part of programs…") and the `supported_programs` block catalogue of the Order's programs (`src/modules/presence/starter/template.ts:L153-L168`; `src/modules/content/blocks-v1/supported-programs.ts:L4-L31`).
- Member records: degree/member-number/insurance fields (`src/components/members/member-form-dialog.tsx:L59, L491`; roster decision `docs/decisions/2026-08-20-member-roster-modelling.md:L7-L24`).
- Council offices and July-1 officer changeover (Form 185 cadence) — migration 059; prayer intentions; recognitions (Knight/Family of the Month); programs pillars Faith/Family/Community/Life (configurable via taxonomy but seeded KofC).
- Newsletter template `council-newsletter` reproducing the founder's award-winning issues; `docs/newsletters/` corpus.
- Compliance/Star Council engine design (unbuilt) — the only module *permitted* vertical logic by rule (`modular-tool-architecture.md:L58`).
- Demo site and admin for Council 16492 (`src/app/demo/`), `[orgSlug]` legacy tree, `mock-data.ts`.
- Navigation editor default external link to `kofc.org` (`NavigationEditor.tsx:L204`); landing-page copy built around fish-fry tickets and councils (`src/app/page.tsx`).
- Pricing, tier names, sales-cycle assumptions (Robert's Rules, May–June budget window) and the entire market analysis.
- Pilot deliverables and presentation generator.

---

## Reusable for a small-business storefront engine? Verdict per module

| Module / area | Verdict | Why |
| --- | --- | --- |
| Tenancy (`src/middleware.ts`, `src/modules/tenancy`, `src/server/domains`) | **Reuse as-is** | Generic subdomain/host resolution, reserved names, production-verified on Workers. Custom domains still have to be built either way. |
| Identity / access / roles (001, 058, `src/server/auth`, `src/app/(dashboard)/dashboard/access`) | **Reuse as-is** | Capability model is generic; role preset names (`council_owner`, `membership_secretary`) are strings in SQL and would be re-seeded. MFA enforcement is missing for both uses. |
| `orgtype` | **Reuse with changes** | Add a business type row; the column name `council_identifier` is "historical" by its own comment (`010:L115`). |
| Publishing / `blocks_v1` / compiler / renderer / editor | **Reuse with changes** | Pipeline, publish gates, live preview and a11y gates are the strongest asset. Block set is council-shaped (officers, volunteer shifts, supported programs, upcoming events) — a storefront needs product grid, hours/location, menu/price list, booking/CTA, testimonials, map. The founder's own v2 direction (direct manipulation over blocks, Puck candidate) is not yet built. |
| Media (R2, upload, variants, gallery, stock search) | **Reuse as-is** | Nothing council-specific; storage quota still needs byte counting. |
| Calendar / events | **Reuse with changes** | Fine for a business's events; registration/ticketing does not exist yet (503 stub). |
| Enquiries form + notification | **Reuse as-is** | Generic contact path with Turnstile and Resend. |
| Navigation editor, appearance, onboarding checklist | **Reuse as-is / light rename** | Checklist tasks are derived from records and partly council-worded. |
| Entitlements + Stripe Billing | **Reuse with changes** | Plan/feature tables and checkout/portal/webhook code are generic; re-seed plans and feature keys; finish go-live steps (account activation, secrets, ACH bounce handling). |
| Payments rail (Connect, orchestration, workers) | **Reuse with changes** | Solid, tested, but has no entry point, status page, Connect onboarding or scheduler; a storefront cart is a new build on top of it. |
| Store | **Build new** | Only feature-flag rows exist (036). |
| Design studio (flyers/banners) | **Reuse with changes** | Useful marketing collateral for a shop; templates are council-branded. |
| Newsletter | **Reuse with changes** (or drop) | Print-to-PDF issue only; no list management, consent or send pipeline; template is the council newsletter. For a business you would want email send, which does not exist. |
| Money report / sealed credentials / ledger (056–060) | **Reuse with changes** | "Read your own Stripe account" is generically useful; bucket vocabulary is council-ish. |
| Members roster / CSV import | **Replace** | Shape is a fraternal roster (degree, insurance); a storefront wants customers/CRM on the generic `contacts` tables (003) instead. |
| Officers, changeover, prayer, recognitions, programs, volunteers | **Drop** | Council-only; no storefront analogue. |
| Compliance | **Drop** | Unbuilt and vertical-only by rule. |
| Platform operator surface | **Reuse as-is** | Verification/suspension/plan override are tenant-platform basics. |
| Waitlist Worker | **Reuse as-is** | Form fields are council-flavoured but trivial to change. |
| Landing page (`src/app/page.tsx`) | **Replace** | Council marketing copy and fee comparison. |
| Demo (`src/app/demo`), `[orgSlug]` tree, `mock-data.ts`, legacy `block-renderer`/`page-editor` | **Drop (delete)** | ~9,000 lines of dead or reference-only code already slated for deletion by the audit. |
| tRPC scaffolding | **Drop or repurpose** | Router is empty; either remove the dependency set or use it for the client-side needs a storefront will have (cart). |
| `deliverables/`, `presentation/`, `analysis/`, `docs/newsletters/` | **Drop** | Pilot-specific artefacts. |
| Docs corpus (decisions, contracts, test matrix) | **Reuse with changes** | The architecture, tenancy, quality and hosting decisions transfer; pricing, market and pilot records do not. |

**Net:** roughly the foundation + presence + media + billing half of HallForge (tenancy, auth/roles, orgtype, publishing, media, calendar, enquiries, navigation/appearance, entitlements/billing, operator surface, CI/tests) is reusable for a storefront engine with vocabulary and block-set changes; the council-operations half (members/offices/changeover/prayer/recognitions/programs/volunteers/compliance/newsletter) is not, and the money-taking half (registration, Connect checkout, store, custom domains) is unfinished for HallForge itself and would be new work in either direction.
