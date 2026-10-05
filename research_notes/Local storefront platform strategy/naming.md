# Name availability and brand fit for the storefront platform (checked 2026-10-05)

Scope: 30 founder-supplied candidates (Latin, Hebrew, Aramaic, mercantile English) plus 33 new candidates generated in the same spirit, checked for .com status, npm and GitHub collisions, obvious software/commerce conflicts, pronunciation and spelling risk, hidden meanings, and the "pizza shop" / "VC room" tests.

## How the checks were actually run (read this before trusting any cell)

### Takeaway
npm and GitHub collisions were checked DIRECTLY and are reliable. Domain (.com and alt-TLD) status could NOT be checked directly from this environment: every RDAP, WHOIS and DNS-over-HTTPS host (rdap.verisign.com, rdap.org, registrar RDAP mirrors, dns.google, cloudflare-dns.com, whois.com, who.is) and even direct fetches of the candidate domains were refused by the sandbox egress proxy with "403 to CONNECT (policy denial)", for both curl and the WebFetch tool. Domain findings below are therefore INDIRECT (web-search evidence of a live site or business at that domain) and are labelled "unverified" where no evidence surfaced. The founder should re-run the script at the end of this section before buying anything.

### Cited Findings
- Egress-proxy status endpoint recorded `connect_rejected ... gateway answered 403 to CONNECT (policy denial or upstream failure)` for rdap.verisign.com, rdap.org, rdap.identitydigital.services, pubapi.registry.google, rdap.godaddy.com, rdap.namecheap.com, rdap.cloudflare.com, rdap.markmonitor.com, rdap.iana.org, dns.google, cloudflare-dns.com, api.domainsdb.info, www.whois.com, who.is, example.com, en.wikipedia.org, statio.com and a nonsense control domain — i.e. the proxy is an allow-list, not a per-domain block, so a 403 carries no information about registration — [proxy status endpoint, local](http://127.0.0.1:38051/__agentproxy/status); the README states "403 / 407 from the proxy: the destination host is not allowed by your organization's egress policy ... Do not retry or route around it" — [/root/.ccr/README.md](file:///root/.ccr/README.md)
- `curl -s -o /dev/null -w "%{http_code}" https://registry.npmjs.org/NAME` worked directly (registry.npmjs.org is on the proxy's noProxy list): 200 = package exists, 404 = free. Control: `zzqqxxyy123abc` returned 404; `statio` returned 200 — [npm registry](https://registry.npmjs.org/statio)
- `api.github.com/users/NAME` via curl and `gh api users/NAME` both returned 403 ("sessions are bound to their configured repositories"), so GitHub collisions were checked with the GitHub Search API through the GitHub MCP server (`search_users`, query `NAME in:login`, and `NAME in:login type:org`). GitHub logins are case-insensitive and users and organizations share one namespace, so any exact login hit (e.g. `ManSio`) means `mansio` is unavailable; a result set without the exact login (e.g. `waystead` returned 0 results) means it is free. Example hit: [github.com/statio](https://github.com/statio); example miss: query `waystead in:login` returned `total_count: 0`.
- WebSearch was used for domain and conflict evidence (query form `"NAME.com"`); the session's shared WebSearch budget (200 calls) was exhausted after the first ~60 domain queries, so the second-round names (Posthouse, Mercantry, Kaupa, Limen, Portus, Loggia, Duka, Comptoir, Nundina, Lonja, Entrepot, Wares, Chandlery, Feira, Winkel) and the dedicated "NAME startup / NAME trademark" conflict queries for Statio, Vicus, Taberna, Tagar, Pundak and Caupo were NOT run. Those cells say "not searched".
- No USPTO/TESS, Crunchbase, Product Hunt or app-store page could be fetched directly (all egress-blocked); trademark evidence below comes only from search-result snippets (CB Insights, Justia trademark pages, Dealroom, company registries) and is flagged as such.

### Inferences
- Treat every ".com: unverified" as "probably registered but possibly parked/for sale" — short dictionary-like Latin words almost never have a free .com; the useful question is whether it is cheaply acquirable, which only an RDAP + registrar lookup can answer.
- Treat "npm 404" and "GitHub free" as reliable as of 2026-10-05.

### Gaps
- Alt-TLD status (.co, .io, .app, .store, .shop) and suffix forms (get-NAME.com, try-NAME.com, NAME-hq.com) are unverified for EVERY name; the founder's script below covers them.
- USPTO live-mark search for the shortlist is not done; the founder should run TESS/Trademark Center queries for the top 3 before committing (class 9, 35, 42).

### Re-verification script for the founder (run from any unrestricted machine)
```bash
# usage: bash verify.sh statio shopstead pundak ...
for n in "$@"; do
  for tld in com co io app store shop; do
    case $tld in com) u="https://rdap.verisign.com/com/v1/domain/$n.$tld";; *) u="https://rdap.org/domain/$n.$tld";; esac
    printf "%-14s .%-5s -> %s\n" "$n" "$tld" "$(curl -s -o /dev/null -w '%{http_code}' "$u")"   # 404 = unregistered, 200 = registered (maybe parked)
  done
  for v in get$n try$n ${n}hq ${n}commerce; do
    printf "%-14s .com   -> %s\n" "$v" "$(curl -s -o /dev/null -w '%{http_code}' "https://rdap.verisign.com/com/v1/domain/$v.com")"
  done
  printf "%-14s github -> %s\n" "$n" "$(curl -s -o /dev/null -w '%{http_code}' "https://api.github.com/users/$n")"      # 404 = free
  printf "%-14s npm    -> %s\n" "$n" "$(curl -s -o /dev/null -w '%{http_code}' "https://registry.npmjs.org/$n")"    # 404 = free
done
# Then: https://tmsearch.uspto.gov (live marks, classes 9/35/42), crunchbase.com, producthunt.com, apps.apple.com for the top 3.
```

## Which names have a free or cheaply acquirable .com (or acceptable alternative) AND no live trademark conflict in software/commerce?

### Takeaway
No candidate was proven to have a free .com (RDAP blocked). On the evidence that could be gathered, the cleanest names are the two coined "-stead" words (Shopstead, Waystead: GitHub user+org free, npm free, no business surfaced at the .com, no software project of that name) and Caupo (GitHub free, npm free, nothing surfaced); the strongest BRANDS with manageable collisions are Statio and Pundak; Mansio, Taberna, Mercantry and Nundina are second tier. Roughly two-thirds of the founder's list is dead on arrival because a live commerce/SaaS company already uses the name (Mercatus, Waystation, Sundry, Provi(s), Wayfair/Wayfare, Depot, Dukkan, Torg, Stoa, Mercat(o), Merx, Fiscus) or because the handle namespace is exhausted.

### Ranked shortlist (12) with per-name evidence

1. **Statio** (Latin: a stopping place / post station on Roman roads; root of "station")
   - .com: no live site or business surfaced for "statio.com" — search returned only the Latin term and a PyPI package — [Wikipedia: Statio (Roman)](https://en.wikipedia.org/wiki/Statio_(Roman)); [PyPI statio](https://pypi.org/project/statio). Registration status unverified.
   - npm: `statio` TAKEN (200) — [registry](https://registry.npmjs.org/statio); `statiohq` FREE (404), `statio-commerce` FREE (404) — [registry statiohq](https://registry.npmjs.org/statiohq).
   - GitHub: user `statio` TAKEN — [github.com/statio](https://github.com/statio); no exact org, nearest `StationA`, `stationfy`. Repo search `statio in:name` returns only "station" projects (ground-station, space-station-14), no product named Statio.
   - Conflicts: Python stats library "statio" (sliding-window computations) — [PyPI](https://pypi.org/project/statio). Dedicated startup/trademark query not run (budget). Confusable with Statista (statista.com) in search results.
   - Pronunciation/spelling: "STAH-tee-oh" vs "STAY-shee-oh" ambiguity; heard aloud most people will type "statio" or "stacio". 3 syllables.
   - Meaning: exactly the brief (waystation/outpost), non-military (the Roman statio was a post/customs/market station, distinct from castrum).
   - Pizza-shop test: pass ("it's like 'station'"). VC-room test: strong pass (same register as Stripe/Strata/Toast).
2. **Shopstead** (coined: shop + Old English "stead" = place, as in homestead)
   - .com: nothing surfaced for "shopstead.com" (no live site) — unverified.
   - npm: FREE (404) — [registry](https://registry.npmjs.org/shopstead). GitHub: user AND org FREE (`shopstead in:login` → 0 results; `type:org` → 0). Repo search: 0 repos named shopstead.
   - Conflicts: none surfaced. Risk: "Shop-" prefix sits next to Shopify's SHOP family of marks; USPTO check essential.
   - Pronunciation/spelling: unambiguous, 2 syllables, spells itself.
   - Pizza-shop test: strong pass. VC-room test: moderate (descriptive, slightly derivative of Shopify/ShopKeep, but "stead" gives the homestead/general-store warmth).
3. **Pundak** (Hebrew פונדק: roadside inn / waystation, from Greek pandocheion)
   - .com: search for "pundak.com" returned only Indonesian health articles ("pundak" = shoulder in Indonesian/Malay); no business at the domain surfaced — [alodokter](https://www.alodokter.com/berbagai-penyebab-sakit-pundak-dan-cara-mengatasinya). Unverified.
   - npm: FREE (404) — [registry](https://registry.npmjs.org/pundak). GitHub: user `Pundak` TAKEN (2022 profile-config account only) — [github.com/Pundak](https://github.com/Pundak); `pundakhq` not checked. A 0-star Indonesian student e-commerce repo named "pundak" exists (May 2026) — [sengkalium/pundak](https://github.com/sengkalium/pundak).
   - Conflicts: none surfaced in commerce/SaaS; dedicated conflict query not run.
   - Pronunciation/spelling: "PUN-dak", spells itself. 2 syllables.
   - Meaning: the Hebrew word for exactly the thing (inn/waystation on a trade road). Hidden meaning: Indonesian "shoulder" (benign).
   - Pizza-shop test: pass (friendly, short). VC-room test: pass (distinctive, non-cutesy; Toast/Gusto register).
4. **Waystead** (coined: way + stead = "the place along the way")
   - .com: nothing surfaced — unverified. npm FREE (404) — [registry](https://registry.npmjs.org/waystead). GitHub user and org FREE (0 results). One 0-star CSS repo `waystead-parent` (Jul 2026) — [ronlai0916/waystead-parent](https://github.com/ronlai0916/waystead-parent).
   - Conflicts: none surfaced.
   - Risk: said quickly it collides with "wasted" (WAY-sted / WAY-stid); expect the joke in every VC meeting. Otherwise 2 syllables, spells itself.
   - Pizza-shop test: pass. VC-room test: marginal because of the pun.
5. **Caupo** (Latin: shopkeeper / innkeeper / small trader)
   - .com: no results at all for "caupo.com" — unverified. npm FREE (404) — [registry](https://registry.npmjs.org/caupo). GitHub: no exact login; nearest `caupo-ab`, `Caupo-RP`, `caupo-net` — [caupo-ab](https://github.com/caupo-ab); only hobby repos (a Firebase demo, a thesis) — [mayupandey/caupo](https://github.com/mayupandey/caupo).
   - Conflicts: none surfaced; dedicated query not run.
   - Risk: English speakers will hear "cow-po"/"cow poo"; classical "KOW-po". Spelling when heard: "cowpo"/"kaupo". Meaning is the best of the set (the Roman small-business owner).
   - Pizza-shop test: marginal (giggle risk). VC-room test: marginal for the same reason.
6. **Mansio** (Latin: official stopping place / inn on Roman roads; root of "mansion")
   - .com: a live French startup "Mansio" (country-house vacation rentals; founded 2022; HQ Aulnay-sous-Bois; raised $2.7M from Quiet Capital and Founders Future) is the dominant result — [CB Insights](https://www.cbinsights.com/company/mansio-1); [Dealroom](https://cednc.dealroom.co/companies/mansio). A Belgian entity "Mansio" (BCE 0404127239) also exists — [busibee](https://busibee.be/en/0404127239-mansio). Mansio GmbH (Germany) has a GitHub org — [Mansio-GmbH](https://github.com/Mansio-GmbH). Registration of mansio.com itself unverified.
   - npm FREE (404) — [registry](https://registry.npmjs.org/mansio); `mansiohq` FREE (404). GitHub user `ManSio` TAKEN (case-insensitive) — [github.com/ManSio](https://github.com/ManSio).
   - Conflicts: hospitality startup (EU) — adjacent to the pizza-shop/restaurant audience; not a US software mark on the evidence found.
   - Pronunciation: "MAN-see-oh", easy; connotes "mansion" (luxury/real-estate drift).
   - Pizza-shop test: pass. VC-room test: pass, but expect "isn't that the French rental company?"
7. **Mercantry** (coined: mercantile + -ry, "the craft of merchants", cf. chandlery)
   - .com: not searched (budget). npm FREE (404) — [registry](https://registry.npmjs.org/mercantry). GitHub: org `mercantry` TAKEN, created 2026 with one TypeScript repo "registry" — [github.com/mercantry](https://github.com/mercantry); [mercantry/registry](https://github.com/mercantry/registry) — someone may be building a product under this name.
   - Pronunciation: "MER-can-tree", spells itself after one hearing. 3 syllables.
   - Pizza-shop test: pass. VC-room test: good (category-defining, non-cutesy).
8. **Taberna** (Latin: the shop/stall units fronting Roman streets; later "tavern")
   - .com: no site surfaced; the name is used by many restaurants (La Verne CA, Fairfield CT, Island Park NY, Washington DC, Lisbon) — [OpenTable](https://www.opentable.co.uk/r/taberna-tapas-fairfield); [Washingtonian](https://www.washingtonian.com/restaurantreviews/2199.html). Unverified.
   - npm FREE (404). GitHub user `taberna` TAKEN — [github.com/taberna](https://github.com/taberna). A dormant PHP "tabernacms — constructor for creating online shops" (2013, 10 stars) exists — [kisanetik/tabernacms](https://github.com/kisanetik/tabernacms).
   - Hidden meaning: Spanish/Portuguese "taberna" = tavern/pub (alcohol connotation for Etsy sellers; fine for pizza shops).
   - Pizza-shop test: pass. VC-room test: marginal ("you named a SaaS after a bar?").
9. **Nundina** (Latin nundinae: the every-ninth-day market; Nundina, goddess of the ninth day)
   - .com: not searched. npm FREE (404) — [registry](https://registry.npmjs.org/nundina). GitHub FREE (only `nundina1st` and org `Nundinae` exist — [Nundinae](https://github.com/Nundinae)).
   - Pronunciation: "nun-DEE-na"; 3 syllables; sounds like a person's name.
   - Pizza-shop test: marginal (obscure). VC-room test: marginal.
10. **Tagar** (Aramaic תגר: merchant)
   - .com: no site surfaced; results were Tag-AR (Univ. of Miami) and the Tagaroo WordPress plugin — [xr.miami.edu](https://www.xr.miami.edu/research/faculty-projects/completed-projects/tag-ar/index.html). Unverified.
   - npm FREE (404). GitHub user `Tagar` TAKEN (active, 28-star repos) — [github.com/Tagar](https://github.com/Tagar).
   - Risks: "TAG-ar" vs "TAY-gar"; Hebrew homonym (to contest/challenge), Indonesian "tagar" = hashtag (general knowledge, unverified).
   - Pizza-shop test: pass. VC-room test: marginal (reads like "tag"/"tagger").
11. **Townway** (coined)
   - .com: nothing surfaced ("Town of Woodway", street names) — unverified. npm FREE (404). GitHub user `townway` TAKEN (since 2010) — [github.com/townway](https://github.com/townway).
   - No conflicts surfaced; bland, low distinctiveness; 2 syllables, spells itself.
   - Pizza-shop test: pass. VC-room test: weak (sounds like a municipal road program).
12. **Posthouse** (a post house = the inn where post-horses were changed, i.e. a mansio)
   - .com: not searched. npm FREE (404). GitHub user `posthouse` TAKEN and `posthousehq` TAKEN — [github.com/posthouse](https://github.com/posthouse); [posthousehq](https://github.com/posthousehq).
   - Meaning is ideal; name is common for hotels/restaurants; handle namespace already crowded.

### Full table of every candidate checked
Columns: .com = indirect web-search evidence only; npm and GitHub = direct checks 2026-10-05; "login" means exact user/org login. Alt TLDs (.co/.io/.app/.store/.shop) are unverified for all rows.

| Name | Origin / meaning | .com (indirect) | npm | GitHub login | Conflicts surfaced | Pronounce / spell risk | Verdict |
|---|---|---|---|---|---|---|---|
| Statio | Latin road post-station | no site surfaced; unverified | taken | taken (user) | PyPI "statio" lib | medium (sta-tee-oh / stay-shee-oh) | SHORTLIST #1 |
| Mansio | Latin road inn | French rental startup Mansio ($2.7M) likely holds brand; unverified | free | taken (ManSio); org Mansio-GmbH | EU hospitality startup; Mansio GmbH | low | SHORTLIST #6 |
| Taberna | Latin shop/stall; tavern | restaurants everywhere; unverified | free | taken | tabernacms (dormant shop CMS) | low; "tavern" meaning | SHORTLIST #8 |
| Emporia | Latin/Greek markets (pl.) | Emporia Energy, Emporia KS dominate | free | taken (org emporia, emporiaenergy) | Emporia Energy (EV chargers), Emporia KS, Emporia State Univ. | low | DEAD |
| Mercatus | Latin market | mercatus.com = Mercatus Technologies grocery e-commerce platform | free | taken (org Mercatus) | DIRECT: e-commerce storefront platform (Toronto, 2002); Mercatus Center think tank | low | DEAD |
| Vicus | Latin village / civilian settlement by a fort | no site; Belgian co. VICUS; unverified | free | taken | VICUS (BE); SIM-VICUS energy software | high ("vicious") ; fort adjacency | LOW |
| Tessera | Latin tile / token (tessera hospitalis) | Tessere mosaic (IT) dominates | taken | taken | mosaic-tile brand; (Tessera Technologies semiconductor — general knowledge) | low | DEAD |
| Fiscus | Latin treasury | Fiscus = Dutch bookkeeping SaaS | taken | taken | DIRECT: SMB finance software | low; "fiscus" = the taxman in NL/DE (general knowledge) | DEAD |
| Mazon | Hebrew food/sustenance | search returns only Amazon (one-letter typo) | free | taken | Amazon confusability; (MAZON hunger nonprofit — general knowledge) | low | DEAD |
| Machsan | Hebrew warehouse | nothing surfaced; unverified | taken | taken | none | high (guttural "ch") | DEAD |
| Miskar | Hebrew commerce (variant) | nothing; Polish co. MISKAR | free | taken | Polish company | high | DEAD |
| Mischar | Hebrew commerce | nothing surfaced | free | taken (MischaR) | none | high; reads "Mischa R" | DEAD |
| Matmon | Hebrew hidden treasure | nothing; "Matmonim" Daf Yomi podcast | free | taken (Matmon, Matmon-Africa) | none in commerce | low | LOW (meaning off-brief) |
| Machoz | Hebrew district | nothing surfaced | free | taken (MachoZ) | none | high ("macho") | DEAD |
| Shuka | Aramaic market | nothing surfaced | taken | taken | Shukha (IL jewelry), Shukr (UK fashion) nearby | medium ("shook-a") | LOW |
| Tagar | Aramaic merchant | nothing surfaced; unverified | free | taken | Tag-AR, Tagaroo nearby | medium | SHORTLIST #10 |
| Pinta | Aramaic (also Columbus ship) | Pinta image editor dominates; CB Insights lists a co. "Pinta" | taken | taken | OSS image editor; startup | Spanish slang ("pinta" = appearance; "pint") | DEAD |
| Piska | Aramaic section | nothing surfaced | free | taken | none | Slavic childish/vulgar slang risk (general knowledge) | DEAD |
| Waystation | English | WayStation AI website-builder for agencies; Waystation logistics API; WayStation MCP | free | taken (org WayStation, waystation-ai) | DIRECT: AI website creation platform | low | DEAD |
| TradePost | English | Tradepost.ai, TradersPost, Minecraft plugin | free | taken | trading-signal SaaS; broker automation | low | DEAD |
| Packhouse | English produce shed | Packhouse Technology (produce logistics); Pack House LLC marks | taken | taken | logistics SaaS; box trademarks | low | LOW |
| Quay | English wharf | quay.com = Quay Australia sunglasses | taken | taken (org quay = container registry) | retail brand; Red Hat Quay | high ("key") | DEAD |
| Pike | English turnpike | pike.com = Pike Electric (NYSE); Pike13 SaaS | taken | taken | utility co.; client-mgmt SaaS | low | DEAD |
| Wayfare | English | Wayfare app, Wayfare Ventures, Framer template | taken | taken | Wayfair confusability | low | DEAD |
| Provis | provisions | Provi.com ($125M raised, $750M val.) alcohol marketplace | free | taken | near-identical marketplace name | low | DEAD |
| Sundry | English | sundry.com = Sundry Clothing (acq. Digital Brands Group) | taken | taken | DIRECT: e-commerce apparel | low | DEAD |
| Commissary | English | commissaries.com = Defense Commissary Agency | free | taken | military commissary; prison store | low | DEAD (military) |
| Depot | English | Depot1.com marks; Home Depot | taken | taken (org depot = depot.dev) | CI/build SaaS; Home Depot | low | DEAD |
| Mercantile | English | Mercantile Bank, Mercantile Stores, The Mercantile | taken | taken | descriptive; banks | 4 syllables | LOW |
| Townway | coined | nothing surfaced | free | taken | none | low | SHORTLIST #11 |
| Stoa | Greek merchants' colonnade | Stoa fintech ($2.4M, Jul 2026), Stoa meditation app, STOA debate | taken | taken | fintech + app | low | DEAD |
| Pundak | Hebrew inn/waystation | nothing (Indonesian "shoulder") | free | taken (Pundak) | tiny student e-com repo | low | SHORTLIST #3 |
| Dukkan | Arabic/Hebrew/Turkish shop | Dukkan (Dubai, 2021) digitizes small local businesses; Dukaan (IN) store builder | free | taken (org dukkan) | DIRECT, same mission | low | DEAD |
| Serai | caravanserai | nothing surfaced | taken | taken (org serai, serai-dex) | DEX crypto project; (HSBC "Serai" B2B platform — general knowledge) | medium (seh-RYE; sarai/seray) | LOW |
| Fondaco | Venetian merchants' warehouse-inn | Fondaco SGR (IT asset mgr), Fondaco AB (SE) | free | taken | finance co. (EU) | medium | LOW |
| Kaupang | Norse trading town | Wikipedia/tourism only | free | taken | none in software | high (spelling) | LOW |
| Torg | Norse/Swedish market square | torg.com = B2B food marketplace (Berlin) | free | taken | DIRECT marketplace | low | DEAD |
| Wic | Old English trading settlement | not searched | taken | not checked | WIC federal nutrition program (general knowledge) | — | DEAD |
| Mutatio | Roman relay station | not searched | free | not checked | — | "mutation" | DEAD |
| Caupo | Latin shopkeeper | nothing surfaced | free | FREE (user+org) | hobby repos only | high ("cow-po") | SHORTLIST #5 |
| Nundina | Latin market day | not searched | free | FREE | none | medium | SHORTLIST #9 |
| Merx | Latin goods | merx.com = MERX Canadian tenders; MerXu EU B2B; Merx Kirby shop plugin | free | taken | B2B platforms + e-commerce plugin | low | DEAD |
| Hanut | Hebrew/Arabic shop | Hanut Sales Corp (IN) | free | taken | importer | low | LOW |
| Shuk | Hebrew market | nothing surfaced | free | taken | restaurants (general knowledge) | low ("shook") | LOW |
| Waystead | coined | nothing surfaced | free | FREE (user+org) | one 0-star repo | "wasted" pun | SHORTLIST #4 |
| Tradestead | coined | tradestead.com = Shenzhen wholesale exporter (search summary) | free | taken | B2B exporter | low | DEAD |
| Shopstead | coined | nothing surfaced | free | FREE (user+org) | none | low | SHORTLIST #2 |
| Mercat | Scots market (Mercat Cross) | results dominated by Mercato grocery marketplace ($26M) | free | taken | confusable with Mercato | low | DEAD |
| Comptoir | French counter / trading post | not searched | free | taken | — | high (comp-twahr) | DEAD |
| Entrepot | French/English warehouse port | not searched | taken | taken | — | accent/spelling | DEAD |
| Stow | Old English place | Stow (storage startup); stow.com per search = ski resort site | taken | taken | storage startup | low | DEAD |
| Vendo | Latin "I sell" | not searched | free | not checked | (Vendo payments — general knowledge) | low | not pursued |
| Shingle | "hang out a shingle" | not searched | taken | not checked | — | shingles (disease) | DEAD |
| Landing | riverboat landing / landing page | not searched | taken | not checked | (Landing flexible-living startup — general knowledge) | — | DEAD |
| Portage | carrying goods between waters | not searched | taken | not checked | Gentoo Portage (general knowledge) | — | DEAD |
| Waypost | English signpost | nothing surfaced | taken | taken | Waypost Advisors | low | LOW |
| Tradehouse | English | Tradehouse Enterprises, TradehouseFlow SaaS | free | taken | generic | low | LOW |
| Almacen | Spanish warehouse/store | not searched | taken | not checked | — | accent | not pursued |
| Lonja | Spanish merchants' exchange | not searched | free | taken | — | "LON-ha" | LOW |
| Caupona | Latin inn | not searched | free | not checked | Caupona Minecraft mod | 3 syl | not pursued |
| Macellum | Latin market hall | not searched | free | not checked | — | 3 syl | not pursued |
| Emporion | Greek trading post | not searched | free | not checked | — | 3 syl | not pursued |
| Arca | Latin strongbox | not searched | taken | not checked | — | — | not pursued |
| Cella | Latin storeroom | not searched | taken | not checked | — | — | not pursued |
| Posthouse | post-horse inn | not searched | free | taken (+ posthousehq) | hotels/restaurants (general knowledge) | low | SHORTLIST #12 |
| Mercantry | coined merchant-craft | not searched | free | taken (org, 2026, one TS repo) | possible new project | low | SHORTLIST #7 |
| Kaupa | Old Norse "to buy" | not searched | free | taken | — | medium | LOW |
| Limen | Greek harbor / Latin threshold | not searched | taken | taken | — | — | DEAD |
| Portus | Latin harbor | not searched | taken | taken (org Portus) | SUSE Portus registry UI (general knowledge) | — | DEAD |
| Loggia | Italian merchants' arcade | not searched | taken | taken | — | — | DEAD |
| Duka | Swahili shop | not searched | taken | taken | — | — | DEAD |
| Chandlery | ship-supplier's shop | not searched | free | taken | — | 3 syl | LOW |
| Wares | English goods | not searched | taken | taken | — | — | DEAD |
| Feira | Portuguese fair/market | not searched | free | taken | — | — | LOW |
| Winkel | Dutch shop | not searched | free | taken | — | — | LOW |
| statiohq / statio-commerce / mansiohq | compounds | not searched | all free | not checked | — | — | fallback handles |

Sources for the table rows (direct checks): npm registry URLs of the form https://registry.npmjs.org/NAME (HTTP 200/404 as listed); GitHub profile URLs of the form https://github.com/LOGIN for every "taken" cell (e.g. [github.com/Mazon](https://github.com/Mazon), [github.com/Tagar](https://github.com/Tagar), [github.com/torg](https://github.com/torg), [github.com/stoa](https://github.com/stoa), [github.com/depot](https://github.com/depot), [github.com/quay](https://github.com/quay), [github.com/WayStation](https://github.com/WayStation), [github.com/dukkan](https://github.com/dukkan), [github.com/mercantry](https://github.com/mercantry)). Indirect-conflict sources: [Mercatus Technologies — CB Insights](https://www.cbinsights.com/company/mercatus-technologies); [Mercatus grocery e-commerce — GetApp](https://www.getapp.com/website-ecommerce-software/a/mercatus-digital-solutions-for-grocery/); [WayStation AI website service — SimilarLabs](https://similarlabs.com/es/p/ai-website-service); [WayStation MCP bundle](https://www.mcpbundles.com/bundles/waystation); [Sundry Clothing — CB Insights](https://www.cbinsights.com/company/sundry-clothing); [Provi about](https://www.provi.com/about-us); [Dukkan — CB Insights](https://www.cbinsights.com/company/dukkan); [Dukaan — Exploding Topics](https://explodingtopics.com/topic/dukaan); [Torg — Dealroom](https://app.dealroom.co/companies/torg_); [Stoa $2.4M — tech.eu](https://tech.eu/2026/07/06/stoa-secures-24m-for-cash-rewards-platform/); [Mercato — Seedtable](https://seedtable.com/companies/mercato/changelog); [MERX — Datanyze](https://www.datanyze.com/companies/merx/24641882); [Fiscus — TrustRadius](https://www.trustradius.com/products/fiscus/details); [Emporia Energy](https://help.emporiaenergy.com/en/collections/8823791-about-emporia-energy); [Quay Australia — Dealroom](https://wa.dealroom.co/companies/quay_australia/team); [Pike Electric people directory](https://anymailfinder.com/directory/pike.com/people/ken-flechler); [Pike13 — Crunchbase](https://crunchbase.com/organization/pike13); [Wayfare Ventures](https://www.tryfundable.ai/investor/wayfare-ventures); [DeCA commissaries](https://corp.commissaries.com/node/8376); [Depot1.com trademarks — Justia](https://trademarks.justia.com/owners/depot1-com-inc-986332); [Pack House LLC trademarks — Justia](https://trademark.justia.com/owners/pack-house-llc-4488804); [Packhouse Technology — CB Insights](https://www.cbinsights.com/compare/kwik-lok-vs-packhouse-technology); [Tradepost.ai — SimilarLabs](https://similarlabs.com/p/tradepost-ai); [TradersPost](https://traderspost.io/about); [Fondaco SGR](https://ippjournal.com/company/fondaco-sgr); [Fondaco AB](https://www.formland.com/suppliers/supplier/fondaco-ab); [Kaupang — Wikipedia](https://en.wikipedia.org/wiki/Kaupang); [Stow storage — CB Insights](https://www.cbinsights.com/company/stow1/); [Hanut Sales Corp](https://www.seair.co.in/indian-trader/hanut-sales-corporation.aspx); [MISKAR (PL)](https://krs-pobierz.pl/miskar-i0001059270); [VICUS (BE)](https://busibee.be/en/0806481853-vicus); [Tessere mosaic](https://www.architectatwork.com/en/curated/brands/tessere-85727); [Matmonim podcast](https://podcasts.apple.com/podcast/id1543165926); [Pinta editor](https://www.clubic.com/telecharger-fiche443241-pinta.html); [Pinta company — CB Insights](https://www.cbinsights.com/company/pinta/alternatives-competitors); [Waypost](https://github.com/waypostadvisors); [TradehouseFlow](https://alternativeto.net/software/tradehouseflow/about); [Mercantile Stores — Wikipedia](https://en.wikipedia.org/wiki/Mercantile_Stores_Company,_Inc.); [Mercantile Bank IR](https://ir.mercbank.com/).

### Inferences
- The founder's instinct that "most plain English commerce nouns are taken" is confirmed and extends to the Latin/Greek market words (Mercatus, Emporia, Stoa, Mercat/Mercato, Merx, Torg, Dukkan) — the grocery/B2B-marketplace wave of 2015-2023 consumed them.
- Two-syllable coined compounds built on "-stead" are the only family where handles came back fully clean; that is a signal that the .coms are at worst parked rather than in active use, but it still needs RDAP confirmation.
- For Statio the handle collisions are hobby/library accounts, not a company, so the "HQ" pattern (statiohq on GitHub/npm/X, statio.com or statiohq.com on the web) is workable — the same pattern Toast (toasttab) and Square (squareup) used.

### Gaps
- No .com was proven free. All domain cells need the founder's RDAP run.
- USPTO live-mark status for Statio, Shopstead, Pundak, Waystead, Caupo, Mansio, Mercantry is unknown (search engine could not reach tmsearch.uspto.gov and the WebSearch budget ran out before the "NAME trademark" queries).
- The GitHub org `mercantry` (created 2026, repo "registry") could be an early-stage product; its website was not identified.
- Social handles (X, Instagram) were not spot-checked for any name (budget).

## Which names have hidden negative meanings in Spanish, Italian, Portuguese, Hebrew, Arabic or common slang?

### Takeaway
The clear language/slang failures are Pinta (Spanish slang and skin disease), Piska (Slavic childish/vulgar term), Mazon (one letter from Amazon), Machoz ("macho"), Vicus ("vicious"), Fiscus (the taxman in Dutch/German), Commissary (military/prison), and the two English-sound traps Caupo ("cow poo") and Waystead ("wasted"). Taberna means pub/tavern in Spanish and Portuguese; Pundak means "shoulder" in Indonesian (benign).

### Cited Findings
- "pundak" is the Indonesian/Malay word for shoulder; every result for "pundak.com" was an Indonesian health article about shoulder pain — [alodokter](https://www.alodokter.com/pundak-terasa-berat-inilah-penyebab-dan-cara-mengatasinya); [halodoc](https://www.halodoc.com/artikel/pundak-terasa-pegal-ini-penyebab-dan-cara-mengatasinya)
- A search for "mazon.com" returned exclusively Amazon.com pages, demonstrating search-engine and typo confusability with Amazon — [example result](https://heykidscomics.fandom.com/wiki/Amazon.com)
- "Taberna" is used as the name of Spanish and Portuguese tapas/tavern restaurants (Fairfield CT, Island Park NY, Lisbon) — [OpenTable](https://www.opentable.ae/r/a-taberna-island-park); [Bairro do Avillez Lisbon](https://www.bairrodoavillez.pt/en/taberna/)
- "Pinta" is the name of an open-source image editor and of a company tracked by CB Insights — [Clubic Pinta](https://www.clubic.com/telecharger-fiche443241-pinta.html); [CB Insights Pinta](https://www.cbinsights.com/company/pinta/alternatives-competitors)
- "Commissary" is dominated online by the US Defense Commissary Agency (military grocery benefit) — [DeCA](https://corp.commissaries.com/node/8376); [Goodfellow AFB](https://www.goodfellow.af.mil/Newsroom/Article-Display/Article/373247/discover-more-of-your-benefit-at-commissariescom/)
- "Mercatus" is best known as the Mercatus Center think tank (libertarian, George Mason University) — [Mercatus Center — Wikipedia](https://www.Wikipedia.com/wiki/Mercatus_Center)
- "Vicus" in Roman usage was "a small civilian settlement outside a Roman fort" — i.e. fort-adjacent, which brushes the founder's no-military rule — [Wiktionary vicus](https://en.wiktionary.com/wiki/vicus)
- "Fondaco" derives from the medieval storage-and-lodging buildings for foreign merchants in the maritime republics (the Fondaco dei Tedeschi in Venice is now a luxury mall) — [Fondaco SGR profile](https://ippjournal.com/company/fondaco-sgr)
- "Kaupang" was the first Viking-age town/marketplace in Norway (c. 800) — [Wikipedia Kaupang](https://en.wikipedia.org/wiki/Kaupang)
- "Statio" historically referred to stopping places on Roman roads for shelter and horse changes — [Wikipedia Statio (Roman)](https://en.wikipedia.org/wiki/Statio_(Roman)); "Mansio" was the official stopping place on a Roman road — [Wikipedia Mansio](https://en.wikipedia.org/wiki/Mansio)

### Inferences (general knowledge; not source-verified in this session)
- Pinta: in Spanish, "tener buena/mala pinta" = to look good/bad; "pinta" is also a tropical skin disease and, in Spain, a pint of beer.
- Piska: in Polish ("pisia"/"piska") and South-Slavic ("piška") childish or vulgar words for genitals/urination — a classic namecheck failure.
- Fiscus: "de fiscus" (NL) / "der Fiskus" (DE) = the tax authority; terrible connotation for merchants.
- Caupo: English ear hears "cow poo"; Waystead: "wasted"; Vicus: "vicious"/"viscous"; Machoz: "macho"; Shuka: "shook-a"; Quay: most Americans say "kway", not "key".
- Tagar: Hebrew "likro tigar" = to challenge/dispute; Indonesian "tagar" = hashtag (benign). Mansio: Italian "mansione" = job duty (benign); Spanish "mansión" = mansion (luxury drift). Lonja (ES) = fish exchange but also "slice". Torg (SE/NO) = town square (benign). Feira (PT) = fair but also the weekday suffix (segunda-feira) (benign). Hanut is the shared Semitic word for shop in both Hebrew and Arabic (benign, positive).

### Gaps
- No native-speaker review was done; the Slavic "piska" and Dutch "fiscus" readings are from general knowledge and should be confirmed before any of those names is reconsidered (neither is on the shortlist anyway).

## Which names pass both the "pizza shop test" and the "VC room test"?

### Takeaway
Both tests are passed cleanly only by Statio, Pundak, Shopstead and (with the EU-conflict caveat) Mansio; Mercantry passes both but has a GitHub squatter; Waystead and Caupo pass the pizza test but trip in the VC room on puns; Taberna, Tagar, Nundina and Townway pass the pizza test and fail or wobble in the VC room.

### Cited Findings
- No external naming-guide page (Lexicon, Igor, YC) could be fetched in this session (egress blocked), so the benchmark criteria used are the founder's own (short, category-defining, not a cutesy portmanteau) — see the Inferences below for the general-knowledge point about suffix domains.
- Mercatus Technologies' own positioning ("web and mobile storefronts, order fulfillment ... for regional and independent grocery retailers") shows how close an existing SaaS name can sit to this venture's pitch, which is why identical-name candidates were scored DEAD regardless of brand merit — [CB Insights Mercatus](https://www.cbinsights.com/company/mercatus-technologies)
- Dukkan's description ("digitizes small local businesses in the UAE and helps business owners sell products on an e-commerce marketplace") is essentially this venture's pitch — [CB Insights Dukkan](https://www.cbinsights.com/company/dukkan)
- WayStation is described as "an AI-powered platform designed to simplify website creation for agencies and developers by automating the process of building and deploying websites" — the same product category — [SimilarLabs](https://similarlabs.com/es/p/ai-website-service)

### Inferences
- General knowledge (not source-verified here): the founder's benchmarks all use one real or near-real word with no portmanteau, and two of them launched on suffix domains (Toast on toasttab.com, Square on squareup.com), so a NAMEhq.com / getNAME.com fallback is consistent with the register the founder wants.
- Pizza-shop test (owner says "my site is on ___" without embarrassment): PASS — Statio, Mansio, Taberna, Pundak, Shopstead, Waystead, Townway, Mercantry, Posthouse, Tagar. MARGINAL — Caupo (giggle), Nundina (obscure), Vicus (sounds hostile). FAIL — Machsan/Mischar/Miskar (unpronounceable), Comptoir/Entrepot (spelling), Kaupang (spelling).
- VC-room test (short, category-defining, non-cutesy): STRONG — Statio, Mercantry, Pundak. GOOD — Mansio, Shopstead. MARGINAL — Waystead ("wasted"), Caupo ("cow poo"), Taberna ("bar"), Tagar ("tagger"), Nundina. WEAK — Townway (municipal), Posthouse (hotel).
- Mercantile-outpost feel (explicit brief): STRONGEST — Pundak (inn), Statio (post station), Mansio (road inn), Posthouse, Caupo (shopkeeper), Waystead; GOOD — Shopstead, Mercantry, Taberna, Nundina (market day); WEAK — Townway.

### Gaps
- No user testing was possible; the pizza-shop and VC verdicts are analytic judgments against the founder's stated criteria, not survey data.

## For the top 3, what domain + handle + repo naming package is recommended?

### Takeaway
Top 3: (1) Statio — brand-first pick, contingent on RDAP showing statio.com free/for sale or statiohq.com free and a clean USPTO search; (2) Shopstead — availability-first pick, every direct handle check free; (3) Pundak — meaning-first pick with free npm, a dormant GitHub squatter, and no commerce conflict surfaced. Waystead is the alternate if the founder can live with the "wasted" joke.

### Cited Findings
- Statio: npm `statiohq` free (404) and `statio-commerce` free (404) while `statio` is taken (200) — [registry statiohq](https://registry.npmjs.org/statiohq); [registry statio-commerce](https://registry.npmjs.org/statio-commerce); [registry statio](https://registry.npmjs.org/statio). GitHub `statio` user exists — [github.com/statio](https://github.com/statio).
- Shopstead: npm free (404) — [registry shopstead](https://registry.npmjs.org/shopstead); GitHub search `shopstead in:login` and `shopstead in:login type:org` each returned 0 results; repo search `shopstead in:name` returned 0.
- Pundak: npm free (404) — [registry pundak](https://registry.npmjs.org/pundak); GitHub user `Pundak` exists with only a profile-config repo (created 2022-03-26) — [Pundak/Pundak](https://github.com/Pundak/Pundak).
- Mansio (runner-up): npm `mansio` and `mansiohq` free (404) — [registry mansio](https://registry.npmjs.org/mansio); [registry mansiohq](https://registry.npmjs.org/mansiohq); conflicting French startup — [CB Insights](https://www.cbinsights.com/company/mansio-1).

### Inferences (recommended packages; every domain line must be confirmed with the script above)
1. **Statio** — Domain: statio.com if RDAP returns 404 or a registrar lists it for sale; otherwise statiohq.com (primary) with statio.co / statio.app as redirects. Handles: @statiohq on X/Instagram/TikTok (the bare @statio should be assumed taken). GitHub org: `statiohq` (bare `statio` is a user). npm: scope `@statiohq/*` (e.g. `@statiohq/cli`, `@statiohq/sdk`), or unscoped `statio-commerce`. Product phrasing: "Statio — storefronts for local merchants". Trademark: file in classes 42 (SaaS) and 35 (e-commerce services); search for STATIO and STATIO-formatives in 9/35/42 first.
2. **Shopstead** — Domain: shopstead.com (verify), fallbacks shopstead.co, shopstead.app, getshopstead.com. Handles: @shopstead everywhere (no collisions surfaced). GitHub org: `shopstead`. npm: `shopstead` unscoped plus scope `@shopstead/*`. Trademark note: clear against Shopify's SHOP family (SHOP, SHOP PAY, SHOPIFY) — the "-stead" suffix and different commercial impression help, but get counsel's opinion before launch.
3. **Pundak** — Domain: pundak.com (verify; only Indonesian health content surfaced, no business), fallbacks pundakhq.com, pundak.co, pundak.app. Handles: @pundak if free, else @pundakhq. GitHub org: `pundakhq` (bare `pundak` is a dormant user; a GitHub "name squatting" release request is possible for inactive accounts but not guaranteed). npm: `pundak` unscoped (free) and scope `@pundak/*`. Trademark: search PUNDAK in 9/35/42; no hits surfaced in this session.
- Alternate: **Waystead** — identical package shape to Shopstead (all handles free); choose it only if the "wasted" pun is acceptable.
- If the founder prefers the Latin register but Statio's .com is unobtainable at a sane price, **Mansio** is the next Latin option: mansiohq.com / @mansiohq / npm `mansio` (free) — but obtain a US trademark opinion given the French Mansio (hospitality) and Mansio GmbH (DE).

### Gaps
- All three packages depend on RDAP confirmation of the .com and alt-TLDs, which could not be performed here.
- Social-handle availability (@statiohq, @shopstead, @pundak) is unverified.
- No price data for any premium/parked .com could be gathered.
