# HallForge: content system, block schema, editor, rendering, media, design studio

**Scope:** read-only review of `/home/user/hallforge` (shallow clone of `mobius29er/hallForge`), covering `src/modules/content`, `src/components/dashboard/pages` (the real editor), `src/components/page-editor` and `src/components/public/block-renderer.tsx` (retired prototype), `src/components/public`, `src/modules/presence` + `src/components/presence`, `src/modules/design` + `src/components/dashboard/designs`, `src/modules/media` + `src/components/media`, `src/server/content`, the public routes under `src/app/(public)`, and the content/media/design migrations under `supabase/migrations`. All paths below are relative to `/home/user/hallforge` unless they start with `/`.

**Headline:** the content system is the most finished part of HallForge and is a credible engine base. It is a strict-Zod, versioned, JSONB block document (`blocks_v1`, 14 block types) with a draft/revision/review/publish pipeline done in SECURITY DEFINER Postgres functions, an isomorphic compiler that produces a stricter published document, one server-component renderer, per-tenant CSS-variable theming, flat navigation, two signup templates plus four page layouts, and a media pipeline on R2 with database-gated delivery. The editor is a **forms-beside-live-preview** editor (no drag-and-drop, no inline editing), which the founder's own decision record calls a stepping stone. There is **no tRPC API over blocks** (the tRPC router is deliberately empty) and **no block-level mutation API at all** (every save sends the whole document). There is **no schema migration path** beyond a version literal. The KofC-specific surface is shallow and concentrated in starter copy, three dynamic blocks (`officers`, `volunteer_shifts`, `contact_details` semantics), the supported-programs list and a handful of UI nouns.

---

## 1. The block schema

### Takeaway

One Zod file defines the document: `src/modules/content/blocks-v1/schema.ts`. Fourteen block types in a `z.discriminatedUnion("type", …)`, each `.strict()`, each carrying `id` (canonical UUID), `version: z.literal(1)`, an optional template `slot`, and two optional section-presentation fields (`anchor`, `background`). A page is stored as JSONB in `content_revisions.document` with `document_schema = 'blocks_v1'` and `schema_version = 1`; publishing compiles it to a stricter `published_page_v1` document stored in `content_publication_snapshots.render_document`. "v1" means exactly one literal version; there is **no migration machinery** of any kind. The schema is a good typed substrate for an AI, but the API over it is document-level (whole-document save with optimistic concurrency), not block-level create/update/move/delete, and it is exposed as Next server actions over Postgres RPCs, not tRPC.

### Findings

**Block types and props.** Every block shares the identity fields at [schema.ts:L443-L449](src/modules/content/blocks-v1/schema.ts) (`id` UUID regex, `version: z.literal(BLOCKS_V1_VERSION)`, optional `slot` matching `^[a-z][a-z0-9_]{0,38}$`) spread with `sectionPresentation` at [schema.ts:L410-L421](src/modules/content/blocks-v1/schema.ts) (`anchor?: string` slug ≤64, `background?: "default"|"soft"|"raised"|"brand"`). Shared sub-schemas: `actionSchema` `{label ≤80, href: safeContentLink, emphasis: primary|secondary}` [schema.ts:L382-L389](src/modules/content/blocks-v1/schema.ts); `richTextDocumentSchema` is a ProseMirror-shaped doc with paragraph/heading(2-4)/bulletList/orderedList/blockquote/horizontalRule nodes, bold/italic/link marks, capped at 1,000 nodes and 20,000 characters [schema.ts:L186-L380](src/modules/content/blocks-v1/schema.ts). The union is at [schema.ts:L855-L870](src/modules/content/blocks-v1/schema.ts); the document wrapper caps at `MAX_BLOCKS_V1 = 40` and enforces unique ids [schema.ts:L872-L891](src/modules/content/blocks-v1/schema.ts).

| type | data props | lines |
|---|---|---|
| `hero` | `eyebrow?`, `heading?`, `lead?` (500), `actions?` ≤2, `backgroundAssetId?` UUID, `stats?` ≤4 `{value, label}` | [L466-L494](src/modules/content/blocks-v1/schema.ts) |
| `rich_text` | `heading?`, `document` (rich text) | [L496-L507](src/modules/content/blocks-v1/schema.ts) |
| `callout` | `heading?`, `body` (1,000), `emphasis: note|important`, `action?` | [L509-L523](src/modules/content/blocks-v1/schema.ts) |
| `call_to_action` | `heading`, `body?`, `actions` ≤2 | [L525-L537](src/modules/content/blocks-v1/schema.ts) |
| `image` | `assetId?`, `alt?`, `isDecorative?`, `caption?`, `heading?`, `body?` (rich text), `layout?: full|text_start|text_end` | [L560-L580](src/modules/content/blocks-v1/schema.ts) |
| `upcoming_events` | `eyebrow?`, `heading?`, `intro?`, `count?` 1-10, `emptyMessage?` (stores no events) | [L592-L606](src/modules/content/blocks-v1/schema.ts) |
| `card_grid` | `eyebrow?`, `heading?`, `intro?`, `emphasis: text|number`, `columns?` 2-4, `cards` ≤8 `{headline, body?, footnote?, footnoteLabel?, icon?}`, `layout?`, `action?` | [L620-L656](src/modules/content/blocks-v1/schema.ts) |
| `contact_details` | `heading?`, `intro?`, `showMeeting?`, `showAddress?`, `showContact?`, `additionalInfo?` (stores no facts; reads org profile) | [L671-L686](src/modules/content/blocks-v1/schema.ts) |
| `officers` | `heading?`, `intro?`, `columns?` (stores no names; reads offices) | [L701-L713](src/modules/content/blocks-v1/schema.ts) |
| `enquiry_form` | `eyebrow?`, `heading?`, `intro?`, `submitLabel?`, `successMessage?`, `layout?` (topics are fixed) | [L727-L742](src/modules/content/blocks-v1/schema.ts) |
| `volunteer_shifts` | `eyebrow?`, `heading?`, `intro?`, `count?` 1-12, `columns?`, `emptyMessage?`, `layout?`, `action?` | [L752-L769](src/modules/content/blocks-v1/schema.ts) |
| `divider` | `{}` | [L771-L777](src/modules/content/blocks-v1/schema.ts) |
| `gallery` | `eyebrow?`, `heading?`, `intro?`, `images` ≤24 `{assetId, alt?, caption?}` | [L788-L814](src/modules/content/blocks-v1/schema.ts) |
| `video` | `eyebrow?`, `heading?`, `intro?`, `youtubeVideoId?` (11 chars), `title?`, `caption?`, `autoplay?` | [L830-L853](src/modules/content/blocks-v1/schema.ts) |

Link safety is centralised: `normalizeSafeContentLink` accepts only `#anchor`, site-relative paths, `mailto:`, `tel:` and `https://` [schema.ts:L87-L183](src/modules/content/blocks-v1/schema.ts). Card icons are a closed set of 12 meaning-keys (`faith`, `family`, `community`, `life`, `council`, …) [card-icons.ts:L18-L31](src/modules/content/blocks-v1/card-icons.ts).

**Versioning.** `BLOCKS_V1_SCHEMA = "blocks_v1"` and `BLOCKS_V1_VERSION = 1` [schema.ts:L4-L5](src/modules/content/blocks-v1/schema.ts). A block with `version: 2` is rejected by the schema (test "unknown schema version" at [schema.test.ts:L170-L171](src/modules/content/blocks-v1/schema.test.ts)). A grep for "migrat" in `src/modules/content`, `src/server/content` and the editor finds only comments explaining why things were designed to avoid migrations (tokens not hexes, asset ids not widths) — there is no `migrate_*` function, no per-block version upgrade, and no fleet re-validation job. The database CHECK admits four document schemas (`blocks_v1`, `tiptap_v1`, `form_presentation_v1`, `event_presentation_v1`) [002_publishing_media.sql:L207-L208](supabase/migrations/002_publishing_media.sql) but only `blocks_v1` is implemented; migration 045 says the `article/tiptap_v1` pairing is "registered, unused" [045_news_posts.sql:L13-L15](supabase/migrations/045_news_posts.sql). The compiler stamps `PUBLISHED_PAGE_SCHEMA = "published_page_v1"`, `PUBLISHED_PAGE_VERSION = 1` and a compiler id [compiler.ts:L31-L33](src/server/content/compiler.ts), and the snapshot row records `ruleset_version`, `compiler_version`, `sanitizer_version` [002:L439-L441](supabase/migrations/002_publishing_media.sql) — so the hooks for a future migration are stored, but nothing reads them.

**Storage.** Tables in `supabase/migrations/002_publishing_media.sql`: `content_items` (kind, lifecycle, working/published pointers, `concurrency_version`) [L80-L120](supabase/migrations/002_publishing_media.sql); `site_routes` (normalized path, content/redirect/gone/system kinds) [L127-L175](supabase/migrations/002_publishing_media.sql); `content_revisions` with `document JSONB`, `document_schema`, `schema_version`, `title`, `summary`, `seo_inputs`, `attribution_inputs`, `content_hash` sha256, `save_kind IN (initial, autosave, manual, import, restore, conflict)`, document capped at 8 MB [L182-L228](supabase/migrations/002_publishing_media.sql); `content_conflicts` [L231]; `content_review_requests` [L272]; `content_publication_commands` [L367]; `content_publication_snapshots` with `render_document`, `plain_text`, `seo_projection`, `structured_data_inputs`, `media_manifest` [L422-L467](supabase/migrations/002_publishing_media.sql). Kind was widened to include `post` in [045_news_posts.sql:L27-L37](supabase/migrations/045_news_posts.sql); posts share the whole pipeline and must live under `/news/` [repository.ts:L70-L80](src/server/content/admin/repository.ts).

**Validation.** Zod at every boundary: the authoring schema in the editor (`blocksV1DocumentSchema.safeParse` on every change, refusing bad edits [page-editor.tsx:L144-L157](src/components/dashboard/pages/page-editor.tsx)), in the server-action repository (`pagePayloadFields` [repository.ts:L35-L42](src/server/content/admin/repository.ts)), and in the compiler which re-parses and then applies publication-completeness rules (`PublicationIssue` codes such as `ACTION_LINK_REQUIRED`, `IMAGE_ALT_REQUIRED`, `ANCHOR_DUPLICATE`, `HEADING_LEVEL_SKIPPED`, `LINK_LABEL_VAGUE` [compiler.ts:L45-L75](src/server/content/compiler.ts)). The published document is a second, stricter Zod union [compiler.ts:L156, L440](src/server/content/compiler.ts). The public resolver re-validates the snapshot on read with `publishedPageDocumentV1Schema` [public-query.ts:L4, L108](src/server/content/public-query.ts). The database adds its own CHECKs (JSON is an object, size bound, hash format).

**API for typed tool calls.** tRPC is intentionally empty: `appRouter = createTRPCRouter({})` with a comment that the v1 replacements are server components and server actions calling SECURITY DEFINER RPCs [root.ts:L1-L17](src/server/root.ts). The content RPC surface is typed in [admin/rpc.ts:L7-L116](src/server/content/admin/rpc.ts): `list_admin_content_items`, `get_content_working_copy`, `create_content_draft`, `save_content_revision` (takes the whole `p_document`, `p_expected_revision_id`, `p_expected_version`, `p_idempotency_key`), `submit_content_for_review`, `decide_content_review`, `request_content_publication`, `unpublish_content_item`, `trash_content_item`, `restore_content_revision`, `get_content_conflict`, `resolve_content_conflict`, `get_site_content_policy`. The repository wraps these as `listPages/listPosts/loadWorkingPage/createPage/createPost/savePage/submitForReview/decideReview/requestPublication/unpublishPage/trashPage/restorePage/loadConflict/resolveConflict/loadPolicy` [repository.ts:L433-L479](src/server/content/admin/repository.ts), and server actions re-export them with same-origin and capability gates [pages/actions.ts:L45-L254](src/app/(dashboard)/dashboard/pages/actions.ts). Block-level operations (`addBlock`, `moveBlock`, `updateBlock`, `confirmRemoval`) exist **only as client-side React functions** inside the editor [page-editor.tsx:L159-L240](src/components/dashboard/pages/page-editor.tsx); `createBlockV1(type, id)` gives typed defaults per block [block-helpers.ts:L112-L200](src/components/dashboard/pages/block-helpers.ts). A programmatic "insert_block / update_block / move_block" tool would have to be written on top of `loadWorkingPage` + `savePage`, which is feasible (document in, document out, optimistic version) but not present.

### Gaps

- No migration path: `version` is a literal, unknown versions are rejected, and there is no `migrate.ts`. Any future block change is additive-only or breaks stored pages.
- No block-level mutation API, no audit of who changed which block; revisions are whole-document with a `created_by`.
- No tRPC. The engine doc's `packages/blocks/tools` tool table (section 5.5) has no counterpart; the closest thing is the repository function set, which is page-level.
- `seo_inputs` and `attribution_inputs` are untyped `Record<string, unknown>` at the contract [admin/contracts.ts:L60, L230-L231](src/server/content/admin/contracts.ts); the worker narrows seo to `{title?, description?}` [worker.ts:L28-L34](src/server/content/publication/worker.ts). No per-page OG image, no JSON-LD type selector.
- No `visibility: {from, until}` schedule on blocks, no sections grouping blocks (a block *is* a section), no per-page `layout` variant.
- README says "twelve block types" [README.md:L59](README.md); the union has fourteen (gallery and video were added 1 September). Doc drift, not a bug.
- `divider` is still offered in the add-section grid [block-helpers.ts:L96-L100](src/components/dashboard/pages/block-helpers.ts) although `docs/page-builder-block-set.md` says it was retired from the picker [page-builder-block-set.md:L31](docs/page-builder-block-set.md).

---

## 2. The editor

### Takeaway

The production editor is `src/components/dashboard/pages/` (7,879 lines incl. tests), not `src/components/page-editor/` which is a retired prototype kept only for `/demo`. It is a **vertical form stack beside a live preview**: each block is a card of labelled fields (`block-fields.tsx`, 2,667 lines, one `XFields` component per type), blocks move with up/down buttons, a new block is appended at the bottom from an "Add a section" grid, and rich text is a deliberately small TipTap toolbar (bold, italic, bullets, numbered, quote, link). Autosave is a tested controller class (idle 5 s / max 15 s / retry 5 s) with IndexedDB recovery and server-side optimistic-concurrency conflicts. The founder's 1 September decision record states this editor "was always a stepping stone" to click-to-edit and drag-to-arrange, with Puck noted as a candidate. Maturity: solid and well-tested for what it is; not the direct-manipulation experience the engine doc or a small-business owner expects.

### Findings

**Two editors exist.** `src/components/page-editor/*` (block-editor 1,047 lines, block-picker, page-canvas with up/down/delete toolbar) targets `LegacyPageBlock` types (`text_image`, `officers_grid`, `meeting_info`, `join_cta`, `stats_grid`, `programs_grid`, `recent_posts`, `newsletter_signup`, …) declared "Retired prototype block shape kept only for the isolated legacy components while they are removed" [legacy-types.ts:L1-L23](src/components/page-editor/legacy-types.ts). Its only consumer outside itself is `/demo` [demo/page.tsx:L3-L6](src/app/demo/page.tsx) via `block-renderer.tsx`, which contains a comment "Upcoming Events Block (placeholder - will fetch real events)" [block-renderer.tsx:L453](src/components/public/block-renderer.tsx). Treat this directory as dead weight.

**The real editor.** `PageEditor` [page-editor.tsx:L66-L84](src/components/dashboard/pages/page-editor.tsx) takes `value {title, summary, document}` and `onChange`, validates every change through `blocksV1DocumentSchema` [L144-L157], supports `addBlock` (append only) [L187-L198], `moveBlock(index, ±1)` [L200-L211], and removal with a confirmation dialog [L213-L240]. Grep for `dnd|draggable|onDrag|sortable` in the directory returns nothing. Layout is a two-column grid above 1280px (form 34rem + preview) and a two-button edit/preview toggle below [L376-L420]. Clicking a section in the preview scrolls the form to that block (`goToBlock` [L92-L103]). Sticky save/review/publish bar at top [L246-L330]. Add-section grid [L620-L660].

**Live preview** runs the real compiler and the real renderer in the browser (`compilePageDocument` + `PublishedPageRenderer` inside `SiteChrome embedded`) [LivePreview.tsx:L1-L33](src/components/dashboard/pages/LivePreview.tsx), in a plain container not an iframe [LivePreview.tsx:L228](src/components/dashboard/pages/LivePreview.tsx). The compiler is isomorphic for exactly this reason [compiler.ts:L17-L29](src/server/content/compiler.ts). Publication issues are shown above the preview instead of blanking it.

**Rich text.** `RichTextField` uses `@tiptap/react` + `StarterKit` with code, codeBlock, strike, underline and trailingNode disabled, headings 2-4 only [RichTextField.tsx:L89-L107](src/components/dashboard/pages/RichTextField.tsx); every update goes through `sanitizeEditorDocument`, a filter (not validator) that strips TipTap attrs the strict schema refuses [rich-text-editing.ts:L7-L50](src/modules/content/blocks-v1/rich-text-editing.ts). Rich text appears in `rich_text.document` and `image.body` only. Dependencies: `@tiptap/*` 3.27 [package.json:L38-L43](package.json).

**Autosave.** `AutosaveController` [controller.ts:L34-L687](src/modules/content/autosave/controller.ts) with constants `AUTOSAVE_IDLE_MS = 5_000`, `AUTOSAVE_MAX_MS = 15_000`, `AUTOSAVE_RETRY_MS = 5_000`, `RECOVERY_TTL_MS = 24h` [types.ts:L1-L5](src/modules/content/autosave/types.ts). Outcomes are typed: saved/unchanged, conflict (with `conflictId`), retryable (offline/timeout/dependency), blocked (auth/authorization/context/content_too_large/invalid) [types.ts:L54-L97]. Recovery is an IndexedDB store keyed by user/org/item/base revision/schema [recovery-store.ts:L8-L44](src/modules/content/autosave/recovery-store.ts). `AutosavingPageEditor` composes the controller with `useSyncExternalStore` [autosaving-page-editor.tsx:L87-L109](src/components/dashboard/pages/autosaving-page-editor.tsx). Server side, `save_content_revision` takes `p_expected_revision_id` and `p_expected_version` and `p_save_kind: autosave|manual` [admin/rpc.ts:L35-L49](src/server/content/admin/rpc.ts); conflicts create a `content_conflicts` row and there is a conflict-resolution UI at `dashboard/pages/conflicts/[conflictId]` with `use_current|use_candidate|combined` [admin/rpc.ts:L96-L110](src/server/content/admin/rpc.ts). 361 lines of controller tests exist [controller.test.ts](src/modules/content/autosave/controller.test.ts).

**Media picking inside the editor.** `media-picker.tsx` (683 lines) fetches `/api/media/library` once per editor and includes free stock search/import (Pexels/Pixabay) [media-picker.tsx:L17-L63](src/components/dashboard/pages/media-picker.tsx); the library route requires `media.manage` capability [api/media/library/route.ts:L17-L28](src/app/api/media/library/route.ts).

**Page creation and templates.** `NewPageForm` offers the four page layouts from `PAGE_LAYOUTS` (`blank`, `council_home`, `about`, `events`) [NewPageForm.tsx:L11-L28](src/app/(dashboard)/dashboard/pages/NewPageForm.tsx), [page-layouts.ts:L77-L311](src/modules/content/layouts/page-layouts.ts). `TemplateSection`/`TemplatePicker` re-apply a site template to the draft, listing what would be added vs kept [template-picker.tsx:L8-L29](src/components/dashboard/pages/template-picker.tsx), via `applyTemplateAction` [pages/actions.ts:L254](src/app/(dashboard)/dashboard/pages/actions.ts). News posts have their own three shapes (`announcement`, `event-recap`, `meeting-notes`, `blank`) [post-templates.ts:L92-L206](src/modules/content/post-templates.ts).

**Maturity, bugs, TODOs.** 200 `*.test.ts(x)` files under `src/`, including an axe accessibility test for the editor [page-editor.a11y.test.tsx](src/components/dashboard/pages/page-editor.a11y.test.tsx). A grep for `TODO|FIXME|HACK|stub` across the content, editor, public, presence, design and media trees finds none in production code (only the retired `block-renderer.tsx` placeholder comment). Known pipeline risks are documented honestly in [docs/publication-pipeline-risks.md:L1-L60](docs/publication-pipeline-risks.md) (one fixed by migration 019; a lapsed image right still blocks the whole page). The direction is recorded in [docs/decisions/2026-09-01-direct-manipulation-editors.md:L13-L45](docs/decisions/2026-09-01-direct-manipulation-editors.md): "click the headline on the page itself and type; drag sections to rearrange; drag new sections in from a palette … The current sidebar-forms block editor was always a stepping stone." Puck is "candidate … not adopted yet"; GrapesJS declined.

### Gaps

- No drag-and-drop, no inline editing, no block insertion at an arbitrary position (append-only, then move up/down), no duplicate-block, no undo beyond browser history of the form.
- No multi-user presence or locking; conflicts surface only after a save.
- Rich text has no images inline, no tables, no headings level 1, no text alignment, no colour (by design).
- No per-block preview-scoped SEO fields, no page-level OG image picker, no scheduling UI for blocks.
- The retired prototype editor (`src/components/page-editor`, `block-renderer.tsx`, `[orgSlug]` routes) is still in the tree and queries tables that no longer exist (`website_pages`, `website_settings`) [orgSlug/[[...slug]]/page.tsx:L45-L75](src/app/(public)/[orgSlug]/[[...slug]]/page.tsx); the `[orgSlug]` layout fails closed with `notFound()` [layout.tsx:L1-L13](src/app/(public)/[orgSlug]/layout.tsx).

---

## 3. Rendering, theming, navigation, templates and starters

### Takeaway

Public pages are **dynamic server components on Cloudflare Workers** (`force-dynamic`, `revalidate = 0`) that call `resolve_public_content` with a site id taken from trusted tenant headers set by middleware. No ISR, no static export, no edge cache purge on publish. Theming is three hex colours plus a logo per organization, emitted as `--hf-brand`/`--hf-accent-light`/`--hf-accent-brand` CSS custom properties on the page root and consumed by a small set of Tailwind token strings; there is no Tailwind config per tenant and no font choice. Navigation is one flat `primary` menu (≤12 items; the tables support nesting, the module refuses it) with an automatic fallback to section anchors. Templates: two site templates (`starter`, `hall`; `hall` is default) built from the same `StarterPageInput`, merged by `slot` so re-applying never rewrites a council's words; four page layouts; signup publishes the default template word-only, and the onboarding checklist offers to "dress" it with Pexels photos.

### Findings

**Request path.** `src/middleware.ts` classifies the hostname (platform vs tenant subdomain vs custom) [host.ts:L3-L30](src/modules/tenancy/host.ts), resolves it through `resolvePublicDomainFromClient`, and rewrites GET/HEAD to the internal `/site` prefix with trusted tenant headers [middleware.ts:L47-L60](src/middleware.ts). The catch-all `src/app/(public)/site/[[...path]]/page.tsx` sets `dynamic = "force-dynamic"; revalidate = 0` [L31-L32](src/app/(public)/site/[[...path]]/page.tsx), reads `readTrustedTenantHeaders` [L313-L330], loads the page via `resolvePublicPage` (RPC `resolve_public_content`) [L39-L52] with navigation and identity in parallel [L199-L203], fetches events/offices/shifts only if the document contains those block types [L205-L214], preloads the hero image with a full `srcset` for LCP [L223-L247], and renders `SiteChrome` + `PublishedPageRenderer` + JSON-LD [L255-L288]. Redirect/gone/not-found/unavailable are handled per `PublicContentResult` [public-query.ts:L62-L80](src/server/content/public-query.ts). Metadata sets `robots: noindex` until a human verifies the organization claim [site/[[...path]]/page.tsx:L132-L139]. Other public routes: `/events`, `/events/[slug]` (with Event JSON-LD), `/news`, `/privacy`, `/llms.txt`, `/sitemap.xml` [src/app/(public)/site/*], all `force-dynamic`.

**Renderer.** `PublishedPageRenderer` is a pure server component taking `document`, `media`, `events`, `identity`, `offices`, `shifts`, `theme` [published-page-renderer.tsx:L51-L70](src/components/public/published-page-renderer.tsx); the switch over block types is exhaustive with `assertNever` [L239-L285]. The only client islands on a page are `EnquiryForm` (posts to `/api/enquiries`; Turnstile optional; honeypot + rate limits) [enquiry-form.tsx:L11-L24](src/components/public/enquiry-form.tsx), [api/enquiries/route.ts:L19-L42](src/app/api/enquiries/route.ts), `VideoEmbed` (youtube-nocookie, poster first) [video-embed.tsx:L5-L19](src/components/public/video-embed.tsx), `Reveal` (entrance motion, reduced-motion aware) [reveal.tsx:L5-L30](src/components/public/reveal.tsx). Gallery is a grid with no lightbox by design [published-page-renderer.tsx:L813]. Accessibility is tested with axe [published-page.a11y.test.tsx](src/components/public/published-page.a11y.test.tsx) and contrast arithmetic [section-contrast.test.ts](src/components/public/section-contrast.test.ts).

**Theming.** `organization_profiles` gained `brand_color`, `accent_color`, `accent_on_brand_color`, `logo_asset_id` (six-hex regex CHECKs) [027_council_appearance.sql:L27-L40](supabase/migrations/027_council_appearance.sql) plus `motion_enabled` (040). `councilAppearanceSchema` and `findAppearanceProblems` refuse combinations under 4.5:1 AA [appearance.ts:L13-L162](src/modules/presence/profile/appearance.ts). The renderer sets `--hf-brand`, `--hf-accent-light`, `--hf-accent-brand` on the article root [published-page-renderer.tsx:L74-L82]; `SiteChrome` sets the same on `<main>` [site-chrome.tsx:L43-L57](src/components/public/site-chrome.tsx). Token strings live in [theme-tokens.ts:L18-L66](src/components/public/theme-tokens.ts) (`BRAND_SURFACE`, `ACCENT_TEXT`, `FILLED_BUTTON`, `RAISED_CARD`, etc.) with amber/stone fallbacks. Section backgrounds are named tokens (`soft|raised|brand`) never hexes, so a rebrand needs no document migration [schema.ts:L404-L409]. Display font is one fixed Google font via `display-font.ts`. There is no per-tenant Tailwind config, no font selection, no radius/spacing tokens.

**Navigation.** Module `src/modules/content/navigation`: `PRIMARY_MENU_KEY = "primary"`, items `{label ≤60, linkKind: content|https|mailto|tel, contentItemId, externalValue}`, max 12, "Flat by design" [navigation/index.ts:L3-L63](src/modules/content/navigation/index.ts). RPCs `get_navigation_menu`, `save_navigation_menu`, `list_linkable_pages` [navigation/data/rpc.ts:L9-L17](src/modules/content/navigation/data/rpc.ts); tables `navigation_menus/revisions/revision_items` support `parent_item_key` nesting [002:L527-L605](supabase/migrations/002_publishing_media.sql). The dashboard editor is up/down buttons, "Deliberately not drag and drop" [NavigationEditor.tsx:L19-L24](src/app/(dashboard)/dashboard/navigation/NavigationEditor.tsx). Public read is `list_public_navigation` and, when the menu is empty, `deriveSectionNavigation(blocks)` builds a one-page anchor menu from block anchors [site/[[...path]]/page.tsx:L261-L265], [public-navigation.ts:L1-L90](src/server/content/public-navigation.ts). Header/footer are `PublishedNavigation` (mobile menu, branded variant) and `PublishedFooter` (meeting/contact from identity) [published-navigation.tsx:L27-L60](src/components/public/published-navigation.tsx), [published-footer.tsx:L15-L40](src/components/public/published-footer.tsx).

**Templates and starters.** Registry with two keys: `starter` ("The council page", nine sections) and `hall` ("The parish hall", fourteen sections incl. video, image×2, gallery, how-to-join cards), `DEFAULT_TEMPLATE_KEY = "hall"` [registry.ts:L24-L207](src/modules/content/templates/registry.ts). Both builders take `{organizationName, unitIdentifier, organizationType, blockIds}` and return a whole page [template.ts:L55-L70](src/modules/presence/starter/template.ts); slots used: `home, about, programs, support, events, serve, connect, join, contact` (+ `about_video, serve_photo, photographs, join_photo, how_to_join` in hall) [template.ts:L83-L250], [hall-template.ts:L46-L306](src/modules/presence/starter/hall-template.ts). Block ids are deterministic UUIDv8 derived from the organization id (idempotent provisioning; max 24 template blocks) [block-ids.ts:L1-L76](src/modules/presence/starter/block-ids.ts). Merge rule "a template may add sections and may not rewrite words", keyed on `slot` [merge.ts:L1-L45](src/modules/content/templates/merge.ts); `previewTemplate` returns `adds/keeps/changesAnything` for the picker [apply.ts:L22-L46](src/modules/content/templates/apply.ts). Signup: `bootstrap-organization.ts` calls `provisionStarterSite` with `starterBlockIds(...)` [bootstrap-organization.ts:L17-L18, L165-L174](src/app/actions/bootstrap-organization.ts); provisioning builds the default template, runs `dressTemplatePhotos` with `canImport: false` to drop photo-dependent sections, compiles before writing, creates the draft at `/` and publishes immediately with `runContentPublicationNow` [provision-starter-site.ts:L17-L120](src/server/presence/provision-starter-site.ts). Dressing later imports curated Pexels photos per slot [registry.ts:L116-L180], [dress.ts:L6-L33](src/server/content/templates/dress.ts). Page layouts: `blank`, `council_home` (6 blocks), `about`, `events` [page-layouts.ts:L77-L311](src/modules/content/layouts/page-layouts.ts). Starter rules: never state a fact nobody told us, never publish an instruction, never link to something that might not exist [template.ts:L29-L50].

**Publishing.** `request_content_publication` enqueues a command; the action calls `runContentPublicationNow`, which uses the service-role client to `load_content_publication_work_item`, compile, and `activate_content_publication_work_item` [run-now.ts:L29-L45](src/server/content/publication/run-now.ts), [007_content_publication_worker.sql:L3-L12, L302, L422](supabase/migrations/007_content_publication_worker.sql). Snapshots are immutable and deduped by hash; an out-of-band retry budget of five was added in 019.

### Gaps

- Every public request hits Postgres (page + nav + identity + optionally events/offices/shifts). There is no CDN cache, no `s-maxage`, no purge-on-publish; `NEXT_INC_CACHE_R2_BUCKET` is bound [wrangler.jsonc:L36-L39](wrangler.jsonc) but the public routes opt out with `revalidate = 0`.
- Theme is three colours and a logo. No fonts, radius, spacing, dark mode, or per-page layout variant. No `theme.overrides_json`.
- Flat single menu; no footer menu, no nested menus, no per-page "hide from nav".
- Templates are two KofC-shaped presets; the registry pattern is generic but every builder's copy is council copy.
- Only subdomain tenancy is wired for real; `presence.custom_domain` is a feature key [features.ts] and the host classifier knows `custom`, but the README says custom domains are "not built" [README.md:L117-L121](README.md).
- The `[orgSlug]` path-based routes are retired legacy that fail closed; still in the tree.

---

## 4. Media

### Takeaway

Files live in two private **Cloudflare R2** buckets reached only through Worker bindings (`hallforge-media-originals`, `hallforge-media-variants`); there is no Supabase Storage. Upload is browser → `/api/media/upload` (multipart, same-origin check, capability `media.upload`) → server sniffs the container and re-measures dimensions → R2, with rows in `media_assets`/`media_objects` created by `reserve_media_upload`/`finalize_media_upload` RPCs. Variants (768/1080/1536 px WebP) are generated **in the browser** with `OffscreenCanvas` because sharp is disabled in the workspace and workerd has no codecs; the server never resizes. Delivery is `/media/[assetId]/[variantKey]`, which asks `resolve_media_delivery` whether this site may serve this asset at this size based on the currently published snapshot's media manifest; rights attestation (`council_owned|licensed|permission_granted|public_domain`) is required before an asset is publishable. Free stock search/import from Pexels and Pixabay is built but dormant until API keys are set.

### Findings

- Buckets and bindings: [buckets.ts:L1-L82](src/server/media/buckets.ts), [wrangler.jsonc:L30-L47](wrangler.jsonc). Tables: `media_assets`, `media_objects`, `media_revision_usages`, `media_public_entitlements`, `media_collections` [002:L687-L940](supabase/migrations/002_publishing_media.sql); `media_upload_slots` [015_media_ingest.sql:L37](supabase/migrations/015_media_ingest.sql).
- Accepted MIME: jpeg/png/webp/avif, SVG deliberately refused; `VARIANT_WIDTHS = [768, 1080, 1536]`; `MAX_ORIGINAL_BYTES = 25 MB` [contracts.ts:L16-L37](src/modules/media/contracts.ts). Rights bases and attestation schema [contracts.ts:L39-L54, L148-L186].
- Browser variant generation with EXIF orientation, never upscaling, WebP q0.82 [prepare-image.ts:L3-L90](src/components/media/prepare-image.ts); sequential uploads [upload-pictures.tsx:L17-L25](src/components/media/upload-pictures.tsx).
- Upload route "Every byte passes through here on its way to R2 … Nothing the browser says is believed" [upload/route.ts:L22-L36](src/app/api/media/upload/route.ts); dimension parsers for PNG/JPEG/WebP in [image-bytes.ts](src/modules/media/domain/image-bytes.ts).
- Delivery route [media/[assetId]/[variantKey]/route.ts:L12-L31](src/app/media/[assetId]/[variantKey]/route.ts); published manifest schema and `buildRenderableImage` (srcset from manifest) [manifest.ts:L15-L80](src/modules/media/domain/manifest.ts). Usages are derived server-side from the document by `record_revision_media_usages`, which knows exactly three shapes: `image.data.assetId`, `hero.data.backgroundAssetId`, `gallery.data.images[].assetId` [017:L1-L27](supabase/migrations/017_hero_background_media_usage.sql), [035:L1-L12](supabase/migrations/035_gallery_media_usage.sql). Any new block carrying an asset needs a migration restating that function.
- Stock photos: [stock-photos.ts:L4-L22](src/server/media/stock-photos.ts), search route with per-org rate limit and 24 h cache [stock-search/route.ts:L14-L34](src/app/api/media/stock-search/route.ts); gated by `media.stock_search` and `PEXELS_API_KEY`/`PIXABAY_API_KEY` secrets (roadmap says dormant until set) [phase-roadmap.md:L120-L128](docs/launch/2026-08-19-phase-roadmap.md).
- Storage quota enforcement exists (041) and member preview (039).

### Gaps

- No server-side image processing at all; a browser without `OffscreenCanvas` cannot upload ("Try Chrome, Edge, or Safari") [upload-pictures.tsx:L50-L56]. No AVIF output, no focal-point cropping UI (columns exist: `focal_x/focal_y`), no video/PDF/document assets.
- Media usage derivation is a hand-maintained SQL function per block shape — a maintenance tax every new block pays.
- The rights-attestation gate is good for a fraternal order and will feel heavy to a small business uploading its own product photos; "lapsed right blocks the whole page" is a known risk [publication-pipeline-risks.md:L51-L60](docs/publication-pipeline-risks.md).
- No CDN-level image transformation (Cloudflare Images declined because the bucket is private) [prepare-image.ts:L6-L9].

---

## 5. Design module (Konva canvas)

### Takeaway

The design studio is a **Canva-lite for printed/social collateral**, not a web design tool: an event flyer (US Letter 816×1056), an event banner (1920×1080 for hall TVs and Facebook), and a from-scratch "custom" canvas with text/image/icon elements on an 8 px grid. Flyer and banner are slot-locked templates (council edits words and a photo; geometry is code); custom is the one place the council drags things. Documents are typed JSON in `design_documents` (kinds `flyer|banner|custom|newsletter`), colours are tokens resolved from the council theme, export is PNG download or print. It is a reasonable "make a flyer for the fish fry" feature; it is tangential to a storefront engine and its templates are KofC-shaped.

### Findings

- Schemas: `flyerDocumentSchema` slots `eyebrow/headline/date/time/place/details/cta` + `photoAssetId` [contracts.ts:L19-L36](src/modules/design/contracts.ts); `bannerDocumentSchema` [L112-L126]; `customDocumentSchema` canvas 200-4000 px, ≤60 elements of `text|image|icon` with bounded sizes and `DESIGN_COLOR_TOKENS = brand|accent|ink|paper` [L176-L300]. `emptyFlyer` defaults to "Pancake Breakfast … After the 9am Mass … Knights of Columbus" [L72-L108].
- Storage: `design_documents` JSONB ≤256 KB, RLS forced, RPC-only access [042_design_studio.sql:L15-L47](supabase/migrations/042_design_studio.sql); kinds widened in [049_design_kinds.sql:L19-L26](supabase/migrations/049_design_kinds.sql); RPCs `list/get/save/delete_design_document` [design/data/rpc.ts:L17-L26](src/modules/design/data/rpc.ts).
- Editors: `flyer-editor`, `banner-editor`, `custom-editor` with `react-konva` loaded via `next/dynamic ssr:false` [custom-editor.tsx:L58-L70](src/components/dashboard/designs/custom-editor.tsx); geometry is pure arithmetic in `flyer-layout.ts`/`banner-layout.ts` (tested) [flyer-layout.ts:L1-L40](src/components/dashboard/designs/flyer-layout.ts); icon set in `icons.ts`/`icon-nodes.ts` (Lucide paths). Autosave 1.2 s debounce, 5 retries [use-design-autosave.ts:L21-L24](src/components/dashboard/designs/use-design-autosave.ts). Export via canvas `toDataURL` → PNG or print window; tainted-canvas failure named plainly [use-design-export.ts:L1-L50](src/components/dashboard/designs/use-design-export.ts). Size presets for custom: Letter tall/wide, Square, Wide screen, Print banner [new-design-picker.tsx:L28-L34](src/components/dashboard/designs/new-design-picker.tsx).
- Gated behind `design.studio` feature [designs/page.tsx:L15-L20](src/app/(dashboard)/dashboard/designs/page.tsx). The founder's direction: banner → flyer → brochure → newsletter, templates carry the design, councils touch slots [direct-manipulation-editors.md:L46-L58](docs/decisions/2026-09-01-direct-manipulation-editors.md).

### Gaps

- Two real templates; no brochure, no multi-page; newsletter is a separate DOM-print module that only shares the table.
- No PDF export (print dialog only), no CMYK/bleed, no shape/line primitives, no layers panel, no text styles beyond size/weight/align.
- Entirely KofC-flavoured starter copy; no business templates (menu, price list, promo, social post sizes beyond 1080² and 1920×1080).
- Not useful for small businesses as-is; marginally useful ("quick promo graphic") only after templates are replaced.

---

## 6. Block-by-block comparison with engine doc section 5

Source list: `/home/user/consultingTemplate/docs/engine-architecture.md` §5.1 (hero, announcement_bar, text, cta, hours, map, contact_form, quote_request, menu, product_grid, product_detail, gallery, reviews, booking, faq, team, testimonials, vehicle_selector, embed) plus the task's list.

| Engine block | HallForge equivalent | Status |
|---|---|---|
| hero | `hero` (eyebrow, heading, lead, ≤2 actions, background asset, ≤4 stats) | **Exists**, close match; add media focal point, overlay/variant |
| announcement bar | `callout` (emphasis note/important, one action) rendered as a section, not a site-wide bar; no schedule | **Partial** — needs a chrome-level bar and `from/until` |
| text | `rich_text` (ProseMirror JSON, not Markdown) | **Exists**; stricter than Markdown, fine |
| cta | `call_to_action` (≤2 actions; href types site/https/mailto/tel) | **Exists**; no SMS/booking/order link kinds, phone is typed not pulled from locations |
| hours | none; `contact_details.showMeeting` renders `meetingSummary` free text from the org profile | **Replace** — no structured hours model |
| map | none; `contact_details` renders a "Get directions" link to `directionsUrl` | **Missing** (link only) |
| contact form | `enquiry_form` + `/api/enquiries` (honeypot, rate limit, Turnstile optional, email notify) with **fixed** topics (membership/volunteering/…) | **Exists**; topics and consent purposes are KofC; no custom fields, no uploads |
| quote request | none | **Missing** |
| booking | none | **Missing** |
| reviews | none | **Missing** |
| menu (restaurant) | none; `card_grid` could fake a price list (headline/body/footnote) | **Missing** |
| product grid / product detail | none; feature keys `store.catalog`/`store.orders` exist but no blocks or catalog UI in the content tree | **Missing** |
| gallery | `gallery` ≤24 images with alt/caption, no lightbox | **Exists** |
| FAQ | none; `rich_text` with headings is the fallback | **Missing** |
| team | `officers` reads the council's office roster (no names stored; per-person consent) | **Partial** — the dynamic-roster pattern is right; the data source is KofC offices |
| testimonials | none; `card_grid` text emphasis is a weak stand-in | **Missing** |
| embed | `video` (YouTube ids only, youtube-nocookie) | **Partial** — allowlist of one provider |
| vehicle_selector | none | **Missing** (vertical-specific anyway) |
| (extra) | `card_grid` (programs/stats), `upcoming_events`, `volunteer_shifts`, `image` (picture + words), `divider` | HallForge has these; `upcoming_events` and `card_grid` are generic and valuable |

Count: of the 17 named in the task, 5 exist in a usable form (hero, text, gallery, contact form, CTA), 3 partial (announcement bar, team, embed/video), 9 missing (menu, product grid, product detail, hours, quote request, booking, reviews, map, FAQ, testimonials — hours/map have only a profile-level stand-in).

---

## Generic vs Knights-of-Columbus-specific (content, editor, rendering, media, design)

**Generic, reusable as-is**
- `blocks_v1` schema machinery: identity, slots, section presentation, link safety, rich-text subset, document caps [schema.ts].
- The authoring-permissive / publication-strict split and the isomorphic compiler [compiler.ts].
- Postgres pipeline: items, routes, revisions, conflicts, reviews, publication commands, immutable snapshots, idempotency [002, 005, 006, 007].
- Autosave controller + IndexedDB recovery [modules/content/autosave].
- Live preview architecture (same compiler, same renderer).
- Theme-token approach (`--hf-*` variables with fallbacks, named section backgrounds).
- Navigation model and RPCs (flat, but tables allow nesting).
- Template registry/merge/apply (slot-keyed, never rewrites words) and deterministic block ids.
- Media ingest, R2 delivery gate, manifest-driven srcset, stock import pipeline.
- `orgtype` vocabulary layer (unit noun, identifier label) — explicitly built so "council" is not hard-coded [orgtype/index.ts:L1-L9](src/modules/orgtype/index.ts).

**KofC-specific (must change)**
- Starter and hall template copy: "Catholic men serving our parish…", "Into the Breach trailer", "Any practical Catholic man of eighteen…" [template.ts:L89, L168], [hall-template.ts:L54, L96-L97, L234].
- `supported-programs.ts` (Coats for Kids, Food for Families, … sourced from kofc.org) [supported-programs.ts:L3-L60].
- `card-icons.ts` first four keys are the Faith in Action pillars (`faith/family/community/life`) [card-icons.ts:L15-L24].
- `officers` block semantics (council offices, "Grand Knight — vacant") [published-page-renderer.tsx:L1421, L1474]; `volunteer_shifts` and `contact_details` read council-shaped records (`meetingSummary`, `meetingLocation`).
- `enquiry_form` fixed topics ("Joining the council", "Helping at an event") [enquiries/index.ts:L12-L18](src/modules/people/enquiries/index.ts).
- `llms.txt` route hard-codes "A Knights of Columbus council … Catholic men" [llms.txt/route.ts:L17-L22](src/app/(public)/site/llms.txt/route.ts).
- Page layout and post-template prompt copy ("Council home page", "What the council does") [page-layouts.ts:L90-L206].
- Design studio defaults ("Pancake Breakfast", "After the 9am Mass") and the `DesignNoun` copy.
- Fallback organization type is "Knights of Columbus Council" [orgtype/contracts.ts:L103-L105](src/modules/orgtype/contracts.ts).
- Component/file naming (`CouncilAppearance`, `RenameCouncilForm`, `PublicCouncilOffice`) — cosmetic but widespread; the KofC-vocabulary grep hits 11 files in `src/modules/content`, 11 in `src/components/public`, 9 in the editor, mostly comments and labels.

---

## Reusable for a small-business storefront engine? Verdict per module

| Module | Verdict | Why |
|---|---|---|
| `modules/content/blocks-v1` (schema, rich text, link safety) | **Reuse with changes** | The substrate is exactly what §5 asks for (Zod, strict, versioned, typed). Needs: new block types (menu, product grid/detail, hours, map, FAQ, testimonials, booking, reviews, announcement bar), a `version` migration mechanism, optional block scheduling. Retire `officers`/`volunteer_shifts` or generalise to "team"/"openings". |
| `server/content` (compiler, publication worker, public query, repository, actions) | **Reuse with changes** | Pipeline is production-grade and tenant-safe. Needs a block-level tool API (insert/update/move/remove over the working copy) and an audit log for AI edits; swap Supabase RPC client for whatever DB the engine lands on (the engine doc says D1 — this code is Postgres SECURITY DEFINER plpgsql end to end, which is the single largest porting cost). |
| `modules/content/autosave` | **Reuse as-is** | Generic, typed, tested; no domain vocabulary. |
| `modules/content/navigation` + `NavigationEditor` | **Reuse with changes** | Add nesting (tables already allow it), a footer menu, per-page nav flag. |
| `modules/content/templates` + `layouts` + `post-templates` | **Reuse the mechanism, replace the content** | Registry/merge/apply/slot ids are generic; every builder and prompt string is council copy. Write per-vertical presets (restaurant, salon, contractor, retail) with the same `build(input)` shape. |
| `modules/presence/starter` + `server/presence/provision-starter-site` | **Reuse the mechanism, replace the content** | Deterministic ids and publish-at-signup are what remote onboarding needs; the page it publishes is KofC. |
| `modules/presence/profile` (identity, appearance, contrast) | **Reuse with changes** | Organization identity + three-colour theme + AA refusal is a sound base; add structured `locations/hours`, fonts, more tokens; rename council→organization nouns via `orgtype`. |
| `components/public` (renderer, chrome, theme tokens, enquiry form, video, reveal) | **Reuse with changes** | Keep the renderer pattern, chrome, token strings, a11y tests; add renderers for the new blocks; replace enquiry topics with form definitions; `block-renderer.tsx`, `public-header.tsx`, `public-footer.tsx` and the `[orgSlug]` routes are legacy — **drop**. |
| `components/dashboard/pages` (editor) | **Reuse short-term, plan to replace** | Works, tested, accessible; but forms-beside-preview is not the §5.5/§11 direction and the founder already calls it a stepping stone. Keep `block-helpers`, `media-picker`, `RichTextField`, `LivePreview`; replace the arrangement shell (Puck or custom) when direct manipulation ships. |
| `components/page-editor` (prototype) | **Drop** | Retired; only `/demo` uses it. |
| `modules/media` + `components/media` + `server/media` + media routes | **Reuse with changes** | R2-through-Worker, validate-then-store, manifest srcset and stock import are directly useful. Relax the rights-attestation gate to a self-attestation default for business-owned photos; add server-side resizing when a Node/Images path exists; generalise `record_revision_media_usages` to a generic "walk the document for assetId" rather than per-block SQL arms. |
| `modules/design` + `components/dashboard/designs` (Konva) | **Drop for the engine; keep as an optional add-on** | Valuable to councils, not to a storefront. If kept, replace flyer/banner templates with business sizes (Instagram, story, menu board) and drop the KofC defaults. |
| `src/server/root.ts` (tRPC) | **Replace** | Empty by design; the engine wants a typed tool surface here (or an equivalent), built over the repository functions. |

**Bottom line for the engine question:** HallForge already has the hard, boring 60% of §5 (typed document, strict compile, immutable publish, tenant-safe rendering, media gate, templates that merge, autosave with recovery). What it does not have is the storefront vocabulary (products, menus, hours, locations, bookings, reviews), a block-level typed mutation API, a schema migration path, a direct-manipulation editor, or any edge caching. Starting from it means porting plpgsql if the engine must be D1, and scrubbing council copy from roughly forty files; starting from scratch means rebuilding everything in the first list.
