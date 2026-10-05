# HallForge — engagement and operations modules

Read-only review of `/home/user/hallforge` (shallow clone of github.com/mobius29er/hallForge). Scope: `src/modules/{newsletter,calendar,events,members,recognitions,programs,people/*}`, the matching `src/components/*` and `src/app/*` surfaces, `src/server/email`, and the migrations those modules own. Every claim is cited as `path:Lstart-Lend` relative to the repo root. "Finished" below means: schema + RPC + TypeScript wrapper + dashboard UI + tests all exist and nothing in the path is gated off or stubbed.

Codebase shape worth knowing before the per-module findings: every module is `contracts.ts` (zod) + `data/rpc.ts` (thin wrapper over a Postgres RPC) + `index.ts` (the only import surface), with all authorization and visibility rules in SECURITY DEFINER SQL functions, never in TypeScript. There are 200 vitest files under `src/` and 39 pgTAP files under `supabase/tests` (counts from `find`). The code comments are unusually thorough and honest about what is not built; most of the "gaps" below are stated in the code itself.

---

## 1. Newsletter

### Takeaway

The "newsletter" is **not an email product**. It is a paged, print-to-PDF publication (US Letter, 612×792pt) assembled from the council's records and the editor's prose, saved as a design document, and turned into a PDF by `window.print()` in Chrome. There is no composition-to-email, no list management, no unsubscribe, no double opt-in, and no sending of any kind in this module. The decision record explicitly chose this ("a render target, not an editor") and deferred the email "digest" stage. What exists is finished and well-tested for what it is; the email newsletter the founder may be imagining does not exist.

### Findings

**What an issue is.** An issue is a design document of kind `newsletter` stored in the same table and behind the same four RPCs as the studio's flyers, rendered by "paged DOM printed by the browser, not a Konva canvas" (`src/modules/newsletter/index.ts:L1-L30`; `src/modules/newsletter/data/rpc.ts:L13-L27`). Saving always sends `p_kind = 'newsletter'` through `save_design_document` (`src/modules/newsletter/data/rpc.ts:L242-L271`); listing filters the whole design table down to that kind (`src/modules/newsletter/data/rpc.ts:L155-L176`).

**Document shape.** `issue` (period window, volume/number, fraternal year start) + `standing` (masthead line, emblem/flag/watermark asset ids, patron quote, website, links) + an ordered `sections[]` array, max 40 sections (`src/modules/newsletter/contracts.ts:L36-L40`, `L104-L135`, `L155-L182`, `L418-L426`). Eleven section types: `cover`, `letter`, `outcome_report`, `recognitions`, `news_bits`, `programs`, `officers`, `assembly_officers`, `story`, `prayer`, `links` (`src/modules/newsletter/contracts.ts:L372-L384`). Slot sections carry editor prose as the page editor's TipTap/blocks-v1 rich-text document (no HTML in the document, same stored-XSS line as pages, `L30-L33`); record sections carry only a heading and a filter, and rows are "never stored in the document — an issue reopened a year later shows today's roster, which the record accepts until a publish step exists to freeze them" (`src/modules/newsletter/contracts.ts:L14-L34`). All schemas are `.strict()`.

**Row loading.** `loadNewsletterRows` reads four *publication* projections — offices, programs, recognitions, prayer intentions — only through those modules' `index.ts`, with the signed-in user's own client so the database gates decide; a forbidden source is reported as `withheld` with the capability name so the renderer prints a "Withheld" marker page instead of a blank one (`src/modules/newsletter/data/rows.ts:L34-L54`, `L89-L96`, `L98-L112`, `L283-L320`). Prayer intentions are the one windowed read (the whole list never leaves the database for someone who cannot manage people) (`L249-L272`).

**Rendering.** `NewsletterRenderer` is a pure React function (no hooks, no fetch, no `'use client'`) so the server print route, the client live preview and the tests render the same tree; every page is a fixed, clipped 612×792pt box with `@page { size: letter; margin: 0 }` (`src/modules/newsletter/render/NewsletterRenderer.tsx:L43-L66`, `L127`; `src/modules/newsletter/render/styles.ts:L1-L42`). Pagination, selection by fraternal year / window / award kind, table of contents and page budgets are pure functions in `src/modules/newsletter/domain/resolve.ts:L1-L20`, `L255-L330`. Two typefaces: bundled Norwester (OFL) for display and Montserrat via `next/font/google` for body (`src/modules/newsletter/fonts.ts:L29-L41`).

**Composition UI.** Click-to-edit forms on the left, scaled live preview on the right, autosave, add/move/remove sections from a catalog (`src/components/dashboard/newsletters/NewsletterEditor.tsx:L42-L60`; `src/components/dashboard/newsletters/section-catalog.ts:L15-L41`). The editor is a separate dashboard route from the Konva design studio (`src/app/(dashboard)/dashboard/newsletters/[id]/page.tsx`).

**Output.** The print route `/dashboard/newsletters/[id]/print` self-authorizes (sign-in, chosen council, `newsletter.issues` entitlement, then the RPC's `content.edit_any` gate) and renders the issue plus one button (`src/app/(print)/dashboard/newsletters/[id]/print/page.tsx:L69-L76`, `L108-L116`). The button is literally `window.print()` after waiting for fonts and image decodes, labelled "Save as PDF", with a note to use Chrome/Edge with background graphics on (`src/components/dashboard/newsletters/PrintControls.tsx:L7-L39`, `L64-L73`).

**Sending provider.** None in this module. The dependency list has no email SDK at all — no `resend`, `postmark`, `@sendgrid/*`, `nodemailer`, `mailchimp`, `brevo` (`package.json:L31-L68`). The only Resend usage in the repo is two raw `fetch("https://api.resend.com/emails")` calls for transactional notifications (see §7). `grep -rniE "unsubscribe|opt-in|double.opt"` over `src/` finds no implementation, only design-document prose.

**Plan gate.** `newsletter.issues` is a Starter-tier feature (`src/modules/entitlements/plans.ts:L89`).

**Stated roadmap.** The decision record splits newsletters into Stage 1 "digest — an auto-compiled email of upcoming events, new pages, recent recognitions, sent to contacts opted into the `news` purpose" and Stage 2 the designed issue, and says email infrastructure (provider, per-recipient delivery records, unsubscribe tokens, SPF/DKIM/DMARC) must move much earlier because it is "a platform cost, not a newsletter cost" (`docs/decisions/2026-08-19-newsletter-as-render-target.md:L58-L66`). The September build plan records `messaging` as "Foundation built … *Remaining: digests, member email, newsletters*" and `newsletter` as "**Nothing**" at the tool level (the issue editor landed after that note) (`docs/launch/2026-08-20-module-build-plan.md:L22`, `L30`, `L82`, `L105`).

### Gaps

- No email newsletter, digest, campaign, or send of any kind; no subscriber list; no unsubscribe link/token; no double opt-in. `grep` for `digest` in `src/` matches only crypto hashing.
- No publish/freeze step: an issue re-renders from live rows every time, so a printed issue cannot be reproduced later if the roster changes (`src/modules/newsletter/contracts.ts:L26-L29` says so).
- No public issue archive or web rendering of an issue — only the authenticated print route.
- PDF generation depends on the user's browser; there is no server-side PDF (no Puppeteer/Chromium worker). The UI tells the officer to use Chrome.
- Section vocabulary is Knights-shaped (`assembly_officers`, `prayer`, fraternal year, pillars) even though headings are left blank for the orgtype layer to fill.

---

## 2. Events and registration

### Takeaway

The event *database* is enterprise-grade and finished (events, occurrences, ticket types with inventory, form fields, registrations, line items, check-in, Stripe payment orders/attempts/ledger/refunds, all with constraint-level invariants). The event *admin* is finished for a plain calendar entry (title, times, timezone, venue, publish/cancel). Everything a visitor would use to *register or pay* is deliberately disabled: the public registration endpoint returns HTTP 503 unconditionally, the checkout component targets a retired route group that the middleware 404s, there is no UI to create ticket types or set capacity, and Stripe checkout and webhook ingestion are behind `false`-by-default server flags with release gates still open. No ICS feed exists.

### Findings

**Data model (migration 003).** Tables: `events` (slug per site, status draft/published/canceled/completed/archived, registration window, default currency, optional `payment_account_id`), `event_occurrences` (one event, many dated occurrences, each with `capacity`, venue, `private_virtual_url`), `event_ticket_types` (price in minor units, `quantity_total/reserved/sold` with a CHECK that reserved+sold ≤ total, min/max per registration, `members_only`, sales window), `event_form_fields`, `event_registrations`, `event_registration_line_items`, `event_registration_answers`, `event_registration_state_events`, `registration_check_in_events`, then `payment_orders`, `payment_attempts`, `payment_provider_events`, `payment_provider_event_processing`, `payment_ledger_entries`, `payment_refunds`, `payment_refund_state_events` (`supabase/migrations/003_events_relationships.sql:L298-L347`, `L349-L383`, `L385-L428`, `L430-L917`). Functions include `create_event_registration`, `release_event_reservation`, `begin_payment_attempt`, `attach_payment_session`, `ingest_payment_provider_event`, `complete_payment_from_provider_event`, `transition_registration_check_in`, `request_payment_refund`, `apply_refund_provider_event` (`L1195-L2096`). These writes were service_role-only, which migration 012 notes meant "list_public_event_occurrences has always returned nothing, because nothing could ever put a row in front of it" (`supabase/migrations/012_calendar_event_administration.sql:L5-L9`).

**Event admin (migration 012 + calendar/admin).** `create_event`, `update_event`, `set_event_status`, `list_events_for_management`, `get_event_for_management`, gated `events.manage`; they "never touch tickets, registrations, or money, and they refuse to operate on an event that has taken a registration" (`supabase/migrations/012_calendar_event_administration.sql:L11-L13`, `L167-L172`). The officer form is flat (slug, title, summary, description, start/end as `datetime-local`, timezone, venue, virtual flag) with correct zone-aware instant conversion (`src/modules/calendar/admin/contracts.ts:L42-L59`, `L135-L155`). There is **no capacity, ticket, price or form-field input** anywhere in the form or RPC arguments (`src/modules/calendar/admin/rpc.ts:L13-L29`; `grep capacity|ticket src/components/calendar/event-form.tsx` → nothing). The dashboard event page shows a registration count and a Full-plan upgrade panel for `calendar.registration` / `payments.checkout` (`src/app/(dashboard)/dashboard/events/[id]/page.tsx:L70-L86`).

**Registration — stubbed.** `POST /api/registrations/create` returns 503 `REGISTRATION_REBUILD_IN_PROGRESS` for everybody, with the comment "The prototype registration endpoint trusted browser-owned tenant, price, and payment fields. Keep the public surface fail-closed until the clean v1 event service, inventory transaction, and abuse controls replace it" (`src/app/api/registrations/create/route.ts:L3-L28`). The request/response zod contracts are written (`src/modules/events/registration/contracts.ts:L37-L86`) and the `checkout_required` result shape carries a Stripe `checkoutUrl`.

**Checkout UI — stale prototype.** `EventCheckout.tsx` is a three-step ticket/details/confirmation dialog that posts to that 503 endpoint (`src/components/events/EventCheckout.tsx:L1-L4`, `L158`). Its `EventTicket` interface (`compare_at_price`, `min_per_order`, `is_visible`) does not match the `event_ticket_types` columns (`L34-L47` vs `003:L385-L428`), and the only page that mounts it queries a table named `event_tickets` that no migration creates (`src/app/(public)/[orgSlug]/events/[eventSlug]/page.tsx:L79-L85`, `L305-L308`; `grep event_tickets supabase/migrations` → none). The middleware explicitly 404s that whole `/[orgSlug]/...` route group as "retired … reads mutable legacy tables" (`src/middleware.ts:L321-L327`). Treat `src/components/events/` and `src/app/(public)/[orgSlug]/` as dead code.

**Payments — implemented, disabled.** `stripe@22.3.2` is a dependency (`package.json:L62`). The application boundary (begin attempt → checkout context → Stripe Checkout create → attach session, idempotent, never trusting browser prices) and the Connect webhook (exact-bytes signature check, allowlist of 5 event types, durable ingest before ack) are implemented but "not approved for live paid registration"; remaining release gates include running the leased provider-event worker and the expiry/reconciliation worker, abuse controls, Connect account validation, and end-to-end sandbox tests (`src/server/payments/orchestration/README.md:L1-L4`, `L11-L29`, `L55-L65`). Both workers exist but "No scheduler is installed by this module" (`src/server/payments/processing/README.md:L17-L24`). Flags `HALLFORGE_PAID_REGISTRATION_ENABLED` and `HALLFORGE_STRIPE_WEBHOOK_INGESTION_ENABLED` default off and any value other than `true`/`false` is a hard config error (`src/server/payments/config.ts:L29-L80`; `.env.example:L26-L29`). `wrangler.jsonc` has no cron triggers.

**Public event pages.** Live route group is `src/app/(public)/site/events/` (list) and `site/events/[slug]/` (detail) rendering from the anonymous snapshot client with schema.org `Event` JSON-LD (`src/app/(public)/site/events/[slug]/page.tsx:L7-L20`, `L102`; `src/modules/calendar/domain/structured-data.ts:L14-L58`). There is no registration control on the live detail page. Public ticket types are readable through `list_public_event_ticket_types` (tenant-state hardened in `supabase/migrations/014_public_ticket_types_tenant_state.sql:L1-L40`) but nothing in the live UI calls it.

**Calendar feeds.** No ICS/iCal/webcal anywhere: `grep -rnE "text/calendar|\.ics\b|VCALENDAR|BEGIN:VEVENT|webcal" src supabase docs` → empty.

**Plan gates.** `calendar.events` is Starter; `calendar.registration` and `payments.checkout` are Full (`src/modules/entitlements/plans.ts:L82`, `L93-L94`).

### Gaps

- Online registration: stubbed (503). No free-RSVP path either.
- Ticket types / capacity / registration questions: schema exists, **no admin UI or RPC** to create them from the dashboard.
- Stripe: code exists, flags off, no scheduler for the two required workers, open release gates per the README.
- No confirmation email to registrants anywhere (`grep -rniE "confirmation|registrant" src/server` → nothing).
- No ICS export/feed; no "add to calendar" links.
- Legacy `EventCheckout` + `[orgSlug]` pages are dead and mismatched with the schema; would need deletion, not reuse.
- Recurrence: schema allows many occurrences per event, but `create_event` creates exactly one and the admin has no recurrence UI (`012:L73-L77`).

---

## 3. Calendar

### Takeaway

Finished and clean for a read-only public calendar: occurrence-based data model, database-enforced visibility, month-grouped public list, slug detail page with JSON-LD, and timezone-correct admin forms. It is a listing, not an interactive calendar (no grid view, no feed, no filters, no categories).

### Findings

- **Model.** Public shape is an *occurrence* (event id + occurrence id + slug, title, summary, description, `startsAt/endsAt` ISO instants, timezone, venue name/address, `isVirtual`, registration window) — "The public calendar is a list of occurrences, not of events, since 'when is the next one' is the question" (`src/modules/calendar/contracts.ts:L3-L50`).
- **Read path.** `list_public_event_occurrences(p_site_id, p_from?, p_to?)`, max 100 rows; the database filters to published events, scheduled occurrences, active site and active organization, so "a suspended council disappears from the calendar the moment it is suspended" (`src/modules/calendar/data/rpc.ts:L11-L19`, `L30-L77`). There is no by-slug RPC: the detail page scans the same public window, which deliberately means "a finished event has no page" (`L79-L113`).
- **Formatting.** Pure helpers `formatOccurrence`, `isUnderway`, `groupByMonth`, `formatLocation` (`src/modules/calendar/domain/format.ts:L51`, `L137`, `L151`, `L173`).
- **Public rendering.** `EventCalendar` groups by month heading with when/where/what cards; empty state copy (`src/components/public/event-calendar.tsx:L14-L55`). An `upcoming_events` block type exists in the blocks-v1 page schema so a council can embed events on any page (block kinds from `src/modules/content/blocks-v1/schema.ts`).
- **Admin.** Statuses `draft | published | canceled` on the form; managed row also carries `completed | archived` and `registration_count` (`src/modules/calendar/admin/contracts.ts:L11-L12`, `L74-L112`).
- **Tests.** `src/modules/calendar/calendar.test.ts` (347 lines), `admin/admin.test.ts` (255 lines).

### Gaps

- No ICS, no grid/month view, no categories/tags, no recurring-event editor, no per-event images.
- Past events are unreachable publicly by design (no archive).
- The organization identity's address is used as a fallback `Place` in JSON-LD only when the occurrence has none (`structured-data.ts:L69-L83`) — reasonable but worth knowing.

---

## 4. Members and import

### Takeaway

Finished, small, and deliberately vertical-neutral. A member is `contacts` (person) + `member_records` (membership: external id, joined date, standing, rank key, notes). CSV import is a careful pure parser + heading-guesser with a preview step and row-by-row writes; dedup is **email-only** (no name/phone/member-number matching, no duplicate review UI). Roles are platform roles (`council_owner`, `membership_secretary`), not member roles. The Knights flavour lives only in labels ("Rank or degree", "Member number") and the orgtype vocabulary.

### Findings

- **Model.** `member_records(organization_id, contact_id UNIQUE per org, external_member_id, joined_on, standing ∈ active|inactive|honorary|transferred|deceased, rank_key, notes)`; `external_member_id` unique per org when present (`supabase/migrations/013_members_roster.sql:L59-L107`). Header comment: "Deliberately vertical-neutral. `external_member_id` is 'the id assigned by whoever charters this organization' — a KofC member number is one instance. `rank_key` is 'a level held within the organization'; KofC degrees motivated it, and the display vocabulary belongs to orgtype" (`L15-L20`). TypeScript mirror: `src/modules/members/contracts.ts:L3-L13`, `L15-L31`, `L42-L74`.
- **Access.** Everything gated `people.manage` in the database; a `membership_secretary` role preset gets `people.manage` + `people.export` ("named for the job, not the vertical") (`013:L24-L57`; `src/modules/members/index.ts:L7-L9`). RPCs: `save_member`, `archive_member`, `list_members`, `get_member` (`src/modules/members/data/rpc.ts:L22-L39`). `list_members`/`get_member` are revoked from anon (`022:L11-L13`).
- **Dedup.** `save_member` with a null record id looks up an existing contact by `email_normalized`; if found it updates that contact (and only fills phone if blank) instead of inserting; then upserts `member_records` `ON CONFLICT (organization_id, contact_id)` (`013:L213-L265`). No match on name, phone or external id; a row without an email always creates a new contact.
- **CSV import.** Hand-written RFC-4180 reader handling BOM, CRLF, quoted commas/newlines, `""` escapes; refuses ragged rows with the line number; max 5,000 rows (`src/modules/members/import/csv.ts:L1-L53`). Heading guesser with alias tables for ten fields including `fullName` (split "Last, First" / "First Last", keeping multi-word surnames), standing aliases (`lapsed`→inactive, `honorary life`→honorary …), and date normalization that refuses ambiguous dd/mm vs mm/dd (`src/modules/members/import/mapping.ts:L44-L103`, `L112-L125`, `L181-L238`). Server action: 2 MB upload cap, preview with per-row errors, then `applyMemberImport` writes ready rows one at a time ("a single bad record cannot roll back the rest"), skipping problem rows and stopping on the first `forbidden` (`src/app/actions/import-members.ts:L25`, `L207-L280`). Plan-gated `members.import` (Starter) with a sentence built from the plan table (`L52`; `src/modules/entitlements/plans.ts:L80-L81`).
- **UI.** `src/components/members/{member-form,member-form-dialog,archive-member-button,import-members}.tsx`; routes `dashboard/members`, `/new`, `/[id]`, `/import`.
- **Tests.** `members.test.ts` (224 lines), `import/import.test.ts` (253 lines).

### Gaps

- No export despite the `people.export` capability (no CSV download in `dashboard/members/page.tsx`).
- No duplicate-review step in import; email-only dedup means spreadsheets without emails create duplicates on re-import.
- No member-facing login/portal, no member self-service profile, no dues/renewals, no member tags/segments (the `contact_tags` tables from 003 are unused by this module).
- `contacts` already has `relationship_stage` and `source` (`manual`, `event_registration`, `import`, `membership`, `referral`, `website_enquiry`) (`023:L30-L36`), which is a CRM spine the member module barely uses.

---

## 5. Enquiries (contact form)

### Takeaway

Finished and the best-hardened public write path in the repo: honeypot + per-IP and per-site rate limits + optional Cloudflare Turnstile that fails closed once configured, a database function that derives the tenant from the site id and upserts a `contacts` row without overwriting officer-entered data, a consent row for the chosen topic, an audit event, a dashboard inbox with new/handled/spam, and a best-effort Resend notification. One real bug: the notification email links to `/dashboard/messages`, a route that does not exist (the inbox is `/dashboard/enquiries`). This maps almost 1:1 onto a storefront "inquiries" module.

### Findings

- **Schema.** `public_enquiries(organization_id, site_id, contact_id, topic, message, status ∈ new|handled|spam, handled_at, handled_by, created_at)`; topic shares the vocabulary of `contact_communication_preferences.purpose` so "an enquiry about membership is the same concept as consent to be contacted about membership" (`supabase/migrations/023_public_enquiries.sql:L38-L60`). Design notes: first anonymous write in the schema; function takes a site id and derives the org; contact upsert "never overwrites a name, a phone number, or a relationship stage a council has since set by hand"; "Nothing here sends email" (`023:L1-L28`).
- **Submit function.** `submit_public_enquiry(p_site_id, p_name, p_email, p_topic, p_message)` → inserts the enquiry, inserts an `opted_in` email preference for that topic with source `website_enquiry` `ON CONFLICT DO NOTHING` (an earlier opt-out wins), writes `enquiry.received` audit (`023:L100`, `L163-L185`). Admin: `list_public_enquiries`, `set_enquiry_status` (`023:L200`, `L235`).
- **TypeScript.** Topics (Knights-worded labels: "Joining the council", "Helping at an event", …), submission schema with the `website` honeypot (`max(0)`), public row schema (`src/modules/people/enquiries/index.ts:L12-L18`, `L38-L49`, `L53-L63`); RPC wrappers with status unions (`src/modules/people/enquiries/data/rpc.ts:L34-L68`, `L88-L138`).
- **API route.** `POST /api/enquiries`: trusted tenant headers (site id never from body) → zod parse → honeypot returns 202 silently → rate limits charged before anything expensive (3/hour per address, 60/hour per site) → Turnstile verify (`not_configured` passes; `failed`/`unavailable` → 403) → RPC → fire-and-forget email (`src/app/api/enquiries/route.ts:L19-L45`, `L59-L63`, `L76-L102`, `L104-L112`, `L128-L133`). Rate limiting is a Postgres RPC `consume_rate_limit` with a three-way allowed/denied/unavailable outcome (`src/server/http/rate-limit.ts:L30-L45`; `supabase/migrations/021_signup_rate_limit.sql`). Turnstile is server-verified only (`src/server/http/turnstile.ts:L1-L35`).
- **Public form.** Posts JSON with `website` honeypot and `cf-turnstile-response`; loads the Turnstile script (`src/components/public/enquiry-form.tsx:L48-L57`, `L135-L153`). Embeddable as `enquiry_form` and `contact_details` block kinds.
- **Dashboard.** `src/app/(dashboard)/dashboard/enquiries/{page.tsx,actions.ts,EnquiryStatusButtons.tsx}`. Feature `enquiries.form` is Free (`plans.ts:L76`).
- **Notification.** `notifyCouncilOfEnquiry` resolves recipient = org `public_email`, else the first active `council_owner`'s auth email via the admin client; sends subject "Someone wrote to {council}" with no message body on purpose (privacy) (`src/server/email/notify-enquiry.ts:L25-L31`, `L48-L84`, `L86-L113`). **Bug:** `dashboardUrl` is `/dashboard/messages` (`L92`) but the route directory is `dashboard/enquiries` (`ls src/app/(dashboard)/dashboard` → `enquiries`, no `messages`; `grep dashboard/messages src next.config.ts` → only this one reference, no redirect).

### Gaps

- Dead link in the notification email (above).
- Notification is best-effort with no retry, no record of send, no reply-from-dashboard (the email says "answer the person from your dashboard" but the inbox has only status buttons; no reply composer).
- No auto-reply/acknowledgement to the person who wrote in.
- No attachments, no custom fields, no assignment to an officer, no SLA/aging view.

---

## 6. Recognitions, programs, prayer, offices, changeover, volunteers (and access)

### Takeaway

All six are finished vertical slices of the same pattern (table + 3–5 RPCs with two gated projections + zod + dashboard forms + tests). Three are genuinely generic with a rename (offices → staff/team directory; programs → services/offerings; volunteers → shifts/needs). Recognitions is semi-generic (awards/testimonials). Prayer and changeover are Knights/church-specific in purpose, though their *mechanics* (consent-attested personal data with expiry; a dated, atomic slate that swaps a directory and access together) are reusable ideas.

### Findings

**Recognitions** — "who the council honoured, and what it said about them". One row = one presentation: `award_kind` ∈ `member_of_the_month | family_of_the_month | certificate_of_appreciation | organization_award | other`, free `award_label`, period label/start, honorees as a bounded JSONB array of `{name, detail?}` (explicitly not FK'd to members because honorees include non-members), presenter, a blocks-v1 rich-text citation, photo asset, `is_published`; gated `content.edit_any`; `list_recognitions` (working) and `list_publication_recognitions` (published only) (`src/modules/recognitions/index.ts:L1-L16`; `contracts.ts:L17-L26`, `L55-L90`; `supabase/migrations/051_recognitions.sql:L1-L45`). UI: `src/components/dashboard/recognitions/recognitions-manager.tsx`, `recognition-form.tsx`. **No public website block** — it prints only in the newsletter.

**Programs** — "what the organization is doing this year, by pillar". `programs(fraternal_year_start, pillar_key, title, description, status ∈ planned|open|ongoing|complete|cancelled, schedule_note, display_order, is_published)`; the pillar is validated against `organization_types.program_taxonomy` (kofc: Faith/Family/Community/Life; generic: Service/Fellowship/Outreach) so the module "must not know the words"; director is an office, not a column (`src/modules/programs/index.ts:L1-L11`; `contracts.ts:L1-L46`; `supabase/migrations/050_programs_faith_in_action.sql:L1-L45`). The header says ledger columns (hours, dollars, people served, reporting codes) "come later". Feature `programs.ledger` (Starter). **No public block.**

**Prayer intentions** — a consent-defined list of named medical/personal intentions. `intention` (printed line), `subject_name`, `requested_by`, `posted_on`, `expires_on` (defaults to +1 year, the council's printed policy), `consent_confirmed` with `consent_attested_by/at` set by the RPC, `consent_note`, `is_published`; a table CHECK forbids publishing without consent; admin gated `people.manage`, newsletter projection gated `content.edit_any` and returns only line + date within a window; "There is no anonymous surface here and none is to be added" (`src/modules/people/prayer/index.ts:L1-L15`; `contracts.ts:L20-L51`, `L89-L100`; `supabase/migrations/052_prayer_intentions.sql:L1-L42`).

**Offices** — "who holds which council office". `council_offices(office_title, holder_name, holder_email, holder_phone, member_record_id?, is_published, show_email, show_phone, display_order, roster_key ∈ council|programs|assembly, pillar_key?)`; all three publish flags default false and the public projection applies them in SQL so a renderer cannot leak; a NULL holder is a vacancy that prints "OPEN" (`src/modules/people/offices/index.ts:L1-L15`; `contracts.ts:L1-L63`; `supabase/migrations/022_council_offices.sql:L1-L26`; `053_council_offices_rosters.sql:L1-L23`). Public block kind `officers`; feature `officers.directory` is Free. Four projections: admin list, roster list (with roster/pillar), public, publication.

**Changeover** — the annual officer handover. A *saved slate* (`draft → ready → applied | abandoned`) of per-office lines (`new_holder | unchanged | vacant`) plus handover notes, applied atomically in one transaction through `save_council_office` and migration 058's access functions (invite/grant/revoke), with `is_published` inherited but `show_email/show_phone` reset because "Bill's permission to publish Bill's telephone number is not Frank's"; produces a printable briefing for incoming officers (`src/modules/people/changeover/index.ts:L1-L14`; `contracts.ts:L3-L58`; `supabase/migrations/059_officer_changeover.sql:L1-L60`; `src/components/dashboard/changeover/{briefing-list,briefing-print-controls,review-panel,slate-line-form}.tsx`). Deeply Knights-framed (1 July fraternal year, Form 185) but the mechanism is "scheduled directory + access swap".

**Volunteers** — shifts, not people. `volunteer_shifts(role_name, occasion, event_id?, starts_at, ends_at, capacity, filled, coordinator, is_published, display_order)`; `filled` is a count the coordinator types ("Public sign-up is a second feature with its own surface — a public write path, its own rate limits, its own spam problem"); no personal data, hence no consent machinery (`src/modules/people/volunteers/index.ts:L1-L11`; `contracts.ts:L3-L31`; `supabase/migrations/026_volunteer_shifts.sql:L1-L25`). Public block kind `volunteer_shifts`. **The entitlement key `volunteers.public_signup` exists (`plans.ts:L84`) but nothing implements it** (`grep public_signup src` → only the plan table).

**Access** (people/access, adjacent) — who can sign in: invite by email (Supabase `inviteUserByEmail` creates the account and sends `invite.html`; an already-registered address gets the Resend "has given you access" mail instead), grant/revoke roles against a privilege ceiling, suspend/restore, last-owner guard enforced by a deferred constraint trigger (`src/modules/people/access/index.ts:L1-L13`; `src/server/auth/invited-account.ts:L5-L24`, `L44`; `supabase/migrations/058_who_can_sign_in.sql:L1-L40`). Generic team-access management.

**Orgtype** — the vocabulary layer all of the above lean on: `organization_types(unit_noun_*, unit_identifier_*, parent_body_noun, member_noun_*, year_label, program_taxonomy …)` with the rule "no module outside `compliance` may hard-code a vertical's nouns" (`src/modules/orgtype/index.ts:L1-L9`; `supabase/migrations/010_orgtype_vocabulary.sql:L1-L40`).

### Gaps

- Recognitions and programs have no public website block; they exist for the newsletter.
- Programs' promised ledger columns (hours/dollars/people served) are not built.
- Volunteer public sign-up: plan key only, no code.
- Prayer has no public or member-only surface and should not get one as designed.
- Changeover is tightly bound to the three Knights rosters and the July year; generalising requires parameterising roster keys and the year boundary through orgtype (the year already is; roster keys are hard-coded in `offices/contracts.ts:L12`).

---

## 7. Notifications and email infrastructure

### Takeaway

Email is the thinnest layer in the repo. Two transactional emails are sent by raw `fetch` to Resend's REST API with no SDK, no retry, no send record, and a silent no-op if `RESEND_API_KEY` is absent. Supabase Auth mail goes through Resend SMTP (confirmed live in the ops runbook, 100 emails/hour). A proper `outbox_messages` table with leasing, attempts and dead-lettering exists in migration 001 but **no TypeScript consumes it**; the `contact_communication_preferences` consent table exists and is written by enquiries and registration, but no sending path reads it. Inngest keys are in `.env.example` but Inngest is not referenced anywhere in `src/`. There is no scheduler of any kind (no Cloudflare cron, no Inngest wiring).

### Findings

- **Transactional sends.** `notifyCouncilOfEnquiry` and `notifyCouncilInvitation` both: `const apiKey = process.env.RESEND_API_KEY?.trim(); if (!apiKey) return;` then `fetch("https://api.resend.com/emails", …)` from `HallForge <no-reply@hallforge.com>`, logging to `console.error` on non-2xx and on throw; both are documented as best-effort with "a mail failure must not turn a completed [action] into an error" (`src/server/email/notify-enquiry.ts:L25-L31`, `L86-L117`; `src/server/email/notify-council-invitation.ts:L20-L68`). Inline table-based HTML with manual escaping; no templating library. One unit test file (`notify-council-invitation.test.ts`).
- **Auth email.** Thirteen Supabase templates live in `supabase/email-templates/` with a README on the token-hash flow and the Resend SMTP setup (`supabase/email-templates/README.md:L1-L13`, `L66-L98`). The runbook records the cutover as done: "custom SMTP now sends through Resend as HallForge <no-reply@hallforge.com>, DKIM-verified on hallforge.com, 100 emails/hour" (`docs/launch/2026-09-01-ops-runbook-two-manual-steps.md:L6-L14`).
- **Outbox (unused).** `outbox_messages(topic, aggregate_type/id, payload JSONB, dedupe_key, status ∈ pending|processing|completed|dead, available_at, attempts, max_attempts ≤100, locked_at/by, processed_at, last_error)` with dedupe unique indexes and a dispatch index, plus `hallforge_private.enqueue_outbox_message(...)` doing `ON CONFLICT DO NOTHING` (`supabase/migrations/001_identity_tenancy.sql:L382-L430`, `L1264-L1310`). `grep -rn outbox_messages src` → only the generated `src/types/database.ts`. A sibling `background_jobs` table follows (`001:L432`).
- **Consent (written, never read for sending).** `contact_communication_preferences(contact_id, channel ∈ email|sms|phone|postal, purpose ∈ event_updates|news|fundraising|membership|volunteering, preference ∈ unknown|opted_in|opted_out, source, captured_at, captured_by)` with an append-only `..._events` audit table (`supabase/migrations/003_events_relationships.sql:L170-L215`). Writers: enquiries (`023:L172-L180`) and paid registration (`004:L358-L410`). Readers for the purpose of sending: none.
- **Workers that do exist** are for content publication and payments, not mail: `src/server/content/publication/worker.ts`, `src/server/payments/processing/{provider-event-worker,expiry-worker}.ts`, all "called from an authenticated server-owned scheduler" that is not installed (`src/server/payments/processing/README.md:L22-L24`).
- **Config.** `.env.example:L48-L53` lists `RESEND_API_KEY` and `INNGEST_EVENT_KEY/SIGNING_KEY`; `grep -rn inngest src` → nothing.
- **Known plan.** Build plan M-level item: "Provider, sending identity with SPF/DKIM/DMARC, unsubscribe tokens, per-send records. Justified here rather than under newsletters because the interest form, auth confirmation, and registration receipts all depend on deliverable mail" (`docs/launch/2026-08-20-module-build-plan.md:L82`).

### Gaps

- No send log / delivery record / webhook from Resend; no retry; no dead-letter; the outbox is dormant.
- No unsubscribe, list, segment, template system, or bulk send.
- No scheduler for the workers that already exist.
- No registrant confirmation, no auto-reply to enquiries, no digest.
- Dead `/dashboard/messages` link in the one product email that is live.

---

## Generic vs Knights-of-Columbus-specific (this area)

**Generic, usable as-is or with a rename**
- `contacts` + `contact_communication_preferences` (+ events) consent spine (`003`).
- `members`/`member_records` with `external_member_id` and `rank_key` (`013`); CSV import parser and heading guesser (`members/import/*`).
- `public_enquiries` + `/api/enquiries` + `enquiry-form` + dashboard inbox (`023`, `src/app/api/enquiries`).
- `events` / `event_occurrences` / ticket types / registrations / payment ledger schema (`003`, `004`, `008`, `014`) and the Stripe orchestration boundary (`src/server/payments/*`).
- Calendar public read + JSON-LD + month list (`src/modules/calendar`, `event-calendar.tsx`).
- `council_offices` as a "people directory with per-field consent" (`022`, `053`) — the table name and `roster_key` values are the only Knights residue.
- `volunteer_shifts` as "needs/shifts" (`026`).
- Access/invitations (`058`, `people/access`).
- Rate limit RPC, Turnstile verifier, honeypot pattern (`021`, `src/server/http/*`).
- `outbox_messages` / `background_jobs` tables (`001`) — unused but generic.
- Orgtype vocabulary table (`010`) — the mechanism for keeping the rest generic.

**Knights-of-Columbus-specific (or church-specific)**
- Newsletter section vocabulary and template: `assembly_officers`, `prayer`, fraternal year, pillar bands, "OPEN" vacancies, Norwester/Montserrat corpus-matched design (`newsletter/contracts.ts`, `render/*`).
- Recognitions award kinds (`member_of_the_month`, `family_of_the_month`) (`051`).
- Programs' "fraternal year" + pillar taxonomy (`050`) — the *shape* is generic (year + category + status), the words are not.
- Prayer intentions in purpose (`052`); mechanism (consent-attested, expiring personal lines) is reusable.
- Changeover's three rosters, 1 July year, Form 185 framing (`059`, `changeover/contracts.ts`).
- Enquiry topic labels ("Joining the council", "Helping at an event") (`enquiries/index.ts:L12-L18`).
- Roles `council_owner`, `membership_secretary`; copy throughout ("Grand Knight", "a man", "Brother Knight").
- Email template branding and copy (`supabase/email-templates/*`, `notify-*.ts`).

---

## Reusable for a small-business storefront engine? — verdict per module

| Module | Verdict | Why / what changes |
|---|---|---|
| **Newsletter (print issue)** | **Drop** (keep the renderer idea) | It is a KofC print publication, not email marketing. The pure-renderer + print-route + blocks-v1 rich-text pattern is worth copying for "printable menu / price list / flyer", but the sections, template and record bindings are all Knights. No sending exists to reuse. |
| **Email infrastructure** | **Replace** (reuse the outbox table and consent tables) | Two raw Resend fetches with no record or retry is not a base. Keep `outbox_messages`, `background_jobs`, `contact_communication_preferences`; build a real sender/worker on top. Choose Resend SDK or keep REST; add send log, webhook ingestion, unsubscribe tokens. |
| **Enquiries** | **Reuse as-is** (rename labels; fix dead link) | Maps directly to storefront "inquiries": site-derived tenant, honeypot + rate limit + Turnstile, contact upsert with consent, inbox with statuses, owner notification. Change topic vocabulary to storefront purposes (quote, booking, general) and add a reply/auto-ack later. |
| **Calendar (public)** | **Reuse as-is** | Occurrence model, DB-enforced visibility, JSON-LD, month list, timezone handling. Add ICS and a grid view if the vertical needs them. |
| **Events admin** | **Reuse with changes** | Finished for simple events; needs ticket-type/capacity/form-field admin and recurrence. |
| **Registration / ticketing / Stripe** | **Reuse schema; rebuild application layer** | Schema and payment boundary are strong; the public endpoint is a 503 stub, the checkout component is dead code against a nonexistent table, flags are off, no scheduler. For a storefront, prefer Stripe Checkout/Payment Links for products and keep this only if event ticketing is in scope. |
| **Members + import** | **Reuse with changes** | Rename to "customers/contacts"; keep `contacts`, CSV import, email dedup. Drop `member_records` standing/rank unless a membership/loyalty concept is wanted. Add export and a duplicate-review step. |
| **Offices** | **Reuse with changes** → "Team / staff directory" | Per-field publish consent and vacancy semantics are directly useful. Remove `roster_key`/`pillar_key` or re-vocabularise via orgtype. |
| **Programs** | **Reuse with changes** → "Services / offerings" | Year + category + status + published is a fine services catalog skeleton; needs price/duration/booking fields and a public block (none exists). |
| **Volunteers (shifts)** | **Drop** or **reuse with changes** → "staffing needs / open shifts" | Only if the storefront vertical has shifts (restaurants, salons). Public sign-up not built. |
| **Recognitions** | **Reuse with changes** → "awards / testimonials / featured customers" | Honorees-as-names, citation, photo, period, publish flag generalise; replace award kinds. No public block yet. |
| **Prayer intentions** | **Drop** | Church-specific purpose; keep the consent-attestation + expiry pattern as a reference for any personal-data listing. |
| **Changeover** | **Drop** (keep the idea) | KofC annual handover. The "scheduled directory + access swap in one transaction" idea could become "ownership transfer / new manager onboarding" later, but nothing here lifts out cleanly. |
| **Access / invitations** | **Reuse as-is** | Generic team access with last-owner guard and privilege ceiling. |
| **Orgtype vocabulary** | **Reuse as-is** | This is the mechanism that would let one engine serve storefront verticals without renaming tables. |
