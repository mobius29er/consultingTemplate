# Engine architecture: one engine, many storefronts

Status: proposal, October 2026. Audience: the founder and any developer who touches the engine. Grounding: the production CatholicJoe codebase at `/home/user/catholicjoe` (cited as `file:Lx-Ly`), the teardown in `research_notes/Local storefront platform strategy/reference_repo_teardown.md`, and the Cloudflare, dashboard, segment and integration notes under `research_notes/`. Figures from blocked vendor pages are marked (unverified) with the last known value.

## Contents

1. [Goals and non-goals](#1-goals-and-non-goals)
2. [One engine, many frontends](#2-one-engine-many-frontends)
3. [Repository layout](#3-repository-layout)
4. [Core modules and the D1 tables they own](#4-core-modules-and-the-d1-tables-they-own)
5. [The block schema](#5-the-block-schema)
6. [Variant presets](#6-variant-presets)
7. [The integrations framework](#7-the-integrations-framework)
8. [The two admins](#8-the-two-admins)
9. [Remote onboarding flow](#9-remote-onboarding-flow)
10. [Deployment model](#10-deployment-model)
11. [Path to the AI self-serve builder](#11-path-to-the-ai-self-serve-builder)
12. [Migration and extraction plan from CatholicJoe](#12-migration-and-extraction-plan-from-catholicjoe)
13. [Open questions and decisions](#13-open-questions-and-decisions)

---

## 1. Goals and non-goals

### Goals

- **Never rebuild the engine per client.** Every tenant runs the same `packages/core` at a pinned version. A client's site differs only in data: a tenant row, block JSON, theme tokens, connection records and secrets. If a client needs code, it is a new block type, adapter or core module that ships to every tenant; never a fork. CatholicJoe already has the right shape: all server modules take an injected `RuntimeEnv`, and only `database.ts` imports `cloudflare:workers` ([src/lib/server/database.ts:L1-L12](/home/user/catholicjoe/src/lib/server/database.ts)).
- **Scale without the founder.** The viability note's capacity check is the constraint: 70 sites at 0.5–1 hour each per month is 35–70 hours of maintenance, which only works if onboarding costs under 8 founder-hours per client and demos and configs are generated, not hand-built (`research_notes/Platform segments integrations competition/viability_solo_founder.md`, Q9). Provisioning, fleet upgrades and integration health must be jobs, not tasks.
- **Remote-first by construction.** Intake, demo, OAuth connection, content collection and go-live work with the owner on the phone and a browser. An in-person visit is a sales tactic that slots into the same flow (section 9).
- **Owners never handle API keys** where a vendor offers OAuth or hosted onboarding; key and webhook entry is the fallback, with inline validation (section 7).
- **One data model from one SKU to thousands**, including year/make/model fitment for the family auto parts store, without a schema per vertical.
- **Keep what CatholicJoe got right**: persist-before-attempt with lookup-only recovery for provider writes, lease and fence batches, signed-webhook verification, owner-revocable support access with audit, and the Miniflare test harness (`reference_repo_teardown.md`, Q10 Inferences).

### Non-goals

- Not a public CMS. The block editor serves the founder, the owner and later an AI, inside the tenant back-office.
- Not a shared-database multi-tenant SaaS in v1: the teardown costs that design at +8–12 days plus ongoing risk because ~28 tables carry global uniqueness constraints (`reference_repo_teardown.md`, Q9 §6). One Worker and one D1 per tenant.
- Not a unified-API vendor; adapters are provider-specific (`integrations_payments_commerce_accounting.md`, Q5).
- Not a POS replacement: the engine owns the web catalog and web order; stock and menus sync from Square, Clover, Lightspeed or Shopify (`integrations_fulfillment_ordering_inventory.md`, Q3).
- Not a self-serve AI builder on day one; section 11 lists what must be true first.

---

## 2. One engine, many frontends

The engine is a set of versioned packages; a tenant site is an Astro app that imports them and is configured by data.

| Layer | Varies per tenant | Never varies |
|---|---|---|
| Tenant config | slug, brand names, hosts, contact name, legal copy, inquiry kinds, enabled modules, tax mode, countries | the `TenantConfig` schema |
| Blocks | pages, sections, blocks and their props, navigation, redirects | schema version, component library, renderer, validation |
| Theme | token values, fonts, logo, OG image | token names, CSS variable contract, base layouts |
| Integrations | which connections exist, status, external ids, secrets | capability interfaces, adapters, webhook ingestion, job runner |
| Data | products, hours, team, gallery, bookings, orders | tables, migrations, APIs, admin screens |
| Core | nothing | auth and roles, checkout and reconciliation, outbox, inbox, fulfillment state machines, security, middleware, tests |

The teardown counted about 40 brand literals in generic CatholicJoe code: the host list ([src/worker.ts:L5](/home/user/catholicjoe/src/worker.ts)), `APP = 'catholic-joe'` as Stripe metadata and idempotency prefix ([src/lib/server/commerce.ts:L14](/home/user/catholicjoe/src/lib/server/commerce.ts)), the `cj_checkout_v2` cookie, the email display name ([src/lib/server/notifications.ts:L65](/home/user/catholicjoe/src/lib/server/notifications.ts)), the webhook header name ([src/lib/server/shipstation-tracking.ts:L64](/home/user/catholicjoe/src/lib/server/shipstation-tracking.ts)) and "Contact Steven" strings (`reference_repo_teardown.md`, Q7). Each becomes a `TenantConfig` field, the cheapest first step toward many frontends.

Two consequences shape everything below. **All mutation goes through one typed API**: the back-office UI, provisioning jobs and the future AI call the same Zod-validated core functions; nothing writes block JSON or connection rows directly. And **a tenant is reproducible from its D1 export plus the pinned template version**, which makes handoff an export-and-redeploy (`cloudflare_architecture.md`, Q6) and a fleet upgrade a job.

---

## 3. Repository layout

A pnpm workspace monorepo; package scope `@engine/` is a placeholder until the brand is chosen (`naming.md`).

```text
engine/
├── packages/
│   ├── core/                      # @engine/core — the engine; no brand strings
│   │   ├── src/
│   │   │   ├── env.ts             # RuntimeEnv, getEnv(), requireDb()   (from database.ts)
│   │   │   ├── tenant/            # TenantConfig schema + resolver
│   │   │   ├── auth/              # better-auth, Principal, memberships, step-up MFA
│   │   │   ├── catalog/  commerce/  fulfillment/  outbox/  inbox/  bookings/
│   │   │   ├── content/           # pages, revisions, nav, redirects (block JSON storage)
│   │   │   ├── media/  analytics/
│   │   │   ├── connections/       # connection records, webhook_events, provider_jobs
│   │   │   ├── jobs/              # budgeted flush of every queue + health checks
│   │   │   ├── security.ts  middleware.ts        # as-is from CatholicJoe
│   │   │   ├── worker.ts          # createWorker(tenantConfig) → fetch + scheduled
│   │   │   └── routes/            # API route handlers, injected via the Astro integration
│   │   ├── migrations/            # 0001-… versioned SQL, applied per tenant
│   │   ├── astro-integration.ts   # injectRoute /api/**, /account/**, /admin/**, /cart, /checkout/**
│   │   └── tests/                 # Miniflare D1 harness + provider fixtures (from tests/)
│   ├── blocks/                    # @engine/blocks
│   │   ├── schema/                # page.ts, section.ts, blocks/*.ts (Zod), migrate.ts
│   │   ├── components/            # Hero.astro, Menu.astro, ProductGrid.astro, ...
│   │   ├── render/                # PageRenderer.astro
│   │   └── tools/                 # typed tool-call definitions (editor + AI)
│   ├── integrations/              # @engine/integrations
│   │   ├── capabilities/          # PaymentsProvider.ts, CatalogSource.ts, ...
│   │   ├── stripe/ square/ mailchimp/ shipstation/ shippo/ shopify-import/ woocommerce/
│   │   ├── quickbooks/ calendly/ google-places/ cloudflare-email/ resend/ clover/
│   │   ├── registry.ts            # provider → adapter, auth mode, capabilities, tile copy
│   │   └── oauth/                 # NangoBroker | InHouseBroker, token store
│   ├── ui-themes/                 # token sets (JSON) + base.css declaring the variables
│   └── presets/                   # presence-services/ food-ordering/ retail-catalog/ custom/
├── apps/
│   ├── site/                      # per-tenant Astro app TEMPLATE, built once per version
│   │   ├── src/worker.ts          # export default createWorker(tenantConfig)
│   │   ├── src/tenant.config.ts   # generated at provision time from the tenant row
│   │   ├── src/pages/[...slug].astro         # renders published block JSON
│   │   ├── src/pages/preview/[...slug].astro
│   │   ├── src/pages/admin/**     # tenant back-office (section 8)
│   │   └── wrangler.template.jsonc            # no committed ids or live flags
│   └── founder-dashboard/         # separate Worker + platform D1 (section 8)
├── tooling/
│   ├── scaffold/                  # preset + intake → tenant row, pages, theme, config files
│   ├── provision/                 # D1 create, migrate, secrets, deploy, hostname, webhooks
│   ├── migrate-fleet/  upgrade-fleet/  smoke/
└── docs/
```

`apps/site` is a template, not a per-client folder; the only per-tenant artifacts are the generated config files and the D1. The dashboard note recommends building the bundle once per template version and uploading it per tenant with different bindings (`admin_dashboard_and_leads.md`, Q10 Inferences). The back-office lives under `/admin` where CatholicJoe's four admin pages are today ([src/pages/admin/index.astro](/home/user/catholicjoe/src/pages/admin/index.astro)), gated by the same middleware ([src/middleware.ts:L5-L23](/home/user/catholicjoe/src/middleware.ts)). `astro-integration.ts` is the teardown's `injectRoute` packaging step (`reference_repo_teardown.md`, Q9 §1).

---

## 4. Core modules and the D1 tables they own

Every tenant D1 carries the full core schema; presets enable modules through settings, never through different migrations. Table names reuse CatholicJoe's where the table survives so existing tests keep passing. Ids are TEXT, timestamps epoch ms.

**4.1 Tenants and config.** The source of brand, hosts, modules and settings; the resolver turns `env` plus the `tenant` row into a typed `TenantConfig` consumed everywhere (replacing `APP`, cookie and storage keys, email display name, auth `appName`, inquiry kinds, the host list). Tables: `tenant` (singleton `id CHECK(id=1)` in the style of `admin_access`, [migrations/0004-admin-access.sql:L1-L6](/home/user/catholicjoe/migrations/0004-admin-access.sql): slug, display_name, preset, template_version, schema_version, status), `tenant_settings` (key, value_json; e.g. `checkout.enabled`, `tax.mode`, `modules.bookings`), `tenant_domains`, `locations` (address, phone, hours_json, special_hours_json), `audit_events` (as-is). `CHECKOUT_ENABLED` moves from a committed var ([wrangler.jsonc:L28](/home/user/catholicjoe/wrangler.jsonc)) to a setting defaulting to off.

**4.2 Auth and roles.** better-auth email/password + TOTP as configured ([src/lib/server/auth.ts:L35-L87](/home/user/catholicjoe/src/lib/server/auth.ts)) with four roles. `owner` and `developer` exist today as `Principal.adminRole`, derived from the `ADMIN_EMAILS` and `DEVELOPER_ADMIN_EMAILS` secrets plus the `admin_access.developer_enabled` switch ([src/lib/server/auth.ts:L102-L120](/home/user/catholicjoe/src/lib/server/auth.ts); [src/lib/server/admin-access.ts:L9-L35](/home/user/catholicjoe/src/lib/server/admin-access.ts)); the teardown recommends a `memberships` table so changing an owner is a row, not a `secret put` (`reference_repo_teardown.md`, Q4 Inferences). `staff` is new (orders, inbox, bookings, products; no settings, apps or team). `platform` is the founder dashboard's service token, honoured only while the owner-controlled switch is on, keeping the revoke-on-next-request property the tests enforce ([tests/admin-access.test.ts:L106-L171](/home/user/catholicjoe/tests/admin-access.test.ts)). Tables: better-auth's `user, session, account, verification, twoFactor, rateLimit` ([migrations/0001-auth-and-messages.sql:L3-L41](/home/user/catholicjoe/migrations/0001-auth-and-messages.sql)), `admin_session_proofs`, `app_rate_limits`, new `memberships`, `platform_access`, `service_tokens`.

**4.3 Catalog and inventory.** Empty for a presence site, a menu for a cafe, hundreds of items for a boutique, thousands of SKUs with fitment for the parts store. CatholicJoe's catalog is a one-element constant priced live from a Stripe Product ([src/lib/server/commerce-catalog.ts:L7-L11, L69-L95](/home/user/catholicjoe/src/lib/server/commerce-catalog.ts)); the engine moves the catalog into D1 and keeps the Stripe Product id as an optional variant field. Tables: `products` (sku, slug, title, brand, product_type, status, core_charge_cents, shippable, requires_pickup, tax_code), `product_variants` (sku, price_cents, stripe_product_id, weight_oz, dims_json), `product_images`, `attribute_definitions` and `product_attributes` (a generic PIES-style layer serving size and color, food modifiers and part attributes alike), `categories`, `product_categories`, `collections` (rule_json), `inventory_levels` (variant_id, location_id, on_hand, reserved, source), `inventory_movements` (append-only), `vehicles` (VCdb base vehicle id, year, make, model, submodel, engine; loaded only for what the store stocks) and `fitments` (product_id, base_vehicle_id, engine_id, qualifier_ids_json, qty). The year/make/model selector resolves to a base vehicle and joins `fitments` → `products`; search is an external-content FTS5 table rebuilt around D1 exports, since databases with virtual tables cannot be exported (`cloudflare_architecture.md`, Q4; `integrations_fulfillment_ordering_inventory.md`, Q6). Checkout consumes `items: [{variantId, quantity}]`, keeping the single-line path as the degenerate case (`reference_repo_teardown.md`, Q9 §1); the CHECK enums in [migrations/0003-commerce-catalog.sql:L1-L5](/home/user/catholicjoe/migrations/0003-commerce-catalog.sql) become plain columns.

**4.4 Orders and payments.** The hosted Stripe Checkout flow as shipped (quote → attempt claims the quote atomically → per-attempt Customer and Session → webhook reconciliation), generalized to N lines behind a `PaymentsProvider` interface. Keep `processStripeEvent` persisting `stripe_events` before any work and deduping on status ([src/lib/server/commerce.ts:L452-L478](/home/user/catholicjoe/src/lib/server/commerce.ts)), the bounded raw body, signature verification and `waitUntil` fan-out ([src/lib/server/commerce-handlers.ts:L256-L302](/home/user/catholicjoe/src/lib/server/commerce-handlers.ts)), the lease and fence batch, and the hashed one-use order claim. Tables from [migrations/0002-orders.sql](/home/user/catholicjoe/migrations/0002-orders.sql) and [0005](/home/user/catholicjoe/migrations/0005-shipping-quotes.sql): `checkout_attempts` (+ `items_json`, `fulfillment_method`), `orders` (+ `location_id`, `pickup_at`, `tip_cents`, `core_charge_cents`), `order_lines`, `stripe_events`, `commerce_locks`, `commerce_sync_guard`, `order_claims`, `shipping_quotes`; new `payments` and `refunds` so a Square or PayPal payment lands without new `orders` columns. The dormant `ui_mode: 'form'` path is deleted: no client calls it (`reference_repo_teardown.md`, Q2 Inferences).

**4.5 Fulfillment.** The provider-agnostic version of the ShipStation export and tracking machinery. `flushShipStation` is the reference pattern for any write to a provider that permits duplicates: expired `processing` leases become `uncertain` ([src/lib/server/shipstation-fulfillment.ts:L108-L111](/home/user/catholicjoe/src/lib/server/shipstation-fulfillment.ts)), every attempt first looks up by `external_shipment_id` ([L139-L140](/home/user/catholicjoe/src/lib/server/shipstation-fulfillment.ts)), `post_uncertain=1` is persisted before the POST ([L172-L177](/home/user/catholicjoe/src/lib/server/shipstation-fulfillment.ts)), and only definite 4xx rejections are retryable ([L182-L185](/home/user/catholicjoe/src/lib/server/shipstation-fulfillment.ts)). Tables: `fulfillment_events` (as-is), `fulfillment_exports` (generalized `shipstation_exports` with a `provider` column), `shipments` (generalized `order_tracking`), `tracking_jobs`, `pickup_slots`. The monotonic tracking rules in `syncLabel` carry over unchanged ([src/lib/server/shipstation-tracking.ts:L96-L179](/home/user/catholicjoe/src/lib/server/shipstation-tracking.ts)).

**4.6 Email outbox.** Durable transactional email with a swappable `EmailSender`: Cloudflare `send_email` (today's binding) and Resend over HTTPS, because the binding ties the sending domain to the Worker's own account ([docs/setup.md:L51-L55](/home/user/catholicjoe/docs/setup.md); `reference_repo_teardown.md`, Q1 Inferences) and Resend's idempotency key lets the engine retry ambiguous sends instead of holding them forever, the teardown's fleet-scale worry (`reference_repo_teardown.md`, Q3 Inferences). The outbox and hold-ambiguous rule stay ([src/lib/server/notifications.ts:L46-L48, L71-L77](/home/user/catholicjoe/src/lib/server/notifications.ts)). Tables: `notification_outbox` (+ `template_key`, `vars_json`, `provider`, `provider_message_id`), `email_templates` so copy such as the order-paid email ([src/lib/server/commerce.ts:L419-L420](/home/user/catholicjoe/src/lib/server/commerce.ts)) is tenant data, `email_suppressions`.

**4.7 Inquiries and messages.** The inbox as shipped (threads, messages, notes, 24-hour claim, escaped search; [src/lib/server/messages.ts:L52-L58, L131-L148](/home/user/catholicjoe/src/lib/server/messages.ts)), with inquiry kinds moved from a code enum ([src/pages/api/inquiries.ts:L25-L28](/home/user/catholicjoe/src/pages/api/inquiries.ts)) to settings, a `quote_requests` shape for trades (service, address, photos, preferred window) and Turnstile on public POSTs, which the docs list as not configured ([docs/setup.md:L131](/home/user/catholicjoe/docs/setup.md)). Tables: `message_threads`, `messages`, `admin_notes`, `inquiry_claims`, `form_definitions`.

**4.8 Bookings.** Native bookings for service tenants without Square Appointments or Calendly, and a `BookingProvider` adapter for those with one. Native is deliberately small: services with duration, price and deposit; weekly availability per staff member; a slot query; a booking with deposit through Stripe Checkout; confirmation and reminders through the outbox. Tables: `booking_services`, `booking_availability`, `booking_blackouts`, `bookings` (+ `provider`, `external_id`, `order_id`), `booking_events`.

**4.9 Content blocks.** Storage and lifecycle of block JSON (section 5). Published and draft revisions are separate rows; publishing is a pointer move so rollback is instant. Tables: `pages` (slug, kind, published_revision_id, draft_revision_id, noindex), `page_revisions` (schema_version, body_json, created_by actor_kind user|platform|ai, validated_at, note), `nav_menus`, `redirects`, `theme` (token_set, overrides_json, fonts_json, logo_media_id).

**4.10 Media.** Uploads to R2 with signed tickets issued by the back-office; variants through the adapter's `compile` image service already in use ([astro.config.mjs:L4-L17](/home/user/catholicjoe/astro.config.mjs)) for template images and the Images binding only for owner uploads, since unique transformations are free only to 5,000 per month across all sites (`cloudflare_architecture.md`, Q4). Tables: `media_assets` (r2_key, mime, bytes, width, height, alt, license_note), `media_variants`.

**4.11 Analytics events.** First-party business events (form submit, add to cart, order paid, booking, click-to-call) written by the Worker and rolled up nightly; Cloudflare Web Analytics stays the page-view tool (`integrations_marketing_comms_booking.md`, Q1). Tables: `analytics_events` (90-day retention), `analytics_daily`.

**4.12 Scheduling.** CatholicJoe's cron flushes tracking only ([src/worker.ts:L27-L30](/home/user/catholicjoe/src/worker.ts)) and leaves email, Stripe and export retries to an owner review routine ([docs/setup.md:L127](/home/user/catholicjoe/docs/setup.md)). The `jobs` module runs every queue under a time budget: outbox (only `E_`-retryable rows), Stripe retries, exports, tracking, health checks, retention purges. The caller is the per-tenant cron below ~50 tenants, then a central scheduler Worker calling `/api/internal/jobs` with the platform token, because cron triggers cap at 250 per account (`cloudflare_architecture.md`, Q4, Q7).

---

## 5. The block schema

### 5.1 Model

A page is a versioned JSON document in `page_revisions.body_json`, validated with Zod schemas from `packages/blocks/schema`. The same Zod types generate the editor forms, the AI tool-call argument schemas and the component props, so the three cannot drift.

```ts
Page    = { schemaVersion: 1, id, slug, title,
            seo: { title?, description?, ogMediaId?, noindex, jsonLd?: 'LocalBusiness'|'Restaurant'|'Product'|'none' },
            layout: 'default'|'landing'|'narrow', sections: Section[] }
Section = { id, style: { background?, paddingY?, container?, mediaId? }, anchor?, blocks: Block[] }
Block   = { id, type: BlockType, version: number, props: <per-type schema>, visibility?: { from?, until? } }
```

Block types at launch and what they bind to: `hero` (heading, media, CTAs), `announcement_bar` (text, schedule), `text` (restricted Markdown), `cta` (buttons: link, phone, SMS, booking, order; phone from `locations`), `hours` and `map` (`locations`), `contact_form` and `quote_request` (`form_definitions`, inbox, media upload), `menu` (categories from the catalog or a `CatalogSource`, order CTA), `product_grid` (collection or rule, facets from `attribute_definitions`), `product_detail` (variant selector, fitment panel, pickup or ship, core charge), `gallery` (media), `reviews` (source `places|native|yelp_embed`, max 5, required attribution props because Google permits at most five reviews through the Places API with attribution and never scraped copies; `integrations_marketing_comms_booking.md`, Q1), `booking` (native, Calendly or Square embed), `faq`, `team`, `testimonials`, `vehicle_selector` (year/make/model), and `embed` (allowlisted providers: Instagram, YouTube, Resy button, OpenTable link).

### 5.2 Rendering and themes

`PageRenderer.astro` loads the published revision for the request path, validates it (fail closed to a 500 with a correlation id, never half a page), and maps blocks to components from `packages/blocks/components`. Components emit only the CSS variables declared by `packages/ui-themes` (`--color-*`, `--font-*`, `--radius`, `--space-*`); these are the `:root` variables and `@fontsource` choices CatholicJoe hard-codes in `global.css` and `BaseLayout.astro` (`reference_repo_teardown.md`, Q9 §2). `theme.overrides_json` is merged over the token set at render time and emitted inline, so a theme change never needs a rebuild. Pages are SSR in the Worker with `s-maxage` caching and a purge on publish; private, no-store and noindex headers for `/admin`, `/account`, cart and checkout stay in middleware ([src/middleware.ts:L25-L28](/home/user/catholicjoe/src/middleware.ts)).

### 5.3 Validation and versioning

Every block type carries `version`; `schema/migrate.ts` holds pure `migrate_<type>_vN_to_vN+1` functions applied on read by the renderer and on save by the editor, and `Page.schemaVersion` governs the envelope. Validation runs on every mutation (reject), on publish (reject) and on render (fail closed). `page_revisions.validated_at` lets the fleet upgrade job re-validate every published page against a new schema before it ships. Raw HTML is not a block; the admin UI's `innerHTML` pattern is a noted risk (`reference_repo_teardown.md`, Q10).

### 5.4 Preview of a draft

A draft is the `draft_revision_id` on a page. `/preview/<slug>?rev=<id>&t=<token>` renders that revision with the same renderer; `t` is a 15-minute HMAC over `(tenant, rev, exp)` signed with a `PREVIEW_SECRET` Worker secret, and responses are `private, no-store, noindex`. The back-office shows the preview in an iframe with a Publish button; the owner can forward the URL from a phone; the AI (section 11) hands back the same URL.

### 5.5 Designed for typed editing

The schema is small-grained so that any editor, human or model, works through a finite tool set exported from `packages/blocks/tools` and used by the back-office UI itself:

| Tool | Arguments | Effect |
|---|---|---|
| `list_pages`, `get_page` | pageId, revision draft\|published | validated Page JSON |
| `create_page`, `update_page_meta` | slug, title, layout, seo | draft revision |
| `list_block_types`, `describe_block_type` | type | schema, defaults, examples |
| `insert_section`, `move_section`, `remove_section` | pageId, index, style | draft revision |
| `insert_block`, `update_block`, `move_block`, `remove_block` | pageId, sectionId, blockId, type, props or JSON merge patch | validated; returns blockId |
| `set_theme`, `set_nav`, `set_hours` | overrides, items, hours | theme, nav, locations |
| `upload_media` | signed ticket; returns mediaId after scan | media_assets |
| `list_products`, `upsert_product` | product fields | catalog |
| `list_integrations`, `get_connect_url` | provider | a URL the owner must open; never completes auth |
| `create_draft`, `discard_draft`, `preview_draft` | pageId | revision pointer, signed preview URL |
| `publish_draft`, `rollback` | pageId, revisionId | pointer move + purge + audit |

Every call is recorded in `audit_events` with actor kind, before and after revision ids and the diff: the audit trail the AI phase needs.

---

## 6. Variant presets

A preset is data: a block allowlist, default pages (block JSON with placeholder props), an integration bundle, default settings and a default theme, versioned with the template.

| Preset | Extra blocks | Default pages | Integration bundle | Settings |
|---|---|---|---|---|
| `presence-services` | shared set (hero, text, cta, hours, map, contact_form, quote_request, reviews, gallery, team, faq, testimonials, booking, announcement_bar, embed) | Home, Services (one page per core service), About, Reviews, Contact/Quote, Hours | Web Analytics + Zaraz pixels, Google reviews, transactional email, Mailchimp OAuth, Calendly or Square Bookings OAuth, GBP when approved | catalog off, bookings native or off, checkout off |
| `food-ordering` | + menu, product_detail with modifiers | Home, Menu, Order (pickup) or order link, Catering form, Hours, Reviews | Square OAuth (menu, orders, POS), site-native ordering via Stripe Checkout, DoorDash Storefront link, Clover OAuth, Mailchimp | catalog=menu, fulfillment=pickup, checkout on once Stripe connected |
| `retail-catalog` | + product_grid, product_detail, collections | Home, Shop, Collections, Product, Cart and Checkout, Shipping and returns, Contact | Stripe Account Links, Shippo OAuth default, ShipStation key, Mailchimp or Klaviyo, Shopify or WooCommerce import, CSV import, QuickBooks | catalog=full, fulfillment ship or pickup, inventory source d1 or POS |
| `custom` | everything + vehicle_selector, fitment panel, dealer pricing (built as reusable blocks) | retail pages + Vehicle finder + Parts by category | retail bundle + POS inventory sync (Square, Lightspeed X, Clover, Shopify), accounting, custom adapters (Epicor Eagle export) | fitment on, pickup default, core charge on |

These map to the founder's four variants (`founder_vision.md`) and to the per-type day-one checklists in `business_types_ranking.md` (Q4): trades, auto repair, salons and cleaners are `presence-services`; cafes, pizza and restaurants are `food-ordering`; boutiques, gift shops and Etsy sellers are `retail-catalog`; the family store is `custom`.

**How a preset becomes a tenant.** `tooling/scaffold` takes `{preset, intake}`, where intake is the founder-confirmed public data (name, address, phone, hours, category, the Overture or Google place reference), and: inserts the `tenant` row with preset and versions; writes `locations`; seeds `pages` and a published revision per default page, substituting intake values into placeholder props (name into `hero.heading`, phone into `cta`, hours into `hours`, place id into `reviews.placeId`) with `noindex=1` and a demo banner while the site is a demo; writes settings and theme defaults; writes `connections` rows with status `suggested` for the bundle so the Apps page already shows the right tiles; generates `tenant.config.ts` and `wrangler.jsonc`; hands off to `tooling/provision`. Because this is substitution, not design, a demo costs minutes, which is what "100 demos a month" assumes (`viability_solo_founder.md`, Q10). The demo safe-harbor practices (noindex, gated, name in plain text only, no copied photos or reviews, visible "unsolicited concept" banner, takedown on request) are preset defaults, not founder discipline (`admin_dashboard_and_leads.md`, Q7 Inferences).

---

## 7. The integrations framework

### 7.1 Capability interfaces

`packages/integrations/capabilities` declares what the core calls; adapters implement one or more; core never imports a vendor SDK directly.

```ts
interface PaymentsProvider       { createCheckout(order, opts): Promise<{url}>; verifyWebhook(req): Promise<Event>; refund(paymentId, cents): Promise<Refund> }
interface CatalogSource          { listProducts(cursor?): AsyncIterable<ExternalProduct>; getStock(sku, locationId?): Promise<number>; onStockChanged?(h): void }
interface EmailMarketingProvider { listAudiences(): Promise<Audience[]>; subscribe(audienceId, contact, tags?): Promise<void>; syncOrderEvent?(order): Promise<void> }
interface ShippingProvider       { quote(shipment): Promise<Rate[]>; export(order, snapshot): Promise<{externalId}>; lookup(externalId): Promise<Shipment|null>; onTracking?(h): void }
interface OrderingProvider       { pushMenu(menu): Promise<void>; onOrder(h): void; acknowledge(orderId): Promise<void> }
interface BookingProvider        { listServices(): Promise<Service[]>; availability(serviceId, range): Promise<Slot[]>; book(input): Promise<Booking>; embedUrl?(serviceId): string }
interface ReviewsProvider        { fetch(placeRef): Promise<{rating, count, reviews: Review[], attribution}> }
interface AccountingProvider     { syncSalesReceipt(order): Promise<{externalId}>; syncRefund(refund): Promise<void> }
```

`ShippingProvider.export` plus `lookup` is the pair that lets fulfillment keep its lookup-before-retry contract: the adapter exposes the lookup, the core owns the state machine. ShipStation V2 is the first implementation, lifted from the redirect-rejecting, body-capped, no-retry transport in [src/lib/server/shipping.ts:L75-L107, L191-L228](/home/user/catholicjoe/src/lib/server/shipping.ts); a flat-rate or free-shipping provider ships alongside so a retail site can launch before any shipping account exists (`reference_repo_teardown.md`, Q9 §1).

### 7.2 Connection records and status

Every integration, OAuth or not, is a row: `connections` (provider, capabilities_json, auth_mode `oauth|hosted_onboarding|api_key|hosted_form|csv|manual`, status `suggested|pending|connected|error|revoked`, external_account_id, external_metadata_json for Square merchant and location ids, Shopify shop, QuickBooks realmId, Xero tenant, Mailchimp dc, Stripe account id; scopes_json, secret_ref, connected_by, connected_at, health_status, last_error, webhook_registration_json). Mailchimp as integrated today is three public strings copied from a hosted form ([src/data/newsletter.ts:L1-L4](/home/user/catholicjoe/src/data/newsletter.ts)); here that is a `hosted_form` connection, and OAuth to the API (audience sync from orders) is the upgrade on the same row.

### 7.3 OAuth: Nango first, in-house for the first providers

The payments note recommends Nango Cloud for auth from day one, with payment webhooks and all business logic in the Worker and an in-house port as a reversible option; Nango's `providers.yaml` encodes the per-provider quirks (Shopify shop-subdomain URLs and hourly tokens, Clover regional hosts, QuickBooks realmId, Xero tenant lookup, Mailchimp data center, Square `session=false`) and its license permits derivative works that are not resold as a hosted service (`integrations_payments_commerce_accounting.md`, Q5). Nango cloud pricing is secondary and conflicting: free tier of about 10 connections, then roughly $50 per month (unverified, 2026).

`OAuthBroker` therefore has two implementations: `NangoBroker` (connect UI session → Nango connection id in `secret_ref`) and `InHouseBroker` (`/start` writes a `pending` row and state nonce; `/callback` exchanges the code, stores tokens, runs the post-connect script to capture ids). The first providers are in-house regardless: **Stripe** via Connect Account Links, since Stripe says OAuth "is not recommended for new Connect platforms"; **Square**, a plain self-serve OAuth2 app; **Mailchimp**, no-scope OAuth2 with the data-center capture (`integrations_payments_commerce_accounting.md`, Q1, Q4; `integrations_marketing_comms_booking.md`, Q1). Nango carries the long tail: QuickBooks, Xero, Shopify import, BigCommerce, Lightspeed X, Clover, Calendly, Klaviyo, Constant Contact, Wave.

### 7.4 API key and webhook fallbacks

The dashboard needs exactly three UI patterns (`integrations_payments_commerce_accounting.md`, Q3 Inferences): **Connect** (OAuth popup or hosted onboarding); **Paste credentials** with inline verification (WooCommerce via `/wp-json/wc/v3/customers`, ShipStation V2 via `/v2/users?page_size=1`, Toast standard credentials, PayPal REST app, Helcim, Authorize.net); **Guided hand-off** (Shopify custom-app token, CSV uploads from Wix, Squarespace and BigCommerce; Pirate Ship as CSV out and tracking CSV in, `integrations_fulfillment_ordering_inventory.md`, Q1). Webhooks are registered by the engine wherever the API allows (Stripe's 13 event types; ShipStation's two V2 webhooks with a per-tenant secret header as documented in [README.md:L64-L73](/home/user/catholicjoe/README.md); Square and Shopify app-level webhooks). Where registration is manual (Clover, Mailchimp), the tile shows the URL and secret and a "test delivery" button flips the row to `connected` on receipt.

### 7.5 Per-tenant secret storage

Secrets are Worker secrets on the tenant's own Worker (`CONN_STRIPE_SECRET_KEY`, `CONN_SHIPSTATION_API_KEY`, ...); `connections.secret_ref` names the variable, never the value, matching CatholicJoe's eight-secret model ([.dev.vars.example:L3-L29](/home/user/catholicjoe/.dev.vars.example)) and per-Worker isolation (128 variables on Paid, 5 KB each; `admin_dashboard_and_leads.md`, Q10). Setting a secret is a provisioning job with the account token, never a browser action. Tokens that rotate at runtime (Shopify and QuickBooks, hourly) live in `connection_tokens` encrypted with AES-GCM under a per-tenant `CONNECTION_KEY` secret, or inside Nango when it is the broker. The Secrets Store beta (100 secrets per account, unverified) fits platform-level keys, not per-tenant credentials.

### 7.6 Webhook ingestion and idempotency

The CatholicJoe pattern, generalized to every provider:

1. Authenticate before parsing, bound the body, accept only expected resource shapes ([src/lib/server/shipstation-tracking.ts:L61-L85](/home/user/catholicjoe/src/lib/server/shipstation-tracking.ts); [src/lib/server/commerce-handlers.ts:L256-L287](/home/user/catholicjoe/src/lib/server/commerce-handlers.ts)).
2. Persist first: `INSERT INTO webhook_events (provider, external_event_id, type, resource_ref, payload_hash, status='pending') ON CONFLICT DO NOTHING`, read back the status and return early if already processed, which is `processStripeEvent` verbatim ([src/lib/server/commerce.ts:L455-L458](/home/user/catholicjoe/src/lib/server/commerce.ts)). Store resource ids rather than customer payloads when the provider offers a resource URL, as the tracking webhook does.
3. Acknowledge, then process under `waitUntil` with a budget; failures leave the row for the job runner.
4. Outbound writes go through `provider_jobs` (provider, kind, payload_json, status pending|processing|done|failed|uncertain|review, attempts, post_uncertain, claim_token, lease_until, next_attempt_at): look up before create, persist `post_uncertain` before the POST, lookup-only after an ambiguous result, `review` on duplicates, exactly as `flushShipStation` ([src/lib/server/shipstation-fulfillment.ts:L104-L200](/home/user/catholicjoe/src/lib/server/shipstation-fulfillment.ts)). Stripe's Connect webhook is platform-level, one endpoint for every connected account with an `account` field (unverified, last known 2025), so the per-tenant work is only the account lookup; Square and Shopify webhooks route by merchant or shop id the same way.

Tables: `webhook_events`, `provider_jobs`, `connection_tokens`, `connection_health`.

### 7.7 Health checks and the app menu

Each adapter implements `health(connection)`, a cheap authenticated read (Stripe account, Square locations, ShipStation `/v2/users`, Mailchimp ping), run hourly by the jobs module into `connection_health`. The tenant Settings JSON already reports `integrations {stripe, mailchimp, shipstation, email, database, auth}` plus queue counts ([src/pages/api/admin/[...path].ts:L105-L132](/home/user/catholicjoe/src/pages/api/admin/[...path].ts)); that endpoint stays the one call the founder dashboard polls per tenant (`reference_repo_teardown.md`, Q5 Inferences).

`/admin/apps` renders tiles from `registry.ts`: name, category (Payments, Orders and POS, Shipping, Email and marketing, Bookings, Reviews and listings, Accounting, Analytics, Automation), auth mode, gating notice, status chip, "what this does for your site", and the single action the auth mode implies. Preset-bundle tiles are pinned on top. A tile's detail view shows health, last webhook, the blocks that reference the connection, and a disconnect that revokes the token, deletes secrets via a job and marks dependent blocks as needing attention rather than breaking the page.

### 7.8 First integrations to ship

| Wave | Integrations | Why |
|---|---|---|
| 0 (every site) | Cloudflare Web Analytics, Zaraz-managed GA4 and Meta Pixel (paste an id), Google reviews via Places API (New) with attribution, transactional email | zero gating (`integrations_marketing_comms_booking.md`, Q3) |
| 1 (in stack today) | Stripe via Account Links, Mailchimp OAuth, ShipStation V2 key | already in production |
| 2 (self-serve, highest value) | Square OAuth (catalog, orders, inventory, bookings, webhooks), Shippo OAuth, Calendly OAuth, Shopify custom-app import, WooCommerce key, CSV import, QuickBooks OAuth | one Square grant covers food, retail and salons (`integrations_fulfillment_ordering_inventory.md`, Q5; `integrations_payments_commerce_accounting.md`, Recommended first six) |
| 3 (apply now, ship when approved) | Google Business Profile, Clover OAuth, Meta Instagram feed and Messenger, Xero | weeks-long gates (`integrations_marketing_comms_booking.md`, Q4) |
| Deferred | PayPal partner referrals (ship as paste-credentials), Toast partner API (paste standard credentials), DoorDash Drive (sandbox now, request production with the first live restaurant), OpenTable and Resy (buttons), Twilio SMS (10DLC wizard later), any marketplace listing | human-gated; sequence after the first two live sites (`integrations_fulfillment_ordering_inventory.md`, Q5) |

---

## 8. The two admins

### 8.1 Founder dashboard (`apps/founder-dashboard`)

A separate Worker at `admin.<brand>.com` behind Cloudflare Access (free to 50 users, unverified; `cloudflare_architecture.md`, Q5) with its own platform D1. Nothing off the shelf runs on Workers plus D1 and hosts the deploy button, so it is built in the monorepo reusing `AccountLayout`, the step-up MFA and the retry panels (`admin_dashboard_and_leads.md`, Q9).

- **Discovery import.** Imports a metro slice of Overture places filtered to phone, no website, open, confidence ≥ 0.5 (measured 8,144 rows for Jacksonville; `no_website_discovery_pipeline.md`, Q2), tags Facebook-only rows, runs the verification ladder (E.164 normalisation, domain probe, search check, then a Google Place Details `websiteUri` call by place_id), and stores only what the terms allow: `place_id` indefinitely, Google lat/lng for 30 days, never Google names or addresses as a directory (`admin_dashboard_and_leads.md`, Q1). A Florida Secretary of State daily-file join flags new storefronts (`no_website_discovery_pipeline.md`, Q5).
- **CRM.** `accounts`, `channel_listings` (one row per Google place, Overture row, Etsy, Amazon or eBay store, website; the dedup backbone), `contacts` with consent flags, `deals` with stages `lead → demo_built → demo_sent → call → paid → live → managed|handed_off|lost`, append-only `activities`, tags, FTS5 search; schema as drafted in `admin_dashboard_and_leads.md` (Q8).
- **Pipeline.** Kanban by stage with rules in code: `demo_built` needs a `sites` row in `preview`; `paid` needs a subscription set by the Stripe webhook; `live` needs `hostname_status` and `ssl_status` both `active`.
- **Fleet.** One row per tenant: script name, preview URL, hostname and SSL chips, template and schema versions, last deploy, and the hourly health snapshot pulled from each tenant's `/api/admin/settings` with the platform token (ready to ship, exports in review, held emails, failed jobs, unhealthy connections). "Tenants behind version X" feeds the upgrade job.
- **Deploy and go-live jobs.** `deploy_jobs` with actions `build_demo | deploy | migrate | upgrade | go_live | attach_domain | rotate_secret | suspend | delete`, run as Cloudflare Workflows so a go-live can sleep for days while a client fixes DNS (`admin_dashboard_and_leads.md`, Q10 Inferences).
- **Billing.** The founder's own Stripe (`subscriptions`, `invoices`), never mixed with a tenant's Stripe.
- **Support access.** "Open tenant back-office" works only while that tenant's `platform_access.platform_enabled=1`; every such session is audited in the tenant's `audit_events`.

Who sees it: the founder, later a second person with a `sales` or `viewer` role via better-auth's admin plugin; never clients.

### 8.2 Tenant back-office (`apps/site` under `/admin`)

Grown from CatholicJoe's Overview, Orders, Inbox and Settings, with the 947-line vanilla-TS client ([src/scripts/account.ts](/home/user/catholicjoe/src/scripts/account.ts)) replaced by server-rendered tables and a few islands, as the teardown and dashboard note both advise (`reference_repo_teardown.md`, Q5 Inferences; `admin_dashboard_and_leads.md`, Q11).

| Screen | Shows | owner | staff | developer (while enabled) | platform token |
|---|---|---|---|---|---|
| Overview | today's orders, bookings, unread inquiries, setup notices, queue issues | yes | yes | yes | read |
| Orders | list, detail, fulfillment actions, refunds, tracking refresh, manual shipping fallback | yes | yes | yes | read |
| Products and inventory | catalog CRUD, CSV import, variants, attributes, stock by location, fitment import | yes | yes | yes | read |
| Hours and locations | hours, special hours, pickup slots | yes | yes | yes | no |
| Pages | block editor, drafts, preview, publish, rollback, nav, theme | yes | no | yes | no |
| Inbox | threads, replies, notes, claims | yes | yes | yes | read |
| Bookings | calendar, services, availability, staff | yes | yes | yes | read |
| Apps | tiles, connect, paste credentials, health, webhooks | yes | no | yes | read health |
| Settings | team and roles, domain, email sender, tax and shipping, checkout on/off, platform and developer access switch, retries | yes | no | partial (no team, billing or switch) | read, run retries |

The owner-only access switch generalizes `setDeveloperAccess` ([src/lib/server/admin-access.ts:L15-L35](/home/user/catholicjoe/src/lib/server/admin-access.ts)): one audited toggle for the developer and one for the platform, both revocable on the next request.

---

## 9. Remote onboarding flow

The published WaaS playbook (intake → proposal → build → finalise → test → launch → maintain) with the build step replaced by data substitution (`viability_solo_founder.md`, Q7). Target: under 8 founder-hours per client.

1. **Intake.** A public `/start` form or the founder entering a prospect from discovery: name, category (selects the preset), address, phone, hours, existing presence, what they sell or do, which tools they already run (Square, Toast, Booksy, Mailchimp, Shopify). Writes `accounts` and `deals(stage='lead')`.
2. **Demo build.** `build_demo` job: scaffold from the preset and the confirmed public data, provision `site-<slug>` and its D1, deploy, map `<slug>.preview.<brand>.com` in the router KV, gate with Access one-time PIN or a signed link, set the demo banner and noindex. Stage → `demo_built`, then `demo_sent` when the Loom or email goes out with the CAN-SPAM footer and suppression check baked in (`admin_dashboard_and_leads.md`, Q7).
3. **Review call.** Screen share over the preview; edits made live in the block editor. Package agreed; Stripe subscription link sent; stage → `paid` on the webhook.
4. **Connect integrations.** The owner is invited as `owner` (the provisioned-owner password flow is already tested, [tests/auth-and-access.test.ts:L238](/home/user/catholicjoe/tests/auth-and-access.test.ts)), opens `/admin/apps` and clicks Connect on the pinned tiles. Stripe's isolation rule is in the tile copy: an account controlled by another platform must create a new Standard account (`integrations_payments_commerce_accounting.md`, Q1 Inferences).
5. **Content collection.** The critical path is assets, not design: photos, a services or menu list with prices, licence numbers, team names. The back-office offers mobile upload via signed R2 tickets, a menu or services spreadsheet importer, and the per-preset checklist from `business_types_ranking.md` (Q4). Missing items block publish only for required blocks.
6. **Go-live with Cloudflare for SaaS.** `go_live` job: `POST /zones/{saas_zone}/custom_hostnames` for `www.<domain>` with `ssl: {method: "http", type: "dv"}`; show the owner the single CNAME (`www → sites.<brand>.com`) and an apex redirect or flattening; poll until `status` and `ssl.status` are both `active`; write `hostname → site-<slug>` to the router KV; update `tenant_domains` and the canonical host; drop the banner and noindex; purge. First 100 hostnames free, then $0.10 per month each; apex proxying is Enterprise-only, so `www` is canonical (`cloudflare_architecture.md`, Q2; `admin_dashboard_and_leads.md`, Q10). Stage → `live`.
7. **Maintain.** Health and queues roll up to the Fleet screen; the owner edits in the back-office; template upgrades arrive through the fleet job.

**Where an in-person visit slots in.** Steps 3 and 5 are the only ones that benefit from presence: the review call becomes a visit, and content collection becomes the founder taking photos and typing the menu at the counter. The visit writes to the same intake, blocks and Apps page, so a business 1,000 miles away gets the identical product.

---

## 10. Deployment model

**Now: one Worker and one D1 per tenant (Pattern B).** Each tenant is a standard Worker `site-<slug>` with `DB` (D1 `site-<slug>`), `MEDIA` (R2), `EMAIL` where used, static assets with `run_worker_first` for the canonical redirect ([wrangler.jsonc:L11](/home/user/catholicjoe/wrangler.jsonc)), `nodejs_compat` and per-tenant secrets. The Cloudflare note recommends this for the first ~100–400 sites on Workers Paid ($5 per month, 500 Workers per account), with infrastructure around $5–10 per month at 10 sites and $25–50 at 200 (`cloudflare_architecture.md`, Q1, Q8). The template carries no account id, database id, product id or live flags, unlike [wrangler.jsonc:L7, L20, L28-L37](/home/user/catholicjoe/wrangler.jsonc).

**Routing and preview.** A router Worker owns `*.preview.<brand>.com/*` and the SaaS fallback origin `sites.<brand>.com` (AAAA `100::`, route `*/*`), resolves `hostname → script` from KV and forwards through a service binding (B) or `env.DISPATCHER.get(name)` (C). The wildcard avoids the 100-custom-domains-per-zone cap and the DNS record limit (`cloudflare_architecture.md`, Q3, Q7). Preview hostnames carry noindex and Access.

**Later: Workers for Platforms (Pattern C).** Move at ~400 Workers, when a tenant needs CPU or subrequest caps, or at the first client-supplied code. Build output and bindings are identical; only the deploy command (`--dispatch-namespace`) and the router change; $25 per month with 1,000 scripts included (`cloudflare_architecture.md`, Q1). Namespaced Workers lose `caches.default` and `request.cf` in untrusted mode, so edge caching of published pages moves into the router.

**Migrations across a fleet.** Core migrations are numbered SQL in `packages/core/migrations`, additive within a major (destructive changes wait for a major and ship with a data-move script). Each tenant D1 has `schema_migrations`; `tooling/migrate-fleet` lists tenants from the platform D1, applies pending migrations per tenant, records `sites.schema_version` and a `deploy_jobs` row, and stops on the first failure. Waves: the founder's own site and the family store first, then 10%, then the rest. Tests load every migration in order, as [tests/admin-access.test.ts:L25-L37](/home/user/catholicjoe/tests/admin-access.test.ts) already does by reading the directory (`reference_repo_teardown.md`, Q6 Inferences).

**Template pinning and fleet upgrades.** A template version is a git tag; CI builds `apps/site` once per tag into R2. `sites.template_version` pins each tenant. An upgrade is a `deploy_jobs(action='upgrade')` per tenant: re-validate every published page against the new block schema, apply migrations, upload the bundle with existing bindings and secrets, run `tooling/smoke` (public 200s, `/admin` redirect, webhooks return 400 on garbage), roll back on failure. Migrate-on-read lets a tenant lag a version without a broken page. A 200-site rebuild is 200–400 Workers Builds minutes against 6,000 included (`cloudflare_architecture.md`, Q7).

**Handoff.** Each tenant is a D1 export, a pinned template and a secret list, so the three handoff models in `cloudflare_architecture.md` (Q6) all reduce to redeploying into the client's account; domains are registered in the client's name from day one so an exit is a CNAME change.

---

## 11. Path to the AI self-serve builder

A later phase (the viability plan's month 10–12 decision; `viability_solo_founder.md`, Q10), listed so the engine is built to allow it without rework.

**What it looks like.** A chat panel in `/admin/pages` and `/admin/apps`. The owner types "add a Saturday brunch section with three items and put a reserve button in the hero". The model receives the page list, block catalog, theme tokens and a short business profile, and responds only with tool calls from section 5.5 (`get_page`, `insert_section`, `insert_block{type:'menu'}`, `update_block{hero.ctas}`, `preview_draft`). The owner sees the preview inline and clicks Publish; the model never publishes. "Connect my Square" yields `get_connect_url('square')`, rendered as a button; the model never touches tokens.

**Guardrails.**
- Schema validation: every call is Zod-validated server-side on the same path the UI uses; invalid calls return the error to the model and write nothing.
- Draft only: the model edits drafts and requests previews; `publish_draft`, `rollback`, theme sets beyond overrides, and anything under Settings or Apps are owner clicks.
- Content policy: only `media_assets` the tenant uploaded, no external image URLs; the `reviews` block must point at a connected `ReviewsProvider`, the model cannot author review text; regulated fields (licence numbers, health claims, bar-rule testimonials) are marked `owner_confirmed_required` and block publish until ticked.
- Cost limits: per-tenant daily token budget and per-turn tool-call cap in `tenant_settings` under a platform ceiling, with spend shown in the founder dashboard.
- Audit and rollback: every call writes `audit_events` with actor `ai`, prompt hash and revision diff; rollback is one click.
- Isolation: the model runs in a separate Worker with a tenant-scoped token carrying only page and media scopes; no orders, customers, secrets or other tenants.

**What must be true in the engine first.** The block schema is strict, versioned and expressive enough that any sensible page needs no raw HTML; all mutations already flow through the typed tool set the human editor uses; preview is a signed URL for a specific revision; integrations connect only through a URL handoff; `audit_events` capture before and after revisions; limits and budgets are settings, not constants; and presets give the model good defaults so the first prompt is "adjust", not "create from nothing".

---

## 12. Migration and extraction plan from CatholicJoe

The teardown estimates ≈44–60 engineering days for a first platform release (core 14–18, config layer 3–4, template and scaffolder 5–7, provisioning 5–7, founder dashboard 12–18, tests and CI 3–4, runbooks 2) and ≈10–14 days for an interim "clone + config + provisioning script" path (`reference_repo_teardown.md`, Q9 §7). This plan keeps those numbers and adds what the teardown did not scope: hardening, the block schema and the integrations framework. Days are solo engineering days; sales time is separate.

| Phase | Scope | Days | Reconciliation |
|---|---|---|---|
| 0. Harden before cloning | nonce CSP and Turnstile on public POSTs; cron flushes outbox, Stripe retries and exports under budgets; strip committed ids and default checkout off; `apiError` correlation id; GitHub Actions deploy; delete the form-checkout path; retention job for quotes, claims and outbox bodies | 4–6 | Q10's list at 0.5–1.5 days each |
| 1. `packages/core` extraction | straight copy of `security.ts`, `database.ts`, middleware, shipping transport, export and tracking state machines, outbox, inbox, admin access, migrations 0001–0007 with the two CHECK enums relaxed, the Miniflare harness; `TenantConfig` threading through the ~40 literals; `memberships` and `platform_access`; `EmailSender` interface; Astro integration and `createWorker(tenantConfig)` | 14–18 | Q9 §1 as written, catalog generalization moved to phase 5 |
| 2. Blocks, themes, presets | Zod page schema with migrate-on-read; the 19 block components; `PageRenderer`; tokens extracted from `global.css` and `BaseLayout.astro`; draft, preview token, publish; typed tool set; the four presets' default pages, `presence-services` complete | 10–14 | new; subsumes the teardown's content-collections item (Q9 §2, 3–4 days) because the block model is also the AI substrate |
| 3. Integrations framework, waves 0–1 | capability interfaces; `connections`, `webhook_events`, `provider_jobs`, `connection_health`; in-house Stripe Account Links, Square OAuth, Mailchimp OAuth; ShipStation key with verification; Nango broker wrapper; Apps page with the three patterns; health job; reviews and analytics blocks wired | 10–14 | new; Stripe and ShipStation webhook code is lifted unchanged; OAuth for ten providers is a few hundred lines plus quirks (`integrations_payments_commerce_accounting.md`, Q5) |
| 4. Site template, scaffolder, provisioning | `apps/site` template with generated config; `tooling/scaffold`; `tooling/provision` (D1 create, migrate, secrets, deploy, preview KV, Stripe and ShipStation webhook registration, custom hostname create and poll); router Worker; smoke tests | 10–14 | Q9 §3 (5–7) plus §4 (5–7) |
| 5. Catalog and checkout generalization | D1 catalog tables, variants, images, attributes, inventory levels; N-line attempts, shipment body and match; flat-rate shipping provider; Products screen with CSV import | 6–8 | Q9 §1 catalog (4–5) plus shipping interface (2) |
| 6. Founder dashboard | platform D1 and CRM; Overture import with the verification ladder and place_id verify; pipeline; fleet view; `deploy_jobs` as Workflows; billing via the founder's Stripe; Access in front | 12–18 | Q9 §5 as written |
| 7. Tests, CI, runbooks | fleet-wide "load every migration" harness, Square and Nango fixtures, Actions matrix for template builds and upgrades, runbooks for provisioning, go-live, handoff, retries | 5–6 | Q9 §7 (3–4) plus runbooks (2) |
| **First platform release** | phases 0–7 | **71–98** | teardown 44–60 plus phases 0, 2 and 3 |
| 8. Custom tier | `vehicles` and `fitments`, VCdb loader, year/make/model selector and fitment panel, core charges and pickup rules, `CatalogSource` adapters for Square, Lightspeed X, Clover, Shopify; Epicor Eagle export as a custom job | 10–15 | funded by the family store; depends on its POS (`integrations_fulfillment_ordering_inventory.md`, Q4 Gaps) |
| 9. Food preset depth | modifiers, pickup slots, Square menu and order sync, DoorDash Drive sandbox, Clover order sync | 8–12 | after a paying restaurant exists (`viability_solo_founder.md`, Q4) |
| 10. AI builder | chat Worker, tool-call runtime, budgets, audit UI, 50-prompt evaluation set | 15–25 | later phase; section 11 |

Ordering follows the viability plan: months 1–3 sell with the presence preset and config-only onboarding; months 4–6 run discovery and demos; months 7–9 add the retail variant with OAuth apps (`viability_solo_founder.md`, Q10). So phases 0, 1, 2 (presence preset only) and the scaffold half of phase 4 come first, with integrations limited to wave 0–1 tiles, before the founder dashboard; the CRM can be a spreadsheet for the first ten accounts, but the per-tenant engine cannot.

**The first thing to build** is phase 1's `TenantConfig` threading plus the `presence-services` preset rendered from block JSON, deployed by a scaffold script to a `*.workers.dev` URL: the teardown's interim path (10–14 days) extended by the minimum of phase 2, roughly 18–24 days. It is the smallest artifact that proves "never rebuild the engine per client": two presence sites for two local businesses from one codebase, plus CatholicJoe itself re-expressed as a `retail-catalog` tenant with a one-SKU catalog, which keeps every existing test green and makes the founder's own production site the first fleet member.

---

## 13. Open questions and decisions

1. **Hosting account model.** Managed on the founder's account via SaaS hostnames (Model 1) by default, or deploy into the client's account (Model 2) from the start? It changes the platform role, billing and the Email Sending domain constraint (`cloudflare_architecture.md`, Q6; `reference_repo_teardown.md`, Q9 Gaps).
2. **Transactional sender.** Keep Cloudflare `send_email` (3,000 per month included on Paid, but sending domains must sit in the founder's account) or standardize on Resend with idempotency and a founder-owned sending domain.
3. **Stripe model.** Connect Standard with Account Links and optional application fees, or each tenant's own keys pasted in as today? Connect is the cleaner OAuth story and allows one platform-level webhook; it changes who pays disputes (`integrations_payments_commerce_accounting.md`, Q1).
4. **Nango or in-house for the long tail.** Nango's 2026 prices are unverified and self-hosting is described as Enterprise-only; depend on the free tier, pay ~$50 per month, or port `providers.yaml` quirks after the first three providers.
5. **Presets at launch.** Presence only, or presence plus retail since CatholicJoe is retail? The viability note argues one variant for 12 months; the family store argues for the custom tier in parallel.
6. **R2 layout.** One bucket per tenant (clean export and deletion) or a shared bucket with prefixes (fewer bindings, simpler provisioning)?
7. **FTS5 in tenant D1.** Needed above a few hundred products but it blocks D1 export; accept drop-and-rebuild around exports or keep search in a separate D1 (`cloudflare_architecture.md`, Q4).
8. **When to adopt Workers for Platforms.** At ~100 preview domains without the router, at ~400 Workers, or at the first client-supplied code.
9. **Central scheduler timing.** Per-tenant cron until ~50 tenants, or central from day one so the 250-cron cap never surprises.
10. **The family store's POS.** The custom tier depends on whether the store runs Epicor Eagle, Vision or a jobber system and whether its distributor supplies ACES/PIES files; VCdb licensing (about $2,500–$4,400 per year, unverified) must be in the store's name (`integrations_fulfillment_ordering_inventory.md`, Q4).
11. **Demo gating.** Access one-time PIN (prospect enters an email) or a signed link (one click, less protection) for `*.preview.<brand>.com`.
12. **Brand.** The npm scope, SaaS zone and preview domain wait on the name (`naming.md`); the name also sets the Google Business Profile that must age 60 days before the GBP API application (`integrations_marketing_comms_booking.md`, Q4).
13. **AI vendor and budget**, and whether the AI phase is a paid add-on; decide before the block tool set is frozen.
14. **Remnants.** This plan deletes the form-checkout routes and the 410 newsletter endpoints from core; confirm nothing external still calls them.
