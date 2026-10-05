# HallForge: tenancy, organizations, onboarding, entitlements, org types, access control, money

Read-only review of `/home/user/hallforge` (shallow clone of mobius29er/hallForge). Every claim cites `path:Lstart-Lend` under that root. Nothing was modified, no git commands were run, no web lookups were made. Where something is stubbed, feature-flagged off, or explicitly documented as unbuilt, it is called out as such.

Orientation facts that shape everything below:

- The codebase was "rebaselined": 60 clean migrations in `supabase/migrations/` (001 to 060, 33k lines total) are the schema of record; `supabase/legacy-reference/` and `supabase/rebaseline/` hold older generations and are not applied. `src/server/root.ts:L1-L19` says the tRPC router is **deliberately empty** because every old router queried retired prototype tables; the v1 pattern is Server Components and Server Actions calling `SECURITY DEFINER` RPCs directly.
- Browser roles get **no table privileges at all** on the core tables (`supabase/migrations/001_identity_tenancy.sql:L1685-L1700`). RLS policies exist (143 `CREATE POLICY`/`FOR ...` lines across migrations, 47 `ENABLE ROW LEVEL SECURITY` statements) but are a backstop; the real door is ~266 `SECURITY DEFINER` functions with `SET search_path = ''`.
- Deployment target is Cloudflare Workers via OpenNext (`wrangler.jsonc`), with routes for `app.hallforge.com`, apex, `www`, and `*.hallforge.com`.

---

## 1. Tenancy: what is a tenant and how a request reaches it

### Takeaway

The tenant model is **organization -> site -> site_domains**, with a strict one-primary-site-per-org rule and one canonical domain per site. A request is mapped to a tenant **by hostname only**: subdomain (`<slug>.hallforge.com`) or verified custom domain, resolved through one anonymous RPC, then forwarded upstream as stripped-and-rewritten internal request headers. Dashboard tenancy is separate: an `httpOnly` cookie holding the active organization UUID, re-validated against membership on every request. Custom-domain **resolution** is finished and plan-gated; custom-domain **provisioning** (verification, certificates, attach flow, Cloudflare for SaaS) is not built at all.

### Findings

**Entities.**
- `organizations` (slug unique, name, `council_identifier`, status active/suspended/archived, tz/locale) `001_identity_tenancy.sql:L71-L91`.
- `sites` belong to an org; `site_key` default `main`; `is_primary` with a partial unique index so an org has at most one primary site `001:L93-L121`.
- `site_domains`: `hostname` globally unique; `kind` in fallback/primary/alias/campaign_redirect; `verification_status` pending/verified/failed; `certificate_status` pending/provisioning/active/failed; `is_active` only allowed when verified+cert active; `is_canonical` only for fallback/primary; `verification_token_hash`, `last_error` columns reserved `001:L123-L188`. One canonical domain per site enforced by partial unique index `001:L180-L182`.
- Bootstrap creates exactly one org, one primary site (`main`), one fallback domain (`<slug>.<APP_DOMAIN>`) pre-marked verified+active+canonical, one active membership with `council_owner` `001:L1477-L1517`.
- Later additions on `organizations`: `organization_type_key` (010), `plan_key` (028), `verification_state` (020/029).

**Request-to-tenant mapping (public sites).**
- The one network boundary is `src/middleware.ts` (kept as `middleware.ts` rather than Next 16 `proxy.ts` because OpenNext/Cloudflare does not yet support the Node proxy convention `src/middleware.ts:L336-L342`).
- Pure classification in `src/modules/tenancy/host.ts:L126-L193`: normalizes the `Host` header, classifies as `platform`, `subdomain` (label under `tenantSubdomainRoot`, minus reserved labels), `custom` (any other multi-label DNS name), or `invalid`. Platform hosts, tenant root and reserved labels come only from env (`NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_APP_DOMAIN`, `HALLFORGE_PLATFORM_HOSTS`, `HALLFORGE_RESERVED_SUBDOMAINS`) `src/middleware.ts:L379-L412`. `ip.ts` reimplements `isIP` because `node:net` crashes the Edge runtime `src/modules/tenancy/ip.ts:L1-L15`.
- Non-platform hosts go to `resolve_public_domain(hostname)` (`src/server/domains/public-resolution.ts:L64-L132`), the only anonymous domain lookup; it returns `organization_id, site_id, requested/canonical hostname, kind, serve|redirect, redirect_path` only for active+verified+cert-active rows `001:L1046-L1098`. Since 046 custom-kind rows additionally require the `presence.custom_domain` entitlement `046_plan_gates_honest.sql:L336-L343`.
- Outcome handling `src/middleware.ts:L201-L272`: 400 for invalid host, 404 for unknown, 503 for resolver error, 308 to canonical for alias/campaign hosts (GET/HEAD only; 421 otherwise), else trusted headers are set.
- Trusted context travels as **request** headers `x-hallforge-site-id`, `x-hallforge-organization-id`, `x-hallforge-tenant-host`, `x-hallforge-tenant-source`; client-supplied copies (and three legacy header names) are stripped first `src/modules/tenancy/headers.ts:L6-L79`. Readers reject partial/non-canonical sets `headers.ts:L85-L120`.
- Tenant GET/HEAD for public paths is rewritten to the internal `/site/<path>` route; direct `/site` hits are 404 `src/middleware.ts:L153-L158, L433-L454`. Application paths (`/api`, `/auth`, `/dashboard`, `/login`, etc.) pass through on tenant hosts `src/middleware.ts:L55-L68`; `/signup` on a tenant host is redirected to the platform origin `src/middleware.ts:L280-L307`.
- On platform hosts, anything that is not a known application path returns 404, which retires the legacy `/[orgSlug]/...` public tree `src/middleware.ts:L322-L328`. That legacy tree still exists in `src/app/(public)/[orgSlug]/{[[...slug]],blog,events}` and reads retired tables (`website_pages`, `blog_posts`, `website_settings`) -- dead code, unreachable.
- Security headers (nosniff, SAMEORIGIN, frame-ancestors, HSTS) added at this boundary `src/middleware.ts:L353-L367`.
- Design rationale and failure matrix: `src/modules/tenancy/DECISIONS.md:L41-L106, L155-L182`.

**Request-to-tenant mapping (dashboard).**
- `getClaims()` is called in middleware only as an optimistic login/dashboard redirect; it is explicitly not the authority `src/middleware.ts:L197-L199, L518-L571`; `DECISIONS.md:L108-L116`.
- Active org = `hallforge_active_organization` cookie (httpOnly, lax, 180 days) holding a canonical UUID `src/server/organizations/active-cookie.ts:L1-L78`. `createRequestContext` verifies identity then calls `get_active_organization_context(p_organization_id)` which returns rows only for the caller's own active membership `src/server/context/request-context.ts:L27-L67`; `001:L983-L1044`. Zero rows means `forbidden`, bad/missing cookie means `selection_required` `src/server/organizations/active-context.ts:L92-L140`.
- The context row carries org, site, membership id, `role_keys`, `capability_keys`, `plan_key`, `entitlement_keys`, `verification_state` `active-context.ts:L15-L32`; unknown feature keys are dropped `L145-L158`.
- `/select-council` lists active memberships via `list_my_active_organizations` and pending invitations; one-membership users are auto-selected `src/app/actions/active-organization.ts:L19-L77`; `src/app/select-council/page.tsx:L56-L111`.
- Dashboard layout redirects to the chooser on `selection_required`/`forbidden` and throws on `unavailable` `src/app/(dashboard)/layout.tsx:L42-L60`.

**Custom-domain provisioning.**
- No write path exists. The only `INSERT INTO public.site_domains` in any migration is bootstrap's fallback row `001:L1496-L1506`; `resolve_management_domain` is a capability-gated read `001:L1100-L1155`. Migration 046 states it plainly: "the attach flow does not exist yet. Nothing in the application can create a custom-kind row today -- only a manual service-role insert can" `046:L282-L287`. Migration 033 made the same point when it flipped `presence.custom_domain.enforced` to false `033_feature_enforcement_truth.sql:L1-L18`.
- No dashboard route for domains exists (`src/app/(dashboard)/dashboard/settings/` has appearance/billing/profile/public-profile only). No code references Cloudflare for SaaS custom hostnames; the hosting decision records it as the intended mechanism "when that feature is built" `docs/decisions/2026-08-18-cloudflare-workers-hosting.md:L21, L40, L49-L54`.
- Wildcard `*.hallforge.com` routing is configured, so **subdomain tenancy works end to end**; custom domains work only if an operator inserts a verified row by hand and DNS/TLS is arranged outside the product.

### Gaps

- Custom-domain attach/verify/certificate lifecycle: schema only. `verification_token_hash`, `certificate_status`, `last_error` have no writer.
- Multi-site per org is modelled but unused: bootstrap always creates `main`, and `get_active_organization_context` only ever returns the primary site `001:L1037-L1040`.
- Path-based tenancy was explicitly rejected and removed; a storefront engine wanting `/shop/<org>` paths would be re-adding something the author deliberately tore out.
- Legacy `src/hooks/use-organization.tsx` queries a nonexistent `user_memberships` table (`L74, L101`); dead.
- `src/types/database.ts` is acknowledged as stale ("generated legacy Database type does not contain the clean-v1 RPC yet" `src/middleware.ts:L105-L107`), so most RPC calls are typed through hand-written structural interfaces.

---

## 2. Roles, permissions, enforcement, and how a member gets access

### Takeaway

Authorization is a **capability model**: role presets map to capability keys, memberships hold roles, and per-membership allow/deny overrides sit on top. Resolution happens live in SQL on every request. Enforcement is layered: the `SECURITY DEFINER` function re-checks the capability against the organization in its argument (authoritative), RLS policies exist but browser roles have no table grants so they are mostly moot, and the application re-checks capabilities as the "polite half" so it can say why. The tRPC middleware chain (`capabilityProcedure`, `featureProcedure`) is fully written but **has no routers mounted**. Invitations, role grant/revoke, suspend/restore/remove exist (migration 058) with an owner-never-lost constraint trigger and a "you cannot hand out more than you hold" ladder.

### Findings

**Capabilities and roles.**
- 21 capability keys seeded: `organization.manage, domains.manage, users.manage, content.view_private/create/edit_own/edit_any/review/publish/restore/delete, media.upload/manage, events.manage, registrations.manage, volunteers.manage, people.manage, people.export, finance.manage, imports.manage, audit.view` `001:L482-L503`.
- 7 original role presets: `council_owner, website_manager, content_author, event_manager, media_contributor, finance_administrator, read_only_auditor` `001:L505-L512`, plus `membership_secretary` (013) `013_members_roster.sql:L45-L57`. `council_owner` later gained `people.manage` (013:L35-L40), `finance.manage` (057:L35-L37) and `people.export` (058:L120-L122), so today it holds every capability.
- Per-membership `membership_capability_overrides` with `allow`/`deny` and a reason `001:L284-L303`. Effective set = (role-derived OR explicit allow) AND NOT explicit deny, only for active memberships in active orgs `001:L835-L882`.
- Role keys are generic snake_case; the KofC-ness is only in the display names/descriptions (`Council Owner`) and the literal `'council_owner'` used by bootstrap `001:L1517`, the owner guard in 058, and the dashboard `OWNER_ROLE_KEY = "council_owner"` `src/components/dashboard/access/person-card.tsx:L45`.

**Enforcement layers.**
- SQL helpers: `has_active_organization_membership(uuid)` and `has_organization_capability(uuid, key)` use `auth.uid()`; the supplied org UUID "is never authority by itself" `001:L907-L936, L1773-L1774`. A worker variant `user_has_organization_capability(user, org, key)` is service-role only `001:L884-L905, L1765-L1767`.
- RLS policies on organizations/sites/site_domains/memberships/roles/overrides/audit use those helpers `001:L1573-L1681`, but all table privileges are revoked from anon/authenticated `001:L1685-L1700`; service_role gets ALL except on `audit_events` (SELECT only) `001:L1702-L1719`. Migration 058 restates: the policies "are real but unreachable from a browser role" `058_who_can_sign_in.sql:L16-L26`.
- Application layer: Server Actions check `verifySameOriginMutation` then capability from the active context (e.g. billing `src/app/(dashboard)/dashboard/settings/billing/actions.ts:L153-L174`; money `.../money/actions.ts:L55-L62`). Comments repeatedly say the DB re-asserts and the action check is "the polite half of one gate" `.../money/actions.ts:L58-L59`.
- tRPC: `authenticatedProcedure`, `organizationProcedure`, `capabilityProcedure(key)`, `featureProcedure(feature, capability?)` with typed boundary reasons `src/server/trpc/index.ts:L103-L207`. Router is `createTRPCRouter({})` `src/server/root.ts:L17`. So tRPC middleware is **finished but unused**.
- Platform operators: a `hallforge_private.platform_operators` roster readable by no browser role; `is_platform_operator()` gates cross-tenant reads/writes (verification, plan) `025_platform_operator.sql:L25-L69`; `am_i_platform_operator()` `025:L195-L206`; seeded from org creators at the time, later revoked/re-granted (043, 044).

**How a member gets access (058 + app).**
- `invite_council_person(org, email, role_keys[])`: requires `users.manage` in that org `058:L137-L163`; refuses empty roles; checks the actor holds every capability of every role being given (`0L000`) `058:L859-L865`; rate-limits 20 invites/org/hour via `consume_rate_limit` `058:L867-L876`; resolves email to `auth.users` and returns `needs_account` if none `058:L878-L888`; creates an `invited` membership or reinstates a removed one with a re-check of kept roles `058:L895-L944`; writes roles and an audit row `058:L946-L965`.
- The server action, on `needs_account`, calls Supabase `auth.admin.inviteUserByEmail` (creates the account and sends Supabase's invite template) then re-asks the DB `src/server/auth/invited-account.ts:L30-L72`; `src/app/(dashboard)/dashboard/access/actions.ts:L144-L195`. Existing accounts get a Resend email pointing at `/login?redirect=/select-council` `src/server/email/notify-council-invitation.ts:L26-L68`. Supabase's built-in mailer is rate-limited and "not for production" until custom SMTP is configured (comment at `invited-account.ts:L51-L58`).
- `accept_council_invitation(org)` flips invited to active for the caller only `058:L982-L1026`; the UI is `src/components/access/pending-invitations.tsx` on `/select-council`.
- Grant/revoke/suspend/restore/remove `058:L1057-L1434`. Nobody may edit their own access (`0LP01`) `058:L168-L180`; a council can never be left without an active `council_owner`, enforced by deferred constraint triggers on three tables plus an advisory lock `058:L39-L50, L264-L427`.
- SQLSTATE-to-reason mapping in TS: `src/modules/people/access/contracts.ts:L320-L356`. All RPC wrappers in `src/modules/people/access/data/rpc.ts:L111-L324`.
- Officer changeover (059) applies a saved slate by calling 058's functions rather than writing memberships directly `059_officer_changeover.sql:L37-L49`.

### Gaps

- No UI for `membership_capability_overrides`; overrides can only be written by service role.
- No custom roles: `role_presets.is_system` exists but there is no path to create a non-system preset.
- No org-level transfer of ownership, suspension, or disputes (025 says these are "not here" `025:L13-L17`).
- Access management is deliberately not plan-gated `058:L105-L108` -- good for reuse.
- `user_profiles` hold display name/locale/tz only; no avatar, no per-user notification preferences.

---

## 3. Onboarding, step by step; generic vs KofC

### Takeaway

Onboarding is two steps: Supabase email/password signup, then a single "council details" form (council number, council name, slug) whose server action calls `bootstrap_organization` with the service-role client, writes the active-org cookie, and best-effort publishes a word-only starter page so the subdomain serves something immediately. Everything after that is a derived (not stored) checklist on the dashboard. The mechanics are generic; the **form, validation and copy are KofC-specific** (council number required and unique, digit-normalized; `America/New_York` and `en-US` hardcoded; "Council" wording throughout), and the starter template's prose says "Through the Knights of Columbus".

### Findings

Step by step:
1. `/signup` page collects email/password (Supabase auth), then shows "Step 2 of 2: Council details": `councilIdentifier` (digits), `councilName` (auto-suggested as `Council <n>`), `slug` `src/app/(auth)/signup/page.tsx:L375-L462`. The page detects an existing council for a signed-in user and offers "Set up another council" `L344-L366`.
2. Contracts: `councilIdentifierSchema` strips non-digits and requires 1-10 digits `src/modules/organizations/onboarding/contracts.ts:L68-L77`; `councilSlugSchema` rejects a 60-word reserved list `L39-L62`; RPC args hardcode `p_default_timezone: "America/New_York"` and `p_default_locale: "en-US"` `L123-L133, L190-L209`; fallback hostname = `<slug>.<NEXT_PUBLIC_APP_DOMAIN>` `L175-L188`; idempotency key = `signup:<user>:<requestId>` `L166-L173`.
3. Server action `src/app/actions/bootstrap-organization.ts`: same-origin check `L49-L54`; parse `L56-L63`; identity `L65-L76`; two rate limits (per account, per client address; counter outage fails open) `L78-L84, L207-L243`; `bootstrap_organization` via admin client `L103-L111`; `23505` on `council_identifier` is surfaced as "already claimed, ask to be invited" `L113-L140`; write active-org cookie `L156-L160`; `provisionStarterSite` as the officer's own session (council_owner has content.create/publish) `L165-L178`; redirect to `/dashboard?welcome=site_live|setup` `L180-L184`.
4. `bootstrap_organization` is idempotent via `idempotency_keys` + advisory lock + fingerprint, creates org/site/domain/membership/role, writes audit and outbox rows `001:L1364-L1556`. Only service_role may execute it `001:L1739-L1753`.
5. Starter site: default template, "word-only" (photo sections dropped because signup must not wait on a photo provider) `src/server/presence/provision-starter-site.ts:L81-L89`; vocabulary loaded through orgtype but the call passes `organizationTypeKey: null` `bootstrap-organization.ts:L172`, so it always falls back to KofC. Template prose includes "Through the Knights of Columbus our council is part of programs..." `src/modules/presence/starter/template.ts:L153-L168`.
6. Post-signup checklist (`src/modules/onboarding/checklist.ts:L100-L218`): 11 tasks -- meeting details, dress page with photos, hero numbers, contact route, who we are, officers, first event, first picture, council colours, volunteer shift, get verified -- each hidden when the plan lacks the feature or the officer lacks the capability `L241-L262`. State is derived live from the records `L12-L21`; loaders in `src/modules/onboarding/load-state.ts:L37-L173`; banner figures read from the home page document `src/modules/onboarding/hero-stats.ts:L307-L344`. Nothing is persisted.
7. Verification: `organizations.verification_state` (unverified/verified/rejected) gates `noindex`/sitemap, set only by a platform operator through `set_organization_verification` `025:L117-L120`; `src/app/(platform)/dashboard/platform/actions.ts:L26-L71`. Design in `docs/decisions/2026-08-19-council-claims-and-disputes.md:L23-L50` (claim on council number, gate publishing not signup, operator reviews each claim at pilot scale). The "dispute" and "request access" flows from that design are **not built**; `009` only adds the unique index and the signup action only shows a sentence.

Generic vs KofC in onboarding:
- Generic: Supabase auth, idempotent bootstrap RPC, slug/hostname derivation, rate limits, cookie write, starter page provisioning, derived checklist pattern, operator verification gate.
- KofC-specific: required numeric `council_identifier` with a unique index on active orgs `009_council_claim_identity.sql:L20-L33`; `councilIdentifierSchema` digits-only; form labels and placeholders ("Council Number", "St. Augustine Council"); `Council <n>` auto-name; `America/New_York` default; template copy; all `select-council`/"council" route and copy names.

### Gaps

- Orgtype is never chosen at signup; `organizations.organization_type_key` defaults to `kofc` `010_orgtype_vocabulary.sql:L107-L109` and no code writes it.
- No city/state/role fields the claims design asked for `claims-and-disputes.md:L27`.
- No email/SMTP configured beyond Resend for transactional notices; Supabase's default mailer is relied on for invites.
- Second-officer confirmation and endorsement verification stages: design only.

---

## 4. Entitlements, plans, and Stripe

### Takeaway

Plans are **data, not code**: `plans`, `features`, `plan_entitlements` (with quotas), and per-org `organization_entitlements` overrides with reason/expiry, resolved by one SQL function and loaded into the request context. Three plans: Free (presence), Starter (communication, $9/mo or $90/yr), Full (fundraising, $29/mo or $290/yr). Stripe is used **three separate ways**: (a) **Stripe Billing** for HallForge's own subscriptions -- hosted Checkout, Customer Portal, snapshot webhooks, idempotent plan application; built, enabled in production config, but with a feature flag and a setup script dependency. (b) **Stripe Connect ticketing** for taking event payments on a council's behalf -- a large, carefully built boundary that is **disabled by default behind four flags, has no Connect-account onboarding, no scheduler for its workers, and is self-described as "not approved for live paid registration"**. (c) A **read-only restricted key** a council pastes so HallForge can report what came into the council's own Stripe account -- built, sealed with AES-GCM, on-demand only.

### Findings

**Plan/feature registry.**
- Tables `plans` (free/starter/full, rank), `features` (`module.feature` keys, `enforced` flag), `plan_entitlements` (included + numeric quota), `organization_entitlements` (included, quota, reason NOT NULL, expires_at, granted_by) `028_plan_entitlements.sql:L48-L227`. `organizations.plan_key` defaults to `free` `028:L236-L243`.
- Resolver `resolve_organization_entitlements(org)`: live override beats plan; absent means false `028:L255-L299` (restated/guarded in 032, boolean twin `organization_includes_feature` in 046 `046:L60-L93`). Fed into `get_active_organization_context` as `plan_key` + `entitlement_keys` `028:L321-L393`.
- Capabilities and entitlements are explicitly orthogonal; "both must pass" `028:L27-L32`; the same rule in `src/server/trpc/index.ts:L177-L190`.
- TS registry of 29 feature keys `src/modules/entitlements/features.ts:L322-L365` with `hasFeature` failing closed on null `L391-L400`; `FEATURE_PLAN` map and plain-English pitches `src/modules/entitlements/plans.ts:L82-L119, L141-L308`. Tests assert TS and migration agree (`plans.test.ts`, `features.test.ts` per the comments at `plans.ts:L33-L38`).
- Tier assignment per 055: presence (Free) = subdomain, pages.*, navigation, appearance.*, officers.directory/changeover, media.library/stock_search, enquiries.form; communication (Starter) = custom_domain, members.*, calendar.events, volunteers.*, news, programs, recognitions, prayer, newsletter, design.studio; fundraising (Full) = calendar.registration, payments.checkout, payments.report, store.* `055_plan_tiers_presence_communication_fundraising.sql:L84-L118`.
- `features.enforced` is kept honest: `calendar.registration`, `volunteers.public_signup`, `payments.checkout` false from birth `028:L120-L125`; `store.catalog/store.orders` false ("registered before it exists") `036_store_features.sql:L11-L27`; the five `pages.*` flipped to false in 046 because no gate was ever built `046:L350-L362`; `presence.custom_domain` true only because the resolver gate exists `046:L364-L371`. Retention-first rule: reads are never plan-gated, writes are `046:L28-L33`.
- Media quota (1/10/50 GB) is the only metered feature `028:L172-L195`; the context carries an empty quota map and the media library asks for its own `active-context.ts:L176-L183`.
- Operator override path: `set_organization_plan(org, plan, reason)` for check-paying councils, refuses while a live Stripe subscription exists `048_billing_subscriptions.sql:L416-L514`; UI `src/app/(platform)/dashboard/platform/actions.ts:L82-L144`. No UI for `organization_entitlements` overrides (service-role only `028:L417-L423`).

**(a) Stripe Billing -- platform subscriptions.**
- Checkout: `createBillingCheckoutAction` requires same-origin + `organization.manage` `billing/actions.ts:L45-L77, L153-L174`; refuses if a live subscription exists (fail closed on read error) `L183-L193, L378-L394`; looks up the price by lookup key (`starter_monthly|starter_annual|full_monthly|full_annual` `plans.ts:L16-L21`); reuses the stored customer; creates `mode: "subscription"` hosted Checkout with `client_reference_id` = org id, `subscription_data.metadata.organization_id`, `allow_promotion_codes`, card only (ACH deliberately excluded with a three-step precondition list) `L234-L275`.
- Portal: `createBillingPortalAction` opens Stripe's Customer Portal, naming a configuration stamped `metadata.hallforge=billing-v1` by `scripts/stripe-billing-setup.mjs` `src/server/billing/portal.ts:L190-L245`; falls back to account default.
- Webhook `/api/webhooks/stripe-billing` (`src/app/api/webhooks/stripe-billing/route.ts`): gated by `HALLFORGE_BILLING_WEBHOOK_ENABLED` (set `true` in `wrangler.jsonc` vars) and `STRIPE_BILLING_WEBHOOK_SECRET` `src/server/billing/config.ts:L1-L64`; bounded exact-body read, single-signature parse, `constructEventAsync` (Workers) `src/server/billing/webhook.ts:L214-L253, L577-L589`; refuses events carrying `account` `L244-L250`.
- Events handled: `checkout.session.completed`, `customer.subscription.created/updated/deleted`, `invoice.paid`, `invoice.payment_failed` `webhook.ts:L25-L32`. For every subscription/invoice event the subscription is **re-read from Stripe** before applying, so stale redeliveries cannot regress state `L289-L300, L303-L320`.
- Idempotency is in SQL: `apply_billing_subscription_state` locks the customer row, mirrors the subscription with an `IS DISTINCT FROM` upsert, derives plan from lookup-key prefix (`starter_`/`full_`, unknown fails loud), keeps paid plan on `past_due`, moves `organizations.plan_key` only on change and audits only then; refuses a second live subscription per org and cross-org subscription ids `048:L224-L401`. `ensure_billing_customer` refuses re-pointing an org to a different customer `048:L136-L213`. Both are service-role only and reject any signed-in caller `048:L152-L156, L245-L249`.
- Race handling: subscription-before-checkout heals the customer link from metadata `webhook.ts:L452-L485`; double-checkout is logged loudly and acknowledged (204) rather than 503-looped `L371-L393`.
- Billing tables `billing_customers`, `billing_subscriptions` are service-role only with forced RLS and no policies `048:L70-L126`.
- Status: `docs/decisions/2026-09-01-stripe-billing-current-state.md` is the research record; the code follows it (hosted Checkout, lookup keys, snapshot events, portal). Not implemented from that record: `invoice.payment_action_required`, ACH, grace-period override on `past_due` (the SQL instead keeps the paid plan).

**(b) Stripe Connect ticketing (paid event registration).**
- Boundary described in `src/server/payments/orchestration/README.md:L1-L65`: "implemented behind disabled-by-default feature gates; not approved for live paid registration" `L3-L4`. Flags: `HALLFORGE_PAID_REGISTRATION_ENABLED`, `HALLFORGE_STRIPE_WEBHOOK_INGESTION_ENABLED` `src/server/payments/config.ts:L1-L61`, plus `HALLFORGE_STRIPE_EVENT_WORKER_ENABLED`, `HALLFORGE_PAYMENT_EXPIRY_WORKER_ENABLED` `processing/README.md:L83-L86`. None are set in `wrangler.jsonc`.
- Checkout: begin attempt -> DB-authoritative context (amount, currency, line items, `provider_account_id` `acct_...`, provider idempotency key) -> Stripe Checkout on the connected account (`stripeAccount`) -> attach session `orchestration/checkout.ts:L33-L140`; `stripe/checkout.ts:L121`.
- Webhook `/api/webhooks/stripe` ingests via `ingest_payment_provider_event_by_account` (immutable, replay-safe) and only then acknowledges; allowlist `checkout.session.completed/expired`, `refund.created/updated/failed` `orchestration/README.md:L31-L50`; leased workers in 008 process events and expiries `008_payment_workers.sql:L3-L7`. "No scheduler is installed by this module" `processing/README.md:L88-L90`.
- **No Connect onboarding**: `payment_accounts` table exists `003_events_relationships.sql:L270-L295` but no application code references it (grep of `src` finds only `src/types/database.ts`), and there is no `accounts.create`/`accountLinks` usage anywhere in `src/server/payments`. A council cannot connect a Stripe account through the product.
- Remaining release gates listed by the author: workers, abuse controls, Connect liability settings, end-to-end sandbox tests `orchestration/README.md:L55-L65`.
- Entitlement side: `payments.checkout` and `calendar.registration` remain `enforced = FALSE`.

**(c) Council's own Stripe account, read-only ("Money").**
- Council pastes an `rk_` restricted key (sk_ refused); sealed AES-256-GCM in the Worker with `HALLFORGE_CREDENTIAL_KEY_CURRENT`, org id as AAD, only last four stored readable `056_council_stripe_reporting.sql:L20-L44`; `src/server/credentials/sealed-credential.ts:L1-L26`. Gated by `finance.manage` + `payments.report` (Full) on writes; reads and removal ungated `056:L73-L87`.
- Report runs on demand only -- "No cron in version one" `src/server/finance/ledger/sync.ts:L11-L34`; stores transactions into buckets with no payer PII by design `060_council_money_buckets.sql:L28-L59`.
- UI: `src/components/dashboard/money/stripe-key-panel.tsx`, `connect-key-instructions.tsx:L19-L23`, report panel with CSV/print, bucket "Guess/Confirmed/Event" wording `bucket-wording.ts:L29-L45`, staleness standings `money-sync-status.ts:L20-L48`. The Money page gates: `MONEY_LANE_CAPABILITY`/`MONEY_LANE_FEATURE` `src/app/(dashboard)/dashboard/money/data.ts:L53-L62`.

### Gaps

- Billing: no trial handling beyond mirroring `trialing`; no tax; no invoices UI (portal only); no handling of `checkout.session.async_payment_*`; webhook requires the `stripe-billing-setup.mjs` catalog (lookup keys, portal config) to have been run on the live account.
- Ticketing: feature-flagged off, no Connect onboarding, no scheduler, no public registration UI gate on (`calendar.registration` unenforced). Treat as a reference implementation, not a shippable rail.
- Store: `store.catalog`/`store.orders` are registry rows only; **no catalog, cart, or order schema exists** `036:L1-L27`.
- Money report: single-account, read-only, on-demand; not a ledger of HallForge-processed payments.
- `organization_entitlements` has no operator UI; pilot grants require SQL.

---

## 5. Org types: what `orgtype` abstracts, and could a `business` type be added?

### Takeaway

`orgtype` abstracts **vocabulary and calendar only**: unit noun, member noun, parent body noun, identifier label/regex/example, operating-year start, program taxonomy ("pillars"), and a `compliance_pack` key. It is reference data in `organization_types` with a `kofc` and a `generic` row. The thesis doc says the core must be organization-agnostic and only `compliance` vertical-specific, and the SQL/schema largely honours that. The TypeScript/UI layer does not: 341 of 498 non-test source files contain the word "council", most call sites pass `null` to the type loader (so KofC vocabulary is always used), and several modules (officers, changeover, programs/pillars, prayer intentions, newsletters) model a volunteer organization, not a business. A `business` **row** could be added without touching core, but it would change labels, not behaviour; making the product fit a business requires removing or hiding volunteer-org modules and the council-number claim.

### Findings

- Table and seed rows: `010_orgtype_vocabulary.sql:L20-L105` (`kofc`: council/Council number/`^[0-9]{1,10}$`/Supreme Council/fraternal year July 1/Faith,Family,Community,Life/compliance_pack kofc; `generic`: organization/Organization number/`^[A-Za-z0-9-]{1,20}$`/National Office/program year Jan 1/Service,Fellowship,Outreach/no pack). `get_organization_type(key)` is anon-readable `010:L126-L163`.
- TS contract and fallback `src/modules/orgtype/contracts.ts:L68-L171`; helpers `unitNoun`, `memberNoun`, `operatingYearFor`, `operatingYearLabel`, `isValidUnitIdentifier`, `pillarsFor` `src/modules/orgtype/domain/vocabulary.ts:L185-L266`; loader with cosmetic fallback to KofC `src/modules/orgtype/index.ts:L39-L54`.
- Rule stated in the module header: "no module outside `compliance` may hard-code a vertical's nouns" `orgtype/index.ts:L7-L8`. Thesis: `docs/decisions/2026-08-20-platform-thesis-and-vertical-packs.md:L42-L60`.
- Adoption: `organization_type_key` is joined in 050 (programs pillars validated against the type's taxonomy) and 053 (offices) and projected by the public profile RPCs (011/020/027/040). In the app, the loader is called from ~12 pages; the dashboard layout, dashboard home, public-profile, designs and bootstrap pass `null` (`src/app/(dashboard)/layout.tsx:L100-L103`; `bootstrap-organization.ts:L172`); officers, programs, members, newsletters pass the real key.
- Identifier: the DB constraint `organizations_council_identifier_digits` (009) and `councilIdentifierSchema` (digits only) **ignore** the type's `unit_identifier_pattern`; the `generic` row's alphanumeric pattern can never be stored. 010 explicitly declined to rename the column `010:L12-L16`.
- No `compliance` module exists in `src/modules` (listing: calendar, content, design, entitlements, events, media, members, newsletter, onboarding, organizations, orgtype, people, presence, programs, recognitions, tenancy). `compliance_pack` is a string nobody reads outside `orgtype` (grep for "compliance" in `src` hits only orgtype and database types).
- Volunteer-org assumptions outside orgtype: `council_offices` with consent flags (022), officer changeover tied to the fraternal year (059:L3-L11), `member_records` with `rank_key`/`standing` (013:L59-L70), prayer intentions (052), programs by pillar (050), recognitions (051), newsletter issues (054). Role names, copy, and routes (`/select-council`, "Grand Knight" in comments and UI) are KofC.

Could `business` be added without touching the core?
- Adding a row `('business', 'Small Business', 'business', 'businesses', 'Business number', ...)` works immediately for nouns/year/pillars. Nothing else changes: signup would still demand a numeric council number (constraint 009 + schema), `organization_type_key` would still default to `kofc` with no selector, the dashboard would still show officers/members/prayer/programs lanes, and the starter template would still mention the Knights. So: **yes at the data layer, no at the product layer.**

### Gaps

- No type selector anywhere; no way to set `organization_type_key` from the product.
- Identifier validation does not consult the type.
- Program taxonomy is the only behavioural lever (DB check on `pillar_key`); everything else is label-only.
- Compliance packs are a plan, not code.

---

## Generic vs Knights-of-Columbus-specific (this area)

Generic (reusable without conceptual change):
- Hostname classification, header stripping/forwarding, canonical redirects, security headers (`src/modules/tenancy/*`, `src/middleware.ts`).
- `organizations / sites / site_domains / organization_memberships / capabilities / role_presets / role_preset_capabilities / membership_capability_overrides / audit_events / idempotency_keys / outbox_messages / background_jobs` (001).
- `resolve_public_domain`, `get_active_organization_context`, `list_my_active_organizations`, `has_organization_capability`, `bootstrap_organization` (001).
- Active-org cookie + request context + tRPC procedure chain (unused but generic).
- Invitation/role/ownership-guard functions in 058 (names say "council" but logic is generic).
- Plans/features/plan_entitlements/organization_entitlements and the resolver (028/032/046/055).
- Stripe Billing rail end to end (048, `src/server/billing/*`, billing actions/page).
- Sealed-credential module and the read-only Stripe report/ledger (056/060, `src/server/credentials`, `src/server/finance/*`).
- Platform operator roster and verification/plan operator actions (025/048).
- Rate limiting (021) and same-origin mutation checks.
- `organization_types` table and the `orgtype` helpers.

KofC-specific:
- `council_identifier` required, numeric, unique on active orgs (009) and the signup form/schemas built around it; "Council <n>" auto-naming; `America/New_York`/`en-US` hardcoded in onboarding contracts.
- `council_owner` role key as the owner sentinel (001 bootstrap, 058 guard, `person-card.tsx:L45`); role display names and descriptions.
- Every user-facing string and route name: `/select-council`, "Choose your council", "Grand Knight", error sentences in server actions.
- Starter template copy (Knights of Columbus programs), officers/changeover (fraternal year), programs pillars (Faith/Family/Community/Life), prayer intentions, recognitions, newsletter -- adjacent modules this note did not review in depth but that the onboarding checklist and entitlement tiers are built around.
- Plan names and tier framing ("presence / communication / fundraising", $9 / $29) and the pitch copy in `plans.ts`.
- `select-council` chooser copy, dashboard layout wording, money-page copy ("council's Stripe account").
- The `kofc` default for `organization_type_key` and the KofC fallback vocabulary.

---

## Reusable for a small-business storefront engine? Verdict per module

| Module / area | Verdict | Why |
|---|---|---|
| `src/modules/tenancy` (host/headers/url/ip) + `src/middleware.ts` | **Reuse as-is** (rename env/header prefixes) | Pure, tested, Edge-safe; subdomain + custom-domain resolution via one RPC. Only the application-path lists and the `/signup` redirect need editing. |
| `organizations/sites/site_domains` schema + `resolve_public_domain` + bootstrap (001) | **Reuse with changes** | Drop `council_identifier` NOT-NULL expectations and the 009 unique index; keep sites/domains. Add the missing custom-domain attach flow (Cloudflare for SaaS) -- it is designed for but absent. |
| Active-org cookie / request context / `get_active_organization_context` | **Reuse as-is** | Generic; already carries plan + entitlements + capabilities per request. |
| tRPC procedure chain (`src/server/trpc/index.ts`) | **Reuse as-is or drop** | Fully written, zero routers. Keep if a client-side API is wanted; otherwise follow the house pattern (actions + RPCs) and delete. |
| Capabilities / role presets / overrides (001) | **Reuse with changes** | Replace the capability list (no `volunteers.manage`, add `orders.manage`, `catalog.manage`, `customers.view`); rename `council_owner` to `owner` everywhere (bootstrap, 058 guard, UI constant). |
| `people/access` (058 + `src/modules/people/access` + `src/components/dashboard/access`, `src/components/access`) | **Reuse with changes** | Invite/accept/grant/revoke/suspend/remove with owner guard and ladder are exactly what a business needs. Strip "council" from function names and copy; replace Supabase default mailer with custom SMTP before launch. |
| Onboarding contracts/action (`src/modules/organizations/onboarding`, `src/app/actions/bootstrap-organization.ts`) | **Replace** (keep the skeleton) | The idempotent bootstrap and rate-limit pattern is good; the form, schemas, tz/locale, council-number claim and starter template are wrong for a business. |
| `src/modules/onboarding` (checklist/load-state/hero-stats) | **Reuse the pattern, replace the tasks** | Derived-not-stored checklist filtered by feature+capability is a strong pattern; every task is council content. |
| `src/modules/entitlements` + 028/032/046/055 | **Reuse as-is** (new feature keys, new tiers) | Data-driven plans with overrides, quotas, fail-closed resolver, drift tests. Rewrite `FEATURE_KEYS`, `FEATURE_PLAN`, `FEATURE_PITCH`, plan names. |
| Stripe Billing rail (048, `src/server/billing`, billing settings) | **Reuse as-is** | Hosted Checkout + Portal + idempotent snapshot-webhook application is production-grade. Needs the setup script run per Stripe account; consider adding `invoice.payment_action_required` and ACH later. |
| Stripe Connect ticketing (003/004/008, `src/server/payments`) | **Replace or heavily adapt** | Flagged off, no Connect onboarding, no scheduler, event-registration shaped. For a storefront that sells on behalf of merchants, keep the webhook-intake/idempotency ideas and the worker-lease schema, but the order/registration model must be rebuilt around products and carts (which do not exist: `store.*` are registry rows only). |
| Read-only Stripe key + Money report (056/060, `src/server/finance`, `src/components/dashboard/money`) | **Reuse with changes** | Sealed credentials and "what came in / what Stripe kept" reporting fit a merchant dashboard; copy and bucket vocabulary need rewriting; on-demand-only sync is a limitation. |
| `src/server/credentials` (sealed credential, master key) | **Reuse as-is** | Generic AES-GCM envelope with versioned rotation. |
| Platform operator roster, verification, `set_organization_plan` (025/048, `(platform)` routes) | **Reuse with changes** | Verification-gates-indexing is reusable as "approved merchant"; copy is council. Add an operator UI for `organization_entitlements`. |
| `src/modules/orgtype` + 010 | **Reuse with changes** | Keep the table and helpers; add a `business` row; add a type selector at signup; make identifier validation consult the type; expect to replace the vocabulary fields with business-relevant ones (storefront noun, customer noun). |
| Legacy `src/app/(public)/[orgSlug]/*`, `src/hooks/use-organization.tsx`, `src/types/database.ts` | **Drop / regenerate** | Unreachable legacy tree reading retired tables; stale generated types. |

Overall: the **tenant, access, entitlement, and platform-billing substrate is finished and generic enough to be the base of an engine**; the **customer-facing product (onboarding, vocabulary, modules, starter content) is KofC through and through**, and the **merchant-payments rail (Connect, store, orders) does not exist beyond schema stubs and a disabled reference implementation**.
