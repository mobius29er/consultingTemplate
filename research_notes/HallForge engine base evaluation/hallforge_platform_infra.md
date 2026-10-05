# HallForge — platform and infrastructure evaluation

Read-only survey of `/home/user/hallforge` (shallow clone of `mobius29er/hallForge`), 2026-10-05.
All citations are relative to that root as `path:Lstart-Lend`. Secret values were never printed; variables are named only.

Scope of what was read: `package.json`, `pnpm-workspace.yaml`, `next.config.ts`, `open-next.config.ts`, `wrangler.jsonc`, `vercel.json`, `.env.example` (names only), `.node-version`, `tsconfig.json`, `eslint.config.mjs`, `vitest.config.ts`, `.github/`, `scripts/`, `README.md`, `supabase/` (config, email templates, all 60 migrations, 39 pgTAP suites, rebaseline and legacy-reference folders), `src/` layout, middleware, tRPC, server actions, API routes, and the relevant `docs/` decisions.

---

## 0. One-paragraph orientation

HallForge is a Next.js 16 / React 19 app deployed as a single Cloudflare Worker through `@opennextjs/cloudflare`, backed by Supabase Postgres 17 and Supabase Auth, with media and the Next incremental cache on three R2 buckets bound to the Worker. **Vercel is retired** (`vercel.json` only disables deployments). **There are no Supabase Edge Functions, no Supabase Storage buckets, no Realtime, no seed file**; the database layer is 60 append-only SQL migrations that create 90 tables (88 in `public`, 2 in `hallforge_private`) and 276 functions, almost all `SECURITY DEFINER`. **The tRPC router is deliberately empty**; the real API is 30 `"use server"` files plus 13 route handlers calling those RPCs. Tenancy is one shared schema, keyed by `organization_id`, enforced by RLS-forced tables with all browser-role privileges revoked and every read/write funnelled through capability-checked RPCs. The free tier is shipped; **billing checkout, Connect payments, custom domains, registration, newsletter sending, the store, and any per-tenant export are either stubbed, un-wired, or absent.**

---

## 1. Deployment targets and bindings

### Takeaway
One production target today: **Cloudflare Workers via OpenNext**, deployed manually from a GitHub Actions `workflow_dispatch`. Vercel is explicitly turned off. The Worker binds static assets and three R2 buckets — nothing else (no D1, KV, Durable Objects, Queues, or Cron Triggers). Supabase supplies Postgres and Auth only. A second, independent Worker (`waitlist/`) uses D1 and Turnstile.

### Findings
- **Scripts and adapter.** `cf:build`, `cf:preview`, `cf:deploy` wrap `opennextjs-cloudflare`; plain `build`/`start` remain for local Next (`package.json:L9-L19`). `@opennextjs/cloudflare ^1.20.2` and `wrangler ^4.123.0` are dev deps (`package.json:L68,L87`). Node `>=24 <25`, pnpm `>=11.13` (`package.json:L5-L8`, `.node-version:L1`, `.nvmrc:L1`).
- **Vercel retired.** `vercel.json:L1-L5` contains only `"git": {"deploymentEnabled": false}`. The hosting decision records that `@vercel/analytics` and the `VERCEL_URL` fallback were to be removed (`docs/decisions/2026-08-18-cloudflare-workers-hosting.md:L36-L37`); grep confirms zero `@vercel` imports remain in `src`. The roadmap lists "Vercel retired" under Phase B complete (`docs/launch/2026-08-19-phase-roadmap.md:L19`).
- **wrangler.jsonc bindings** (`wrangler.jsonc:L1-L72`):
  - `main: .open-next/worker.js`, `compatibility_date 2026-08-01`, `nodejs_compat` (L4-L6).
  - Routes: `app.hallforge.com` custom domain; `www`/apex via zone Routes (so the old Vercel DNS can stay and be intercepted); wildcard `*.hallforge.com/*` for council sites (L12-L20). Comment at L16-L18 warns that `mail` and `waitlist` hostnames are held by more specific routes.
  - `assets` binding `ASSETS` → `.open-next/assets` (L21-L24).
  - `r2_buckets`: `NEXT_INC_CACHE_R2_BUCKET` → `hallforge-next-cache`; `MEDIA_ORIGINALS` → `hallforge-media-originals`; `MEDIA_VARIANTS` → `hallforge-media-variants` (L31-L44). Neither media bucket has a public URL — "that absence IS the authorization model" (L25-L30; same statement in `src/server/media/buckets.ts:L3-L15`). Bucket names are pinned by a DB CHECK (`supabase/migrations/002_publishing_media.sql:L798-L804`).
  - `observability.enabled: true` (L45).
  - Public `vars`: `NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_APP_DOMAIN`, `HALLFORGE_BILLING_WEBHOOK_ENABLED`, `STRIPE_EXPECTED_LIVEMODE`, `HALLFORGE_PLATFORM_HOSTS`, `HALLFORGE_RESERVED_SUBDOMAINS`, `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (L52-L71). Secrets (`SUPABASE_SECRET_KEY`, Stripe secrets, `HALLFORGE_CREDENTIAL_KEY_CURRENT`) are `wrangler secret put` only (L46-L51).
  - **Absent:** `d1_databases`, `kv_namespaces`, `durable_objects`, `queues`, `triggers.crons`, `images`, `hyperdrive`.
- **open-next.config.ts** sets only `incrementalCache: r2IncrementalCache` (`open-next.config.ts:L1-L8`). No tag cache, no queue override.
- **Supabase side.** `supabase/config.toml:L1-L25` configures local ports and `major_version = 17` only. `supabase/functions/` **does not exist**; no `seed.sql` exists (verified with `ls`). No `CREATE EXTENSION`, no `pg_cron`/`pg_net`, no `storage.buckets` rows anywhere in migrations (grep across all 60 files returned nothing). `src` has zero `.channel(`/`storage.from(` calls. Supabase Auth is used, with seven custom email templates kept in-repo and pasted into the dashboard (`supabase/email-templates/README.md:L1-L40`).
- **Third-party services actually wired in code:** Stripe (`stripe` imported in 4 files: billing + payments), Resend via raw `fetch("https://api.resend.com/emails")` (`src/server/email/notify-enquiry.ts:L30,L93`; `notify-council-invitation.ts:L31`), Cloudflare Turnstile (`src/server/http/turnstile.ts`, `src/app/api/enquiries/route.ts`), Pexels/Pixabay stock search (5 files). **Declared in `.env.example` but unused in `src`:** `ANTHROPIC_API_KEY` (L17), `INNGEST_EVENT_KEY`/`INNGEST_SIGNING_KEY` (L52-L53) — grep finds 0 references to either.
- **Waitlist Worker** (`waitlist/wrangler.jsonc:L1-L20`): separate deployable at `waitlist.hallforge.com`, `assets` + **D1** binding `DB` (`hallforge-waitlist`) + Turnstile vars; `waitlist/schema.sql:L1-L27` has two SQLite tables (`waitlist`, `admin_attempts`) with KofC-specific columns (`council_number`, `council_name`). `eslint.config.mjs:L47-L51` excludes it from the app's lint.
- **CI** (`.github/workflows/ci.yml`): jobs `lint-and-typecheck` (L14-L37), `build` (Next, L39-L63), `cloudflare-build` (OpenNext bundle, L65-L91), `cloudflare-deploy` gated on `workflow_dispatch` only (L93-L131) with a post-deploy smoke check that expects platform 200 and unknown-tenant 404 (L140-L168), `test` (L170-L191), `production-database` (fresh Supabase, pgTAP, `db lint --fail-on error`, L193-L216), and `presentation` (builds a pptx deck, L218-L236). Deploy order rule: migration first (`supabase db push --linked`), then Worker (`README.md:L85-L95`).

### Gaps
- No scheduler of any kind: the hosting decision planned Cron Triggers for payment workers (`docs/decisions/2026-08-18-cloudflare-workers-hosting.md:L23,L41`) but `wrangler.jsonc` has no `triggers`, and grep for `scheduled(`/`cron` in `src` finds only a comment saying "No cron in version one" (`src/server/finance/ledger/sync.ts:L12`).
- CI still passes `NEXT_PUBLIC_SUPABASE_ANON_KEY` (`ci.yml:L63,L89,L118`) while the app reads `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (`.env.example:L8`); the admin client has a legacy fallback to `SUPABASE_SERVICE_ROLE_KEY` (`src/lib/supabase/admin.ts:L15-L17`).
- No staging environment, no preview deployments, no IaC for DNS/R2/Supabase (buckets and routes are created by hand per the runbooks).
- The `presentation/` CI job builds a KofC pilot pptx on every push — pure overhead for an engine.

---

## 2. Auth, roles, and API organisation

### Takeaway
Auth is Supabase Auth through `@supabase/ssr`, verified at the edge by `getClaims()` and a Zod claims schema — never trusting `getUser()` shortcuts. Authorisation is a three-layer model: **capabilities** (22 keys) bundled into **role presets** (7) per organization membership, **plan entitlements** (free/starter/full × 29 feature keys), and a separate `hallforge_private.platform_operators` table for cross-tenant staff. **tRPC exists as plumbing only — the `appRouter` is `{}`.** All mutations go through Next server actions (30 `"use server"` files) and 13 route handlers that call `SECURITY DEFINER` RPCs.

### Findings
- **Session verification.** `src/middleware.ts:L1-L4` imports `createServerClient` from `@supabase/ssr`; `verifyIdentity` calls `auth.getClaims()` and validates `sub`, `session_id`, `aud`, `role: "authenticated"`, `aal`, `exp`, `iat`, `is_anonymous` with Zod and rejects expired/anonymous/future-issued claims (`src/server/auth/identity.ts:L10-L80`). Results are tri-state: `authenticated | anonymous | unavailable` (L38-L41), and an Auth outage is deliberately not treated as anonymity (`src/middleware.ts:L159-L163` via the matcher tail).
- **Server client** is the standard cookie-bridged `createServerClient` (`src/lib/supabase/server.ts:L1-L31`). **Admin client** uses `SUPABASE_SECRET_KEY` and is `server-only` (`src/lib/supabase/admin.ts:L1-L31`).
- **Middleware is the single network boundary** (`src/middleware.ts:L126-L331`): classifies host → platform vs tenant; resolves tenant hostname through RPC `resolve_public_domain` (L201-L223); 308-redirects non-canonical hosts (L225-L253); injects trusted `x-hallforge-site-id/organization-id/tenant-host/tenant-source` headers after stripping any client-supplied copies (`src/modules/tenancy/headers.ts:L1-L25`); rewrites tenant public pages to the internal `/site/...` route and 404s direct hits on `/site` (L153-L158, L309-L314); redirects tenant-host signups to the platform host (L279-L307); adds HSTS/CSP-frame-ancestors/etc. (L353-L367). Note at L336-L340: Next 16 renamed `middleware.ts` → `proxy.ts` but the OpenNext adapter did not yet support the Node-runtime proxy, so the file stays `middleware.ts` on the **edge runtime**.
- **Roles/capabilities seed** (`supabase/migrations/001_identity_tenancy.sql:L483-L512`): 22 capabilities (`organization.manage`, `domains.manage`, `users.manage`, `content.*` ×8, `media.upload/manage`, `events.manage`, `registrations.manage`, `volunteers.manage`, `people.manage`, `people.export`, `finance.manage`, `imports.manage`, `audit.view`) and 7 role presets (`council_owner`, `website_manager`, `content_author`, `event_manager`, `media_contributor`, `finance_administrator`, `read_only_auditor`). Effective capabilities = preset ∪ per-membership overrides (`membership_capability_overrides`, L284). Capability check helper: `hallforge_private.user_has_organization_capability` (L884-L905).
- **Active organisation context** is a single RPC `get_active_organization_context` whose row carries `role_keys`, `capability_keys`, `plan_key`, `entitlement_keys`, `verification_state` (`src/server/organizations/active-context.ts:L15-L49`), persisted per user in a cookie (`active-cookie.ts`), with a `/select-council` chooser.
- **Plans and features.** `plans` seeded `free/starter/full` (`028_plan_entitlements.sql:L63-L66`); 29 feature keys across migrations 028/036/038/046/055 (`appearance.*`, `calendar.*`, `design.studio`, `enquiries.form`, `media.*`, `members.*`, `navigation.editor`, `news.posts`, `newsletter.issues`, `officers.directory`, `pages.*`, `payments.checkout`, `prayer.intentions`, `presence.*`, `programs.ledger`, `recognitions.awards`, `store.catalog/orders`, `volunteers.*`). Code mirror: `PLAN_KEYS` in `src/modules/entitlements/features.ts:L67`. Features carry an `enforced` flag that is honest about what is gated (`028:L79`).
- **Platform operators** live in `hallforge_private.platform_operators` (`025_platform_operator.sql:L25`), granted to the founder in `044_founder_operator_granted.sql`, with a cross-tenant list at `/dashboard/platform` (`src/app/(platform)/dashboard/platform/page.tsx`).
- **tRPC.** `src/server/root.ts:L1-L19`: `appRouter = createTRPCRouter({})`, with a comment that every former router queried the retired prototype schema. `src/server/trpc/index.ts:L103-L207` still defines `authenticatedProcedure`, `organizationProcedure`, `capabilityProcedure(cap)`, `featureProcedure(feature, cap?)` middleware — well-designed, **zero procedures use them**. Router/procedure count: **0 routers, 0 procedures.**
- **Actual API surface.**
  - `src/app/actions/*.ts`: 9 files, 15 exported server actions (`active-organization` 2, `attest-picture` 1, `bootstrap-organization` 1, `import-members` 2, `manage-events` 3, `manage-members` 2, `rename-organization` 2, `save-public-profile` 1, `update-my-profile` 1). A further 21 colocated `actions.ts` files under `src/app/(dashboard)/...` bring the `"use server"` total to 30 files.
  - Route handlers (13 non-test): `api/enquiries`, `api/media/{library,preview/[assetId]/[variantKey],stock-import,stock-search,upload}`, `api/pages/preview-context`, `api/registrations/create` (**503 stub**, `route.ts:L1-L28`), `api/trpc/[trpc]` (serves the empty router), `api/webhooks/stripe` (Connect), `api/webhooks/stripe-billing` (subscriptions), plus `auth/confirm`, `auth/callback`, `signout`, `robots.txt`, `sitemap.xml`.
  - Dashboard pages: 37 under `(dashboard)`, 1 `(platform)`, 3 `(print)`; auth: 4; public live: 5 under `(public)/site/**`; **retired**: 4 under `(public)/[orgSlug]/**` (still in tree, uses `(supabase as any).from("organizations")` against columns the v1 schema does not have — `src/app/(public)/[orgSlug]/[[...slug]]/page.tsx:L21-L30`; middleware 404s it at L322-L328); demo: 16 mock pages under `src/app/demo/**`.

### Gaps
- tRPC is dead weight: `@trpc/*`, `@tanstack/react-query`, `superjson` are shipped for a `{}` router. Either delete or adopt — the guard middlewares are the only valuable part.
- No MFA enforcement in code (claims carry `aal` but nothing requires `aal2`); the roadmap's release gate wants two owners with MFA (`docs/launch/2026-08-19-phase-roadmap.md:L391-L393`).
- Feature gating "mechanism shipped; nothing gated yet" (`phase-roadmap.md:L211`, L151-L156) — `featureProcedure`/`hasFeature` exist, routes mostly do not call them.
- The retired `[orgSlug]` tree and `src/lib/mock-data.ts` (used only by `[orgSlug]/blog/*`) are unreachable code that still compiles.

---

## 3. Data model and tenancy

### Takeaway
**One shared Postgres schema, 90 tables, every tenant table keyed by `organization_id`** (plus `site_id` for site-scoped rows), with composite FKs `(id, organization_id)` so a child cannot point at a parent in another tenant. RLS is enabled on all 88 public tables and forced on most; browser roles have no table privileges; 266 `SECURITY DEFINER` functions with `SET search_path = ''` are the only path, and pgTAP scans `pg_proc` to prove it. **There is no per-tenant export, archive, or handoff primitive** — it exists only as a capability name and a roadmap line.

### Findings — counts
| Object | Count | Evidence |
|---|---|---|
| Migrations | 60 (`001`–`060`) | `ls supabase/migrations/*.sql \| wc -l` |
| `CREATE TABLE` | 90 (88 `public`, 2 `hallforge_private`) | grep across migrations |
| `CREATE FUNCTION` | 276 (many `CREATE OR REPLACE` restatements of the same signature) | grep |
| `SECURITY DEFINER` mentions | 266 | grep |
| `CREATE POLICY` | 21 (all SELECT/UPDATE on identity + registry tables; **no** policy grants INSERT/DELETE to browser roles) | `001:L1573-L1683`, `011:L80`, `013:L116`, `015:L99`, `028:L410-L421` |
| RLS enabled | 47 explicit statements + loops over 22 tables (`002:L2696-L2727`, ENABLE+FORCE) and 23 tables (`003:L2194-L2210`, ENABLE) → effectively every tenant table | |
| `CREATE TRIGGER` | 32 (23 `set_updated_at`, 2 audit immutability, 4 `guard_facts`/reject-update, `hallforge_auth_user_created`, `media_assets_guard_delete`, `payment_provider_event_processing_clear_lease`) | grep list |
| Indexes | 93 | grep |
| Views / custom types / extensions / storage buckets / edge functions | 0 / 0 / 0 / 0 / 0 | grep, `ls` |
| Schemas | `public`, `hallforge_private` (`021:L24`, `025:L25`) | |
| `GRANT ... TO anon` | 22 statements (3 `GRANT EXECUTE`): only public projections are anon-callable | grep |
| Generated types | `src/types/database.ts` 6,896 lines | `wc -l` |

### Findings — schema by owning module
Tables are grouped by the migration that created them; "owning module" follows the codebase's own taxonomy (`docs/decisions/2026-08-20-modular-tool-architecture.md:L27-L47`).

| Module | Tables (migration) | Per-tenant? | Notes |
|---|---|---|---|
| **tenancy / identity / access** | `organizations`, `sites`, `site_domains`, `user_profiles`, `organization_memberships`, `capabilities`, `role_presets`, `role_preset_capabilities`, `organization_membership_roles`, `membership_capability_overrides`, `audit_events`, `idempotency_keys`, `outbox_messages`, `background_jobs` (001) · `hallforge_private.rate_limit_counters` (021) · `hallforge_private.platform_operators` (025) | all except `capabilities`, `role_presets`, `role_preset_capabilities`, `platform_operators`, `user_profiles` | `organizations.status` and `sites.status` are the kill switches (`001:L71-L122`). `site_domains` models verification/certificate lifecycle but **no application code touches it** (1 comment-only hit in `src`). `outbox_messages`/`background_jobs` exist with no consumer. |
| **orgtype** | `organization_types` (010) | global registry | nouns, identifier pattern, year boundary, program taxonomy per vertical (`010:L1-L40`); only KofC seeded. |
| **publishing (content)** | `site_content_policies`, `content_items`, `site_routes`, `content_revisions`, `content_conflicts`, `content_review_requests`, `content_validation_runs`, `content_publication_commands`, `content_publication_snapshots`, `content_publication_events`, `navigation_menus`, `navigation_revisions`, `navigation_revision_items`, `content_patterns`, `content_pattern_revisions`, `content_import_provenance` (002); extended by 005–007, 018–020, 024, 045, 047 | yes | News posts are `content_items.kind = 'post'`, no new table (`045:L1-L19`). `kind` admits `page/article/campaign/form/event_presentation/post` (`045:L27-L29`) but only `page` and `post` are built. |
| **media** | `media_assets`, `media_objects`, `media_revision_usages`, `media_public_entitlements`, `media_collections`, `media_collection_items` (002); `media_upload_slots` (015); 016/017/035/039/041 | yes | Object paths `org/<uuid>/asset/<uuid>/...` (`002:L805-L806`); storage quota enforced (041). |
| **presence** | `organization_profiles` (011); 027 appearance, 040 motion | yes | identity, meeting time, colours. |
| **calendar / events** | `events`, `event_occurrences`, `event_ticket_types`, `event_form_fields`, `event_registrations`, `event_registration_line_items`, `event_registration_answers`, `event_registration_state_events`, `registration_check_in_events` (003); `event_registration_communication_consents` (004); 012, 014 | yes | registration tables exist; public endpoint is a 503 stub. |
| **people (CRM-lite)** | `contacts`, `contact_communication_preferences`, `contact_communication_preference_events`, `contact_tags`, `contact_tag_assignments`, `contact_notes` (003); `public_enquiries` (023); `council_offices` (022, 053); `volunteer_shifts` (026) | yes | `contacts` family has no UI yet. |
| **members** | `member_records` (013) | yes | roster + CSV import. |
| **payments (Connect/ticketing)** | `payment_accounts`, `payment_orders`, `payment_attempts`, `payment_provider_events`, `payment_provider_event_processing`, `payment_ledger_entries`, `payment_refunds`, `payment_refund_state_events` (003); `payment_status_receipts`, `payment_refund_submissions`, `payment_refund_submission_events`, `payment_expiry_claims` (004); workers 008 | yes | orchestration has zero callers outside the webhook; workers never invoked. |
| **billing (platform subscriptions)** | `plans`, `features`, `plan_entitlements`, `organization_entitlements` (028; 030/032/033/036/038/046/055); `billing_customers`, `billing_subscriptions` (048) | entitlements per org; plans global | checkout + portal + webhook code exists; Stripe account not activated (see §5). |
| **design / newsletter** | `design_documents` (042; kinds widened by 049, 054) | yes | Konva flyer/banner canvas; newsletter issue is a `design_documents` row of kind `newsletter`, rendered to paged DOM for print — **no email send** (`054:L1-L13`). |
| **programs / recognitions / prayer** | `programs` (050), `recognitions` (051), `prayer_intentions` (052) | yes | KofC "Faith in Action" pillars; Catholic prayer intentions. |
| **officer changeover** | `officer_handovers`, `officer_handover_lines` (059) | yes | fraternal-year officer transition workflow (2,231 lines). |
| **finance (council's own money)** | `finance_stripe_connections` (056), `finance_buckets`, `finance_bucket_rules`, `finance_transactions`, `finance_transaction_sync` (060) | yes | read-only restricted Stripe key sealed with AES-256-GCM under a Worker master key (`056:L1-L30`, `src/server/credentials/`). |

### Findings — how tenancy is enforced
- Rebaseline conventions: "all tenant-owned records include `organization_id` and use tenant-safe foreign keys"; "Browser roles receive no broad business-table access"; every `SECURITY DEFINER` has `SET search_path = ''` and explicit grants; client-supplied org ids are never authoritative (`supabase/rebaseline/README.md:L29-L41`).
- Policies are written as `USING ((SELECT public.has_active_organization_membership(organization_id)))` / `has_organization_capability(organization_id, 'organization.manage')` (`001:L1573-L1600`); i.e. **RLS by org id via membership functions**, not `auth.jwt()` claims and not schema-per-tenant.
- Public reads are projections filtered on `organizations.status='active' AND sites.status='active'` (`README.md:L53-L55`).
- Request-time tenant identity comes from the hostname → `resolve_public_domain` → trusted headers (`src/middleware.ts:L201-L271`), never from a client path segment.
- A pgTAP assertion scans `pg_catalog.pg_proc` for any function missing the search_path/grant discipline (`supabase/tests/database/001_identity_tenancy.test.sql:L264`).
- Signup abuse: DB-side rate-limit counters per account and per hashed address (`021:L1-L20`); enquiries go through Turnstile.

### What per-tenant export or handoff would look like today
- **Nothing exists.** No function named `*export*`, `*portab*`, `*dump*`, or `delete_organization` in any migration (grep of 276 function names). The only "handover" is officer changeover (`059`), which transfers roles inside a tenant. `people.export` is a seeded capability with no implementation (`001:L500`). Docs list "data exports" and "officer handoff and data export" as pilot/exit gates, not shipped (`docs/decisions/2026-07-15-hallforge-production-architecture.md:L311,L327`; `2026-07-15-council-16492-hallforge-pilot.md:L134`).
- **What makes it tractable:** every row carries `organization_id`; media is keyed `org/<org-id>/...` in R2 (`002:L805-L806`); published content is an immutable JSON snapshot (`content_publication_snapshots`); the site is `blocks_v1` JSON. A handoff could therefore be: `SELECT ... WHERE organization_id = $1` across ~75 tenant tables → JSON, plus an R2 prefix copy, plus a static render of each snapshot. Because *everything* goes through RPCs, a `export_organization(p_org)` `SECURITY DEFINER` function returning JSONB is the natural shape and would be ~1 migration + pgTAP suite.
- **What makes it hard:** no schema-per-tenant or database-per-tenant means you cannot hand a customer "their database"; cross-tenant registries (`plans`, `features`, `organization_types`, `capabilities`) are shared; Stripe Connect accounts and sealed credentials are bound to HallForge's master key (`.env.example:L31-L46`) and would need re-sealing or exclusion.

### Gaps
- Several tables are infrastructure with no consumer: `outbox_messages`, `background_jobs` (no worker), `content_patterns`/`content_pattern_revisions`, `content_import_provenance`, the whole `contacts` family (no UI), `site_domains` (no code), `event_form_fields`, `media_collections`.
- `content_items.kind` lists five kinds; two are built.
- `supabase/legacy-reference/` (4,366-line prototype migration) and `supabase/rebaseline/` are duplicates of 001–008 kept for history; harmless but ~20k lines of noise in the folder.

---

## 4. CI, lint, typecheck, test setup and coverage

### Takeaway
Tooling is modern and strict: ESLint 9 flat config (`@eslint/js` + `typescript-eslint` recommended + Next core-web-vitals), `tsc --noEmit` with `strict: true`, Vitest 4 on jsdom with a 120 s timeout for axe suites, and pgTAP in a disposable Supabase container with `db lint --fail-on error`. **201 Vitest files / ~2,070 cases (1,990 `it(`/`test(` + 82 `it.each(`)** and **39 pgTAP files / 1,575 planned assertions**. Coverage is heaviest on content/publishing, dashboard components, finance, and the middleware; thinnest on billing, media, and anything Stripe-live.

### Findings
- **Lint:** `eslint.config.mjs:L5-L18` — `no-unused-vars` error (`_` prefix ignored), `no-explicit-any` warn; separate globals for `scripts/**/*.mjs` and `presentation/**/*.cjs` (L19-L46); ignores `.next`, `.open-next`, `node_modules`, `waitlist` (L50).
- **Typecheck:** `tsconfig.json:L2-L29` — ES2022, `strict`, `moduleResolution: bundler`, `@/*` alias; includes `.next/types`.
- **Vitest:** `vitest.config.ts:L5-L33` — jsdom, globals, `src/test/setup.ts` (adds `.hidden{display:none}` so visibility tests match a browser, `setup.ts:L16-L20`), aliases stub `server-only` and `next/font`. `testTimeout: 120_000` because axe passes over the full editor exceed 30 s (L11-L23). `@testing-library/react 16.3.2`, `axe-core` present (`package.json:L73-L79`).
- **pnpm-workspace.yaml:L1-L17** is not a monorepo manifest — it holds `allowBuilds` and security `overrides` (minimatch, brace-expansion, picomatch, flatted) only.
- **Vitest counts (computed):**
  - Files: 201 (`src/app` 60, `src/modules` 55, `src/server` 44, `src/components` 32, `src/lib` 5, `src/test` 3, `src/middleware.test.ts` 1, `scripts/` 1).
  - Cases by area: `src/modules` 685, `src/app` 499, `src/server` 390, `src/components` 339, `scripts` 27, `src/test` 19, `src/lib` 15; 472 `describe` blocks.
  - Modules covered: content 21 files, newsletter 6, tenancy 4, people 4, presence 3, design 3, members/media/entitlements/calendar 2 each, recognitions/programs/orgtype/organizations/onboarding/events 1 each. Server: content 14, finance 9, payments 6, organizations 3, http/auth 2, and 1 each for trpc/profiles/media/email/domains/credentials/context/billing. Components: dashboard 24, public 7, access 1.
- **pgTAP:** 39 suites under `supabase/tests/database/` (001–007, 010–015, 021–028, 031, 039, 041, 045, 046, 048–060); `plan()` sum 1,575. (Repo prose cites earlier snapshots: "736 unit tests, 611 database tests" in `ci.yml:L134-L135`; "1,137 application tests and 806 pgTAP assertions across 21 files" in `phase-roadmap.md:L373-L375`.)
- **What is not there:** the product contract's `test:integration`, `test:rls`, `test:e2e`, `test:a11y` scripts do not exist; there is **no browser-driven end-to-end suite** (`phase-roadmap.md:L373-L379`). No coverage reporting is configured. `scripts/seed-16492-q2-2024.test.ts` validates a KofC newsletter seed, not product code.
- **Operator scripts:** `scripts/stripe-billing-setup.mjs:L1-L30` idempotently creates the two Stripe products, four prices (lookup keys), portal config and webhook endpoint — reads `.env` by relative path. `scripts/seed-16492-q2-2024*.mjs` seed one council's newsletter for a print-parity proof (`scripts/README-seed-16492-q2-2024.md:L1-L25`) — KofC-specific throwaway.

### Gaps
- No E2E, no visual/regression tests, no coverage thresholds, no dependency-audit job, no CodeQL/secret scanning in CI.
- Database job runs on every push but is **not** a dependency of `cloudflare-deploy` (`ci.yml:L97` needs only lint and test) — a migration can be deployed to Workers without the pgTAP job having gated it.
- `.github/agents/*.md` (2,200 lines) are Copilot agent prompts, not CI.

---

## 5. Runtime constraints and hosting cost

### Takeaway
The app leans on the **edge-runtime middleware** and **Node-compat route handlers**; it uses **no ISR, no `next/image`, no static params** — the live public site is `force-dynamic` with `revalidate = 0`, so the R2 incremental cache binding is configured but effectively idle. There is no scheduler, so every background worker in the payments module is dead code until Cron Triggers or Queues are added. Hosting cost is modelled at ~$40/mo pilot → ~$358/mo at 250 orgs → ~$8.8k/mo at 17k orgs (Path D).

### Findings
- **Middleware on edge:** `src/middleware.ts:L336-L340` states the file remains `middleware.ts` (edge runtime) because `@opennextjs/cloudflare` did not support the Node-runtime `proxy.ts`; the hosting decision's standing constraint is "Node.js APIs are not available inside proxy" (`cloudflare-workers-hosting.md:L31`). CI comment records a prior outage from `node:net` in middleware (`ci.yml:L133-L139`). The matcher excludes `_next/static`, `_next/image`, favicon and image extensions (`middleware.ts:L364-L368`).
- **Route runtimes:** media and webhook handlers declare `export const runtime = "nodejs"; export const dynamic = "force-dynamic"` (`api/media/upload/route.ts:L18-L19`, `api/media/library/route.ts:L6-L7`, `api/media/stock-import/route.ts:L11-L12`, `api/media/stock-search/route.ts:L11`, `api/webhooks/stripe-billing/route.ts:L17-L18`, `api/webhooks/stripe/route.ts:L19-L20`).
- **ISR / caching:** only the retired `[orgSlug]` route declares `revalidate = 3600` (`(public)/[orgSlug]/[[...slug]]/page.tsx:L7-L8`); the live public route is `dynamic = "force-dynamic"; revalidate = 0` (`(public)/site/[[...path]]/page.tsx:L31-L32`). `revalidatePath` is used in 8 action files for dashboard paths only. No `"use cache"`, `unstable_cache`, or `generateStaticParams` anywhere. **Every tenant page render hits Supabase** (several RPCs per page: `page.tsx:L8-L29`).
- **Images:** `next.config.ts:L33-L45` whitelists Supabase storage and Unsplash remote patterns, but grep finds **0 files importing `next/image`**; media is served by the Worker from R2 through `/api/media/preview/...` and `/media/...` after `resolve_public_media` authorises (`src/server/media/buckets.ts:L3-L15`). Cloudflare Images transformations are not used; variants are produced in-Worker (migration 047 "reencoded variants").
- **R2 bindings** resolved via `getCloudflareContext({ async: true })` with a dynamic import so Node builds/tests never load the shim (`buckets.ts:L45-L60`).
- **Server actions** body limit 2 MB (`next.config.ts:L25-L30`) — relevant to upload paths.
- **Stripe on Workers:** billing client reuses the payments client's `httpClient`/API version constants (`src/server/billing/stripe.ts:L11-L30`); webhook verification must use `constructEventAsync` on Workers (`docs/decisions/2026-09-01-stripe-billing-current-state.md:L20-L23`).
- **Background work:** payments `processing/{expiry-worker,provider-event-worker}.ts` and `orchestration/checkout.ts` have **no callers outside `src/server/payments`** except the webhook route (grep); roadmap: "the orchestration exists with zero callers … workers exist and nothing invokes them" (`phase-roadmap.md:L301-L305`). Finance ledger sync is on-demand by design (`src/server/finance/ledger/sync.ts:L12-L30`).
- **Billing state (Sept 2):** catalog, portal config and webhook endpoint exist in Stripe; but the Stripe account is not activated (`details_submitted` false), production Workers lack `STRIPE_SECRET_KEY`/`STRIPE_BILLING_WEBHOOK_SECRET`, and migration 055 was unpushed at the time (`docs/launch/2026-09-02-billing-go-live.md:L8-L22`). Fail-closed flags `HALLFORGE_PAID_REGISTRATION_ENABLED`, `HALLFORGE_STRIPE_WEBHOOK_INGESTION_ENABLED` default off (`.env.example:L26-L29`).
- **Cost model** (`docs/decisions/2026-08-18-platform-cost-paths.md:L24-L33,L47-L51,L62-L69`): Path D (Supabase + R2, staged) ≈ **$40/mo pilot (5 orgs), $358/mo growth (250 orgs), $8,783/mo national (17k orgs)**. Drivers: Supabase Pro $25, Workers Paid $5 + $0.30/M req + $0.02/M CPU-ms, R2 $0.015/GB with $0 egress, Resend free→$20, Cloudflare for SaaS hostnames $0.10/mo after 100. Decisive finding: media egress on Supabase Storage would be ~$16k/mo at national scale, hence R2 (L37). Exit path if Supabase SLA pricing bites: Neon + Better Auth, not Clerk (L45).
- **No doc records Worker bundle-size or CPU-time limits** as a measured concern (grep of `docs/` for size/CPU/isolate found only pricing lines). The OpenNext bundle is built in CI but its size is not asserted.

### Gaps
- No Cron Triggers/Queues/Durable Objects → no scheduled publication, no payment expiry sweeps, no outbox drain, no newsletter send.
- No cache strategy for public pages (every visitor hit = multiple RPCs); R2 incremental cache is wired but unused by any ISR route.
- No Cloudflare for SaaS custom hostnames implemented (roadmap phase 6 "Firm", not started).
- Edge middleware constraint will bite any plan to add Node-only libraries at the boundary.

---

## 6. Generic vs Knights-of-Columbus-specific (platform/infrastructure view)

**Generic (vertical-neutral, reusable):**
- Cloudflare Worker + OpenNext pipeline, wildcard-subdomain tenant routing, trusted tenant headers, security headers (`wrangler.jsonc`, `src/middleware.ts`, `src/modules/tenancy/*`).
- R2 private-bucket media model with DB-pinned bucket names and org-prefixed object paths (`002`, `015`–`017`, `035`, `039`, `041`, `047`, `src/server/media/*`).
- Identity/tenancy/access core: organizations, sites, memberships, capabilities, role presets, overrides, audit, idempotency, outbox, jobs, rate limits, platform operators (`001`, `009`, `021`, `025`, `029`–`034`, `043`, `044`, `058`).
- `orgtype` vocabulary layer — explicitly built so "no core module may hard-code a vertical's nouns" (`010:L1-L16`; `modular-tool-architecture.md:L40-L41`).
- Publishing pipeline (`blocks_v1`, revisions, review, snapshots, navigation, news posts, sitemap/robots/JSON-LD, search visibility) (`002`, `005`–`007`, `018`–`020`, `024`, `045`, `src/server/content/*`, `src/modules/content/*`).
- Plans/features/entitlements registry and Stripe Billing subscriptions (`028`, `046`, `048`, `055`, `src/server/billing/*`, `scripts/stripe-billing-setup.mjs`) — only the tier *names* and prices are HallForge-specific.
- Events/calendar/registration/ticketing schema and Stripe Connect payment rails (`003`, `004`, `008`, `012`, `014`, `src/server/payments/*`).
- Enquiries + Turnstile, Resend transactional email, design studio (Konva flyers/banners), finance read-only Stripe reporting + money buckets (`023`, `042`, `049`, `056`, `060`).
- CI/lint/typecheck/vitest/pgTAP harness.

**KofC-specific (or Catholic-specific):**
- Vocabulary everywhere in UI and comments: "council" appears in 46 migration files and 341 `src` files; "Grand Knight" in 9/35; "fraternal year" 4/20; "KofC"/"Knights of Columbus" ~10/40 (grep counts). Mechanism is generic, prose is not.
- `organizations.council_identifier` column name (kept deliberately, `010:L12-L16`); `RESERVED_COUNCIL_SLUGS`, `/select-council` route, `HALLFORGE_RESERVED_SUBDOMAINS` list.
- `council_offices` + rosters (`022`, `053`) with council/assembly/program-director rosters; `officers` block in `blocks_v1`.
- `programs` "Faith in Action" pillars (`050`), `recognitions` awards (`051`), `prayer_intentions` (`052`), `officer_handovers` on the 1 July fraternal year (`059`).
- Starter site template content and the `/demo/**` mock admin (KofC emblem placeholders, `src/app/demo/*`), `docs/newsletters/*.pdf`, `scripts/seed-16492-*`, `deliverables/`, `presentation/`, `analysis/roast-uknight*`.
- Waitlist Worker fields (`council_number`, `council_name`), auth email copy ("your council's website").
- Pricing tiers Free/Starter/Full at $9/$29 and the "council store" dials.

---

## 7. Reusable for a small-business storefront engine? Verdict per module

| Module (code + migrations) | Verdict | Why |
|---|---|---|
| Cloudflare/OpenNext deploy, wrangler, CI (`wrangler.jsonc`, `open-next.config.ts`, `ci.yml`) | **Reuse as-is** | Already multi-tenant wildcard routing on Workers with R2; drop the `presentation` job and KofC hostnames/vars. |
| Tenant middleware + `modules/tenancy` | **Reuse as-is** | Host classification, canonical redirect, trusted headers, `/site` rewrite are vertical-neutral; rename `council` strings. |
| Identity / memberships / capabilities / role presets / audit (`001`, `058`) | **Reuse as-is** | Rename the 7 presets (owner, manager, author, …) in seed data only; capability keys are already generic. |
| Platform operator (`025`, `043`, `044`, `/dashboard/platform`) | **Reuse as-is** | Cross-tenant staff list and verification; add suspension/transfer later. |
| `orgtype` (`010`) | **Reuse with changes** | Keep the mechanism; seed a `business` type and widen fields (hours, tax id label) instead of fraternal-year/program taxonomy. |
| Publishing / blocks_v1 / navigation / news / SEO (`002`, `005`–`007`, `018`–`020`, `024`, `045`) | **Reuse as-is** | This is the strongest asset: isomorphic compiler/renderer, immutable snapshots, review and conflict handling, sitemap/JSON-LD. Swap the `officers`/`volunteer_shifts` blocks for `products`/`hours`/`testimonials`. |
| Media on R2 (`002`, `015`–`017`, `035`, `039`, `041`, `047`) | **Reuse as-is** | Private buckets, entitlements, quotas, stock-photo import — all generic. |
| Presence / appearance / profile (`011`, `027`, `040`) | **Reuse with changes** | Colours, badge, contrast gate, meeting time → business hours/address. Starter template must be rewritten. |
| Entitlements + plans + Stripe Billing (`028`, `046`, `048`, `055`, `src/server/billing`) | **Reuse with changes** | Mechanism is finished; rename tiers, re-seed features, activate the Stripe account, wire `featureProcedure` into routes (currently un-called). |
| Events / calendar / registration (`003`, `004`, `012`, `014`) | **Reuse with changes** | Schema is rich; the public registration endpoint is a 503 stub and must be built. Good base for bookings/classes. |
| Stripe Connect payments + workers (`003`, `004`, `008`, `src/server/payments`) | **Reuse with changes (large)** | Ledger, refunds, idempotency and gateway code exist and are tested, but nothing calls checkout and no scheduler runs the workers; needs Cron Triggers/Queues plus Connect onboarding UI. |
| Enquiries + Turnstile + Resend (`023`, `src/server/email`) | **Reuse as-is** | Contact form → inbox → email notify is exactly a storefront lead form. |
| Members roster + import (`013`) | **Reuse with changes** | Becomes customers/subscribers; strip degree/membership-number fields. |
| People / `contacts` family (`003`) | **Reuse with changes** | Unused CRM tables with consent events; good seed for a customer list if a UI is built. |
| `council_offices` / rosters (`022`, `053`) | **Replace** | Officer directories are fraternal; a storefront wants "team/staff" at most — simpler table. |
| Volunteer shifts (`026`) | **Drop** | No storefront analogue. |
| Programs / Faith in Action (`050`), Recognitions (`051`), Prayer intentions (`052`), Officer changeover (`059`) | **Drop** | KofC/Catholic-specific; 3,700 lines of SQL you would never ship to a shop. |
| Design studio flyers/banners (`042`, `049`, Konva components) | **Reuse with changes** | Promo flyers and social banners are useful to small businesses; templates are KofC-themed. |
| Newsletter issue (`054`, `modules/newsletter`) | **Reuse with changes** | Print-render only; a storefront needs email send + list management, which do not exist. |
| Finance: sealed read-only Stripe key, buckets, transactions (`056`, `057`, `060`, `src/server/finance`) | **Reuse as-is** | A "what came in this month" dashboard for an owner's own Stripe account is directly useful; credential sealing is reusable infra. |
| Store dials (`036`) | **Replace** | Only two feature flags exist; catalog/cart/orders must be built from scratch (roadmap 11b "registered, unbuilt"). |
| tRPC plumbing (`src/server/trpc`, `lib/trpc`, `api/trpc`) | **Drop (or adopt deliberately)** | Empty router; keep only the guard middlewares if you want a client-side API. |
| Retired `[orgSlug]` routes, `lib/mock-data.ts`, `src/app/demo/**` | **Drop** | Dead or mock code with KofC content. |
| Waitlist Worker (`waitlist/`) | **Reuse with changes** | Tiny D1 + Turnstile lead-capture; rename fields. |
| `scripts/seed-16492-*`, `deliverables/`, `presentation/`, `docs/newsletters/`, `analysis/`, `files/`, `files.zip` | **Drop** | Pilot artefacts. |
| Test harness (vitest + pgTAP + axe gate) | **Reuse as-is** | Keep the pgTAP `pg_proc` guard and the tenancy negative-test pattern; add E2E. |

**Bottom line for the founder:** the infrastructure, tenancy, publishing, media, entitlements and billing layers are genuinely finished and vertical-neutral — roughly 60–65% of the migrations and most of `src/server` and `src/modules/{content,media,tenancy,entitlements,presence}`. What is *not* finished is everything that moves money or runs in the background (checkout, Connect, registration, scheduler, newsletter send, custom domains, export), and about a third of the schema (offices, programs, recognitions, prayer, changeover, volunteers) is KofC-only and should be left behind rather than generalised.
