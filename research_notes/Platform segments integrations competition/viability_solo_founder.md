# Can a solo founder build a remote-first, done-for-you website + integrations + managed platform for US small businesses that supports a family?

Researcher note, 5 October 2026, US market. Serves the founder vision in `founder_vision.md` (solo founder, family to support, Astro 7 on Cloudflare Workers + D1 + Stripe + ShipStation + Mailchimp already in production, family auto parts store as first custom client, remote-first, $500 sites up to $40k-$100k custom builds, one reusable engine with an OAuth app menu).

Sourcing caveat. The egress proxy blocked every page I tried to open directly this session (bls.gov, census.gov, starterstory.com, indiehackers.com, wp-umbrella.com, kolonell.com, substack.com, workd.com, hostinger.com, casestudies.com, saashero.net, sevenfigureagency.com, kinsta.com, hingemarketing.com, ishouldbeyourwpguy.com, redlib). Every figure below therefore comes from WebSearch result summaries of those pages (all 25 searches used) or from the sibling notes in this folder and in `research_notes/Local storefront platform strategy/` (which were researched in an earlier session with more fetch access). Figures that could only be seen in a search summary are marked "(via search summary)". Nothing in this note is from memory alone; where no source exists the item is in Gaps.

Cross-references used (not repeated here): `business_model_and_market.md` Q1 (builder and white-label pricing), Q2 (restaurant ordering competitors), Q3 (agency/freelancer benchmarks and care-plan churn), Q6 (spec-work funnel), Q7 (productized service vs managed platform vs self-serve SaaS); `business_types_ranking.md` (first-wave segments); `no_website_discovery_pipeline.md` (Overture/FSQ prospect pipeline); the three `integrations_*.md` notes (OAuth and partner gating per connector).

---

## Key question 1: What income does the business have to replace, and what does that mean in MRR and client counts at each price tier?

### Takeaway
The 2025 US median household income is $87,460 (released September 2026), so "supports a family at the median" means roughly $109k/yr of gross profit after self-employment tax and health insurance, or about $9,100/mo of MRR-equivalent; a $100k take-home needs about $10,400/mo. At a $149/mo blended subscription that is 62-70 subscribed sites with no custom work, or 17-26 sites if the founder closes two $40k custom builds a year.

### Cited Findings
- Median US household income was $87,460 in 2025, up 2.6% from $85,210 in 2024 and the highest on record since 1967; median post-tax household income rose from $73,760 to $76,060; official poverty rate 10.2% (Census Bureau, Income and Poverty report released September 2026, via search summary of syndicated AP coverage) — [Fox 9 / AP](https://www.fox9.com/news/americans-household-income-higher-data)
- Freelance web designers charge $50-$100/hr and $500-$5,000 flat per project; a simple 4-5 page small-business site "takes a few hours to one weekend if photos and content are ready", 2-3 weeks part-time with custom photography and copy; a DIY multi-page site is 40-80 hours (via search summary) — [PhotoBiz](https://blog.photobiz.com/website-builder/how-fast-can-you-really-build-a-website); [Abbacus Technologies](https://www.abbacustechnologies.com/how-many-hours-to-build-a-website/)
- A 5-10 page small-business site is $1,500-$5,000 from a freelancer and $5,000-$15,000 from an agency; care plans $50-$500/mo; "$0 down, $99-$199/mo, 12-month minimum" is an established offer; recurring web contracts reportedly churn 20-35%/yr — sibling note `business_model_and_market.md` Q3, citing [redefineweb](https://redefineweb.com/services/pay-monthly-websites-design-smb/), [seahawkmedia](https://seahawkmedia.com/wordpress/wordpress-maintenance-packages/), [kolonell](https://kolonell.com/en/blog/retain-web-clients-reduce-churn-agency-2026)
- The founder's cost of goods per site is far below any white-label builder: Cloudflare Workers Paid is $5/mo per account, not per site, versus $5-$17/site/mo on Sitejet/Duda/Brizy — sibling note `business_model_and_market.md` Q1, citing [costbench](https://www.costbench.com/software/cloud-infrastructure/cloudflare-pages-workers/)
- Broadly (a Duda customer) "moved from giving most websites away for free to consistently charging $99 per month" (via search summary) — [casestudies.com: Broadly](https://www.casestudies.com/company/duda/case-study/broadly-boosts-mrr-29-with-duda)
- GoHighLevel agencies typically resell at $97-$297/mo per client; 20 clients at $197 is $3,940/mo (vendor-ecosystem blog, via search summary) — [webdew](https://www.webdew.com/blog/gohighlevel-saas-mode)

### Inferences (calculation; assumptions stated)
Assumptions: the founder is a sole proprietor or S-corp paying ~14% self-employment tax on net profit and buying family health insurance (~$9k/yr), so take-home = gross profit / 1.25. COGS per site (Cloudflare, Stripe fee on the subscription, email, domain) is under $10/mo and is ignored below; it would add 5-10% to client counts at the $49 tier and 2-4% at $149+.

| Target | Take-home | Gross profit needed | MRR-equivalent |
|---|---|---|---|
| Median household (2025) | $87,460 | ~$109,300 | ~$9,100/mo |
| $100k | $100,000 | ~$125,000 | ~$10,400/mo |

Subscribed sites needed with no custom work:

| Monthly price | For median income | For $100k |
|---|---|---|
| $49 (bare presence) | 186 | 213 |
| $99 (presence + GBP sync + email) | 93 | 106 |
| $149 (presence + 2-3 OAuth apps) | 62 | 70 |
| $199 (retail/booking variant) | 46 | 53 |
| $249 (commerce + ShipStation + Mailchimp) | 37 | 42 |
| $399 (food ordering / heavy integrations) | 23 | 27 |

Sites needed when custom builds carry part of the load (median target / $100k target):

| Custom work per year | Remaining MRR needed | Sites at $149 | Sites at $249 |
|---|---|---|---|
| 1 x $40k | $5,800 / $7,100 | 39 / 48 | 24 / 29 |
| 2 x $40k | $2,400 / $3,750 | 17 / 26 | 10 / 16 |
| 1 x $100k | $800 / $2,100 | 6 / 14 | 4 / 9 |

- Setup fees are deliberately excluded from the tables; a $500-$1,500 setup fee on each new site is cash that funds the first year but is not income the family can plan on. Treat it as a buffer, not as the model.
- The realistic shape is a mix: 40 x $99 + 25 x $199 + 10 x $349 = $12,400 MRR, which clears the $100k target without custom work; the same base plus one $40k build a year is ~$155k gross profit.

### Gaps
- Could not open the Census press release itself; the $87,460 figure is from AP coverage syndicated by Fox local stations. The number is consistent across all ten syndicated copies in the search results.
- No source gives the health-insurance and SE-tax gross-up for a specific family; 1.25x is my assumption and the founder should replace it with their own numbers.

---

## Key question 2: What do solo and indie website-as-a-service operators actually achieve (time to $5k and $10k MRR, client counts, churn, hours per client, pricing that worked)?

### Takeaway
Direct, dated milestone data for solo WaaS operators is thin because the operator pages were blocked and few publish it; what exists says $99-$297/mo is the price band that works, maintenance is 20-30 minutes per site per month at scale, churn on small-business subscriptions can be very low (one developer: 2 cancellations out of 45 subscriptions over 5 years, the only churn being businesses closing) while agency blogs claim 20-35%/yr, and solo productized operators cap themselves at 20-35 active clients when each client is high-touch.

### Cited Findings
- Solo WaaS evidence, local businesses: a developer "charging no downpayment and a recurring subscription" to small local businesses reported ~12 clients with "only about 4 edits in the past 12 months combined"; another reported 45 clients over 5 years with only two subscriptions discontinued, churn coming only from businesses closing, because switching providers (new developer, hosting, domain, DNS) is painful for the owner; construction, contracting and cleaning companies were ~60% of clients; some hosts charge $100-$300/yr (Indie Hackers thread, via search summary) — [Indie Hackers: two sales tips for freelance web devs](https://www.indiehackers.com/post/two-sales-tips-for-freelance-web-devs-dd204a2e56); [Indie Hackers: FLX Websites](https://www.indiehackers.com/product/flx-websites/how-i-built-and-scaled-a-profitable-web-design-business-selling-to-local-regional-smbs--Nr3T_zPMbik2W5pTQLA)
- Maintenance load: one WordPress maintenance operator with 30 clients reports an average of 20-30 minutes updating each site; plan tiers quoted at $99/mo (updates, monitoring, backups, reporting), $399/mo with 2 hours of changes, $799/mo with 5 hours (via search summary) — [ishouldbeyourwpguy](https://ishouldbeyourwpguy.com/how-much-do-you-need-to-spend-to-maintain-your-website/)
- Solo productized design: Designjoy (Brett Williams) reached $1.3M-$1.74M/yr with no employees, capped at 30-35 clients (other summaries say ~20 active at 4-6 hours/day), raised prices nine times from $449 to $5,995/mo, and found "$500 clients proved demanding and unreliable, $5,000 clients valued efficiency and quality" (via search summaries) — [Starter Story / Design Joy](https://www.starterstory.com/ideas/design-agency/success-stories); [Indie Hackers AMA](https://www.indiehackers.com/post/i-make-50k-m-running-my-solo-unlimited-design-service-ama-133d3bba9d); [onepage-research summary](https://onepage-research.sliplane.app/articles/youtube.com/yt-dXKzST0FE-A)
- A different agency-of-one passed $1.5M ARR (Indie Hackers post title; body not retrievable) — [Indie Hackers](https://www.indiehackers.com/post/broke-the-1-5m-arr-mark-as-an-agency-of-one-e26ae36fe7)
- Starter Story: a designer went from $200 websites to a $600k/yr studio doing $100k projects (title only; body blocked) — [Starter Story](https://www.starterstory.com/stories/i-went-from-designing-website-for-200-to-starting-a-studio-that-makes-100k-projects)
- Churn benchmarks: recurring web contracts 20-35%/yr, under 15% with monthly reporting and annual contracts (agency blog, no survey) — [kolonell](https://kolonell.com/en/blog/retain-web-clients-reduce-churn-agency-2026); Recurly network data July 2026: 3.6%/mo all industries, 3.22%/mo SaaS — [enrichlabs](https://www.enrichlabs.ai/blog/churn-rate-complete-guide-2026); an indie SaaS reported 10-15% monthly churn requiring $2k of new MRR a month just to stay flat — [Indie Hackers](https://www.indiehackers.com/post/hitting-16k-mrr-after-years-of-failed-products-h5lM0xJmVrRZM0B23MT6)
- Why care-plan clients leave: "many clients treat a website like a one-time purchase rather than an ongoing investment, and if they don't see monthly benefits clearly, they might drop the service"; agencies "complete a contract and then the relationship fades" (via search summary) — [rocket.net](https://rocket.net/blog/4-ways-to-reduce-churn-in-your-wordpress-agency/); [yomotherboard](https://yomotherboard.com/question/what-causes-high-client-churn-rates-for-web-development-agencies/); [gowp](https://gowp.com/minimize-your-churn-rate-with-these-7-ideas/)
- Platform-scale comparators: Duda customers report 100+ sites/month at one agency, 1,000 sites producing $100k MRR for a hospitality software company (build time cut from 80 days to 14), Quantifi Media 100+ recurring client sites and 328% revenue growth in 3 years, Moovs selling 300 websites and doubling ARR in 7 months (vendor case studies, via search summary) — [Duda success stories](https://duda.co/success-stories); [casestudies.com: Quantifi](https://www.casestudies.com/company/duda/case-study/quantifi-media-drives-328-revenue-growth-with-duda); [casestudies.com: Moovs](https://www.casestudies.com/company/duda/case-study/moovs-doubles-arr-in-7-months-with-duda)
- Operational ceiling: "the scrappiest agency structure breaks somewhere between 8 and 15 clients, at which point defined roles and documented systems become non-negotiable" (agency consultant, via search summary) — [SaaS Hero](https://www.saashero.net/strategy/common-growth-agency-mistakes/)

### Inferences
- Two churn regimes exist and the founder's model sits in the low one if it is done right: "website as plumbing" (hosting + domain + DNS + integrations owned by the provider) churns only when the business dies, while "care plan on a site the client could move" churns 20-35%/yr. Integrations that the owner connected via OAuth inside the founder's dashboard (Stripe, Mailchimp, ShipStation, GBP) are the plumbing that makes leaving painful. This is the strongest argument for the app-menu design in the vision.
- Hours per client per month, for a templated engine with no code changes per client: plan on 20-30 minutes of maintenance plus 0-1 content edit requests, i.e. 0.5-1 hour; 80 sites is 27-40 hours/month of maintenance before any sales or builds. That is the founder's second-largest time sink after sales; it must be automated (one engine, one deploy, per-tenant config only) or the model caps at ~60-80 sites.
- Time to $5k and $10k MRR, modelled rather than observed (no operator published dated milestones that I could retrieve): with 15%/yr churn, 3 net-new sites a month at $149 ARPU reaches ~$2.6k MRR at month 6 and ~$5.0k at month 12; 4 a month reaches $5.1k at month 9 and $6.7k at month 12; 5 a month at $179 ARPU reaches $10k at month 12. So $5k MRR in 9-12 months and $10k MRR in 12-18 months is the plausible band for a solo founder who also sells, and the custom-build line (one $40k build = $3.3k/mo equivalent) is what makes year one survivable.
- Designjoy's "$500 clients were demanding and unreliable" is a warning about the $500 one-off tier: it is a lead magnet and case-study generator, not a profit centre, and it should be priced to convert into a subscription ($0-$500 down, $99+/mo) rather than sold as a finished project.

### Gaps
- No retrievable source gave a solo WaaS operator's dated path to $5k or $10k MRR; the FLX Websites and Starter Story bodies were blocked. The milestone timings above are a model, not an observation.
- No survey of hours-per-client for template-driven (non-WordPress) subscription sites; the 20-30 minute figure is one WordPress operator.
- No independent (non-vendor) churn data for Duda, Sitejet or GoHighLevel agency sub-accounts.

---

## Key question 3: Demand signal and churn driver: small-business formation and survival (SBA, BLS, Census BFS)

### Takeaway
The pool is large and refilling fast (36.2M US small businesses; 491k-579k new business applications every month in 2026, ~145k/month "high-propensity"), but half of new establishments are gone within five years and roughly one-fifth within a year, so a client base weighted toward new businesses will lose 10-20% a year to closures regardless of service quality.

### Cited Findings
- 36.2 million US small businesses, ~46% of private-sector employment, 1.1 million new establishments opened and 1.2 million net new jobs created in the most recent year; California 4.34M, Texas 3.52M, Florida 3.49M (SBA Office of Advocacy 2025 profiles, via search summary) — [SBA Advocacy](https://advocacy.sba.gov/2025/10/28/advocacy-releases-2025-small-business-profiles-for-major-metropolitan-areas/); [SBA Advocacy report](https://advocacy.sba.gov/?p=30239)
- Business applications: 496,443 in February 2026 (down 5.8% m/m); 491,941 in March 2026 with 144,952 high-propensity applications, 45% above March 2020; 578,926 in July 2026 (up 8.1% m/m) (Census BFS press releases, via search summary) — [Census BFS March 2026](https://www.census.gov/newsroom/press-releases/2026/business-formation-statistics-april8.html); [Census BFS July 2026](https://www.census.gov/newsroom/press-releases/2026/business-formation-statistics-jul9.html); [Social Explorer analysis](https://home.socialexplorer.com/post/business-formation-figures-show-smaller-than-expected-improvement-since-2020)
- Survival: 34.7% of establishments born in 2013 were still operating in 2023 (BLS TED, 2024); the 1995, 2000 and 2005 cohorts had 50%, 49% and 47% five-year survival; a recent summary puts five-year survival at 51.4% and says 48.6% of the March 2020 cohort was gone by March 2025; across sectors 66% survive 2 years and 44% survive 4 years; education/health services survive best (73%/55% at 2/4 years), information worst (63%/38%) (BLS BED, via search summaries) — [BLS TED 2024](https://www.bls.gov/opub/ted/2024/34-7-percent-of-business-establishments-born-in-2013-were-still-operating-in-2023.htm); [BLS MLR 2007](https://www.bls.gov/opub/mlr/2007/09/art1full.pdf); [startbusinessbystate summary](https://startbusinessbystate.com/?p=9502)
- Share without a website: Clutch (Aug 2025) 83% have a site, 17% offline, down from 36% offline in 2018; other surveys 27-29% offline; reasons in 2025: 34% "not relevant to my industry", 35% "too small to need one", 16% cost (cost was 26% in 2018) — [Clutch via BusinessWire](https://www.businesswire.com/news/home/20250821848152/en/Clutch-Report-No-Code-Tools-Fuel-Website-Growth-Yet-17-of-Small-Businesses-are-Still-Offline); [review42](https://resources.review42.com/what-percentage-of-small-businesses-have-a-website/); [workd](https://www.workd.com/insights/articles/small-businesses-without-websites/); [salesfuel](https://salesfuel.com/ad-agencies-targeting-small-business-owners-still-lack-websites/)
- One-third of the smallest businesses have no website (Small Business Majority, 2025) — [Small Business Majority](https://smallbusinessmajority.org/node/389880)
- Measured local pool (sibling note): Jacksonville, FL bbox in Overture places 2026-09-23.1 has 10,656 rows with a phone and no website, 8,144 of them open with confidence >= 0.5; a Chattanooga-sized bbox 2,665 — `no_website_discovery_pipeline.md` Q2, citing [Overture S3 listing](https://overturemaps-us-west-2.s3.amazonaws.com/?list-type=2&prefix=release/)

### Inferences
- Demand is not the constraint. One mid-size metro holds ~8,000 open, phone-listed, website-less businesses; the founder needs 60-100 of them. Even at a 1% close rate on personalised outreach that is 6,000-10,000 contacts, which is the metro's whole pool, so the funnel in `business_model_and_market.md` Q6 (100 personalised demos → 1-2 closes) has to be made cheaper per demo, or the founder needs warmer channels (referrals, chamber, in-person) than cold email.
- Closure churn: if the base is mostly established businesses (3+ years old) closure-driven churn is well under 10%/yr; if the founder chases new-formation lists (Florida daily filings) it is 15-20%/yr. The plan should mix both: new formations are easy to find and have no incumbent site, established businesses stay longer.
- The 2025 Clutch reasons ("not relevant", "too small") mean the pitch is not "a website is cheap now" but "customers already look you up; here is what they see", which is what the demo-first motion does.

### Gaps
- Could not open any BLS or Census page; all survival and BFS figures are from search summaries of those pages.
- No source splits website adoption by business age; the closure-churn split above is inferred from BLS survival curves.

---

## Key question 4: Failure modes (spec-work waste, support load, scope creep, too many verticals, building before selling, founder burnout)

### Takeaway
The documented ways this business dies are operational, not market-driven: unpaid custom work and scope creep that erode the retainer, maintenance requests that are awkward to refuse, chasing every vertical so nothing is repeatable, founder dependency with no SOPs, and the demo-first motion's >90% waste rate if demos are hand-built.

### Cited Findings
- "Growth agencies usually stall because of internal operational issues like weak niche focus, ignored SOPs, and founder dependency, not external market conditions"; "internal chaos such as missed deliverables, scope creep, and late launches usually hurts growth more than a weak niche or poor lead flow"; structure breaks at 8-15 clients (via search summary) — [SaaS Hero](https://www.saashero.net/strategy/common-growth-agency-mistakes/)
- Scope creep "occurs when you do more work than agreed upon with your client with no extra fees", leading to feeling "overworked and underpaid, eventually leading to burnout"; one freelancer "wasn't burned out because they had too much work, but because they had too many moving targets" (via search summary) — [Very Good Productized Guides: burnt out doing custom work](https://verygoodproductizedguides.substack.com/p/i-got-burnt-out-doing-custom-work); [the art of dealing with scope creep](https://verygoodproductizedguides.substack.com/p/the-art-of-dealing-with-scope-creep?r=1lrc0)
- "Maintenance costs can erode agency margins when clients request updates and adjustments under a retainer plan, and account managers find it awkward to say no, leading to hours of non-billable work every week that compounds across multiple clients" (via search summary) — [Lilach Bullock](https://www.lilachbullock.com/why-growing-agencies-struggle-with-website-maintenance-demands/)
- Agency owners cite "the fear of turning down revenue" and "letting go of the notion that their way was the only approach" as the burnout drivers; boundaries are the fix (Smart Agency podcast ep. 686, via search summary) — [Smart Agency Podcast](https://music.amazon.it/podcasts/56b5bd78-c746-4fec-b993-f100e4634023/episodes/3dba88da-880d-4e09-bbe1-53328effdeb5/smart-agency-podcast-the-1-digital-marketing-agency-podcast-for-social-media-seo-ppc-creative-agencies-avoiding-burnout-learning-how-to-let-go-with-brendan-chard-ep-686)
- Spec work: AIGA "strongly discourages" it; an 820-email cold test to small businesses got 3 replies (0.37%); a "free demo" offer converted better than any pricing discussion; expect >90% of unsolicited demos to be wasted — sibling note `business_model_and_market.md` Q6, citing [Indie Hackers 800 cold emails](https://www.indiehackers.com/post/800-cold-emails-later-heres-what-actually-moves-the-needle-and-what-s-a-complete-waste-of-time-1e67d7e295) and [boostlabs on spec work](https://boostlabs.com/blog/design-spec-work/)
- Restaurants are the hardest segment to win first: ordering is a POS problem and the incumbents price at $0 (Square Online, DoorDash Storefront) or $119-$499/mo plus per-order fees — sibling note `business_model_and_market.md` Q2
- Self-serve "cheaper Shopify" entrants repeatedly died or shrank (Volusion -74% stores since 2020; Tictail and Selz acquired and shut) — sibling note `business_model_and_market.md` Q7, citing [TechnologyChecker](https://www.technologychecker.io/blog/shopify-analytics-trends-insights)
- Designjoy: $500 clients "proved demanding and unreliable" (via search summary) — [onepage-research summary](https://onepage-research.sliplane.app/articles/youtube.com/yt-dXKzST0FE-A)

### Inferences
- The founder's specific trap is the one the vision already names: the $40k-$100k custom builds that fund the engine are also the scope-creep and burnout vector. A custom build sold as "inventory management for a retailer" has no natural edge; the mitigation is to sell it as fixed-scope engine modules (catalog import, inventory sync, dealer pricing) with change orders, and to accept at most one at a time.
- "Building before selling" is the quiet one: the vision lists four variants (food, retail, presence, custom) plus a self-serve AI builder. Only the presence/services variant is a complete product on the current stack; food needs ordering/POS integrations the engine does not have. Building the food variant before a paying restaurant exists is the classic mistake.
- Support load is the one that compounds silently. A site a client cannot break (no CMS they can wreck, content edits via a form that the founder approves) and a published "what's included" list are the cheapest insurance.

### Gaps
- No quantitative study of how much non-billable maintenance time typical agencies absorb per client; only qualitative agency-blog claims.
- No retrievable account of a solo WaaS operator who failed and wrote it up; survivorship bias in all operator stories.

---

## Key question 5: Success factors (referral loops, vertical focus, recurring-revenue mix, in-person trust early then remote, integrations as lock-in) and the productized-service playbook

### Takeaway
Specialisation pays measurably (vertical-focused agencies: 14.3% margin vs 8.6% and 2.5x the referrals), productised offers run 60-90% gross margin versus ~40% for custom, and the operators who stay solo do it by capping client count, forcing asynchronous requests, raising prices instead of adding staff, and letting recurring revenue be the base that projects build on.

### Cited Findings
- Specialised agencies focused on an industry vertical had an average profit margin of 14.3% versus 8.6% for generalists; specialised agencies receive 2.5x as many referrals; narrowing focus "created more growth opportunities than staying broad" (Hinge research via HubSpot; Kinsta agency survey; via search summaries) — [Hinge / HubSpot](https://hingemarketing.com/about-hinge/news-events/news/article/hubspot-publishes-hinge-article-practices-that-gain-client-referrals); [Kinsta: focus, retention and sustainable growth](https://kinsta.com/agency-growth-insights/focus-retention-and-sustainable-growth/)
- Referrals depend on "how well you know them and how well they know you", not network size (via search summary) — [Small But Mighty Agency](https://smallbutmightyagency.buzzsprout.com/1495015/episodes/16134277-is-your-agency-missing-out-on-referral-ready-relationships)
- Productised services: 60-90% gross profit on standardised offerings vs ~40% on custom; the method is document the process, template it, sell the template, automate delivery (via search summaries) — [Shiprocket on productised services](https://www.shiprocket.in/blog/productised-services/); [Visualize Value](https://visualizevalue.com/concepts/productization); [Jane Portman, Productized Consulting Guide](https://uibreakfast.gumroad.com/l/productized-basic)
- Brennan Dunn: price on the value of the outcome, not the feature list; "the more you can align the positioning of your product/service to the outcome the client wants the better chance you have of making the sale"; a higher price signals quality and makes most buyers curious rather than put off (via search summaries) — [Kalzumeus podcast 3](https://www.kalzumeus.com/2012/10/10/kalzumeus-podcast-3-growing-consulting-practices-with-brennan-dunn/); [Baremetrics founder chat](https://baremetrics.com/founder-chats/brennan-dunn); [freelancelift Q&A](https://www.freelancelift.com/?p=1090)
- Care plans "smooth seasonal volatility, lower dependency on constant sales activity, reduce churn, and create a base level of income that projects build on rather than replace" (via search summary) — [WP Umbrella](https://wp-umbrella.com/blog/why-wordpress-agencies-need-to-offer-care-plans-in-2026/); [The White Label Agency](https://thewhitelabelagency.com/wordpress-care-plans)
- Designjoy's solo rules: clients submit asynchronously via a Trello board, one request at a time, nine price rises from $449 to $5,995 to manage demand (via search summary) — [onepage-research summary](https://onepage-research.sliplane.app/articles/youtube.com/yt-dXKzST0FE-A)
- Switching cost as retention: owners stay because moving means "finding a new developer, potentially a new hosting platform, domain and DNS issues" (via search summary) — [Indie Hackers: two sales tips](https://www.indiehackers.com/post/two-sales-tips-for-freelance-web-devs-dd204a2e56)
- Personalised demo outreach: referencing the prospect's own site lifts replies 3-5x; Loom screen-shares reported 15% replies vs 5-8% for text; one agency claims ~40% higher close with a merchant-named demo — sibling note `business_model_and_market.md` Q6
- The segments where the presence variant is a complete product on day one are home/property services, independent auto repair and barbers/salons, where 40-57% have no site and the owner answers the phone — sibling note `business_types_ranking.md`

### Inferences
- Vertical focus for this founder means two verticals, not one: home/property services for the presence subscription volume (large pool, no incumbent, low support) and independent parts/hardware/retail for the commerce variant and custom-build revenue, anchored by the family auto parts store as the case study. Everything else waits.
- The "in-person first" tactic in the vision is consistent with the referral evidence: a few deeply-known local clients generate more referrals than a large cold list. The constraint is that in-person must produce artefacts (a photographed launch, a Google review, a testimonial video) that the remote funnel can reuse.
- Integrations as lock-in works only if the owner connects them inside the founder's dashboard (OAuth) and the founder's engine holds the state (orders, subscribers, GBP sync). A site that just links to the owner's own Square page has no lock-in.

### Gaps
- The 14.3%/8.6% margin and 2.5x referral statistics come from Hinge's professional-services research (sample and year not visible in the summary); treat as directional.
- No source quantifies referral rates for local-business web studios specifically.

---

## Key question 6: The AI-builder threat to the $500 tier, and how done-for-you + integrations + managed service defends

### Takeaway
AI builders (Wix AI, Squarespace Blueprint at $16/mo, GoDaddy Airo, Hostinger, Durable, B12) have made a generic brochure site effectively free in effort and $16-$42/mo in cost, and 73% of small-business owners say they plan to use AI for web design or content by end-2026, so the $500 one-off site has no durable margin; what AI builders do not do is the owner's work (photos, menu, hours, GBP, domain/DNS, Stripe, ShipStation, Mailchimp), carry the integration approvals, or answer the phone when something breaks, and that bundle is what the subscription sells.

### Cited Findings
- AI website builder market $2.69B (2025) to $3.24B (2026), $17.43B by 2035; 58% of small businesses use generative AI (23% in 2023); 73% of small-business owners plan to use AI for web design or content creation by end of 2026 (Hostinger compilation, via search summary; underlying surveys not visible) — [Hostinger AI website builder statistics 2026](https://www.hostinger.com/blog/ai-website-builder)
- "AI is rapidly changing the lower end of the market: simple brochure websites, basic landing pages, and template-driven builds are becoming easier and cheaper to produce with AI tools. Designers and agencies who rely only on production work will feel the pressure first"; mid-tier agencies squeezed from both sides (via search summaries) — [ColorWhistle](https://colorwhistle.com/diy-ai-website-agents-vs-agencies/); [AddWeb Solution on Webflow AI](https://www.addwebsolution.com/blog/ai-in-webflow-today-20-setups)
- Squarespace Blueprint: conversational setup, paid plans from $16/mo annual ($21-$25 monthly), 14-day trial, no free tier; GoDaddy Airo requires a domain purchase before generation; Wix AI ranked best overall in a 300-hour, 12-platform test; none has a permanent free plan (via search summaries) — [HostAdvice: Airo vs Squarespace](https://hostadvice.com/ai-app-builders/godaddy-airo-vs-squarespace/); [TechRadar: Wix AI vs Blueprint](https://www.techradar.com/pro/website-building/wix-ai-vs-squarespace-blueprint); [websitesetup.org](https://websitesetup.org/ai-website-builders/)
- 10Web from $10/mo; B12 (AI site + CRM + invoicing + bookings) from $42/mo; Durable and Framer in the same band — sibling note `business_model_and_market.md` Q1, citing [websitesetup.org](https://websitesetup.org/ai-website-builders/)
- Vertical SaaS now bundles free sites: Housecall Pro and Jobber ship free website builders wired to their booking/quote tools; Square Online Free and DoorDash Storefront are $0/mo — sibling notes `business_types_ranking.md` and `business_model_and_market.md` Q2, citing [PHCP Pros](https://www.phcppros.com/articles/9644-housecall-pro-offers-website-buider) and [HVAC Insider](https://hvacinsider.com/jobber-empowers-home-service-pros-to-attract-more-customers-and-grow-smarter-with-new-digital-marketing-tools/)
- Hold-outs are not price-blocked: only 16% of businesses without a site cite cost; 34% say not relevant, 35% too small — [Clutch via BusinessWire](https://www.businesswire.com/news/home/20250821848152/en/Clutch-Report-No-Code-Tools-Fuel-Website-Growth-Yet-17-of-Small-Businesses-are-Still-Offline)
- Partner gating is the moat AI builders cannot shortcut for a client: several connectors in the app menu require partner approval measured in weeks (see the three `integrations_*.md` notes for per-vendor gating and OAuth availability).

### Inferences
- The $500 tier should survive only as a priced-to-convert entry ("$500 down, then $99/mo", or "$0 down, $149/mo, 12 months") whose deliverable is explicitly the owner's done-for-you work: photos resized, hours and menu entered, GBP linked, domain moved, email set up. AI builders leave all of that to the owner, and the Clutch "too small / not relevant" owners are precisely the ones who will never do it themselves.
- The defensible line is "managed": the founder's dashboard holds the OAuth connections, the Stripe account link, the email list and the GBP sync; a site the owner never has to log into is worth $99-$249/mo to an owner who would not pay $16/mo for a builder they must operate.
- The vision's long-term self-serve AI builder competes directly with Wix AI and Blueprint on their home turf; it should be a retention feature for existing subscribers (owner adjusts copy by conversation) rather than an acquisition product.

### Gaps
- The 73% and 58% adoption figures are from a hosting-company compilation whose source surveys were not visible; mark as (unverified, Hostinger 2026).
- No data on what share of businesses that build an AI site keep it live after 12 months.

---

## Key question 7: The remote onboarding playbook (intake forms, screen-share setup, OAuth connectors, seven-day launch)

### Takeaway
The published playbook is intake form → proposal → build → finalise → test → launch → maintain, with a welcome email, a written process guide and a kickoff call; remote operators run intake online, collect assets before the build, and one Duda customer cut build time from 80 days to 14 by templating. A seven-day launch is consistent with "a few hours to one weekend" build times when content is ready, which makes asset collection, not design, the critical path.

### Cited Findings
- Seven-step process: intake form with the right questions, proposal, design, finalise, test, launch, monitor/maintain (Wix Partners / Wix Studio, via search summary) — [Wix Studio: optimise your web design process](https://wix.com/studio/blog/optimize-your-web-design-process)
- Intake form contents: name/contact, budget, SEO requirements, target audience, brand preferences, required features, timeline; "an online form is a better and more efficient way to present intake forms", enabling remote onboarding (via search summaries) — [Bonsai website intake form](https://www.hellobonsai.com/a/website-intake-form); [Tallyfy intake workflow](https://tallyfy.com/templates/forms/web-design-project-client-intake-form/); [Wizara questionnaire](https://www.wizara.com/form-templates/client-website-design-questionnaire)
- Onboarding actions: welcome email with next steps, a welcome package outlining process and expectations, a scheduled kickoff meeting (via search summary) — [CoordinateHQ](https://www.coordinatehq.com/solutions-articles/mastering-the-web-design-client-onboarding-process-a-comprehensive-guide)
- A simple 4-5 page site takes "a few hours to one weekend if your photos and content are ready ahead of time" (via search summary) — [PhotoBiz](https://blog.photobiz.com/website-builder/how-fast-can-you-really-build-a-website)
- Duda customer cut website build time from 80 days to 14 with templated builds; 1,000 sites, $100k MRR (vendor case study, via search summary) — [Duda success stories](https://duda.co/success-stories)
- Reddit practitioners recommend building a free AI draft, walking it through on a Google Meet and closing on the call — sibling note `business_model_and_market.md` Q6, citing [r/webdev getting first clients](https://redlib.hbubli.cc/r/webdev/comments/1j8kghd/getting_my_first_clients)
- OAuth availability, partner gating and webhook support for each connector in the app menu are documented per vendor in `integrations_payments_commerce_accounting.md`, `integrations_marketing_comms_booking.md` and `integrations_fulfillment_ordering_inventory.md`; Google Business Profile APIs can only manage listings the client owns or authorises — `no_website_discovery_pipeline.md` Q4, citing [Google Maps Platform Terms](https://cloud.google.com/maps-platform/terms)
- The founder's biggest content cost is photos and a services/price list, which the owner rarely has — sibling note `business_types_ranking.md` (day-one integrations section)

### Inferences (the playbook, assembled from the findings)
- Day 0 (sale): personalised demo already exists; close on a screen-share; collect card via Stripe Checkout for setup + first month; the demo URL becomes the staging site.
- Day 1: automated welcome email + one intake form (hours, services/prices, 10-20 photos upload, logo, domain registrar login or a "we'll register it" option, Google Business Profile owner email, which apps they use today: Square/Toast/Jobber/Mailchimp/etc.).
- Day 2-3: founder (later: AI) fills the engine's per-tenant config from the intake; no code per client.
- Day 4: 30-minute screen-share: owner clicks "Connect" on Stripe, Mailchimp, GBP, ShipStation in the dashboard (OAuth; founder never sees keys); domain/DNS moved to Cloudflare or CNAME'd.
- Day 5-6: review link, one round of edits via the dashboard's request form.
- Day 7: launch, GBP website field updated, review request sent, testimonial asked for at day 30.
- Integration approvals that take weeks (see integrations notes) must be applied for by the founder as a platform, once, not per client; where a connector is still pending the owner gets "API key + webhook" fallback with a guided screen.

### Gaps
- No published seven-day-launch case study with measured hours per stage from a solo operator; the timeline above is assembled from the generic playbook and build-time figures.
- No source on how many intake forms go unanswered (asset-collection drop-off), which in practice is the main schedule risk.

---

## Key question 8: Who is the competition?

### Takeaway
There are five layers, and the founder's product competes with a different one at each price point: DIY/AI builders at $0-$42/mo (Wix, Squarespace, GoDaddy Airo, Hostinger, Durable, B12), done-for-you incumbents that already sell to phone-book businesses at $99-$1,475/mo (Thryv, Hibu, Web.com, Yelp, GoDaddy design services), the thousands of local and GoHighLevel/Duda/Sitejet-powered agencies reselling sites at $97-$297/mo, vertical SaaS with free sites (Housecall Pro, Jobber, Square Online, Toast, Owner.com, Slice, DoorDash Storefront), and "the nephew" plus Facebook/Instagram as the website.

### Cited Findings
- DIY/AI builders: Squarespace from $16/mo annual; Wix AI best overall in a 12-platform test; GoDaddy Airo tied to a domain purchase; Hostinger, Durable, Framer, 10Web ($10/mo), B12 ($42/mo) — [HostAdvice](https://hostadvice.com/ai-app-builders/godaddy-airo-vs-wix/); [websitesetup.org](https://websitesetup.org/ai-website-builders/); sibling note `business_model_and_market.md` Q1 (full price table)
- Done-for-you incumbents: Thryv Starter $99/mo, Signature $399/mo, Custom; Ignite from $881/mo (CRM, email, invoices, booking) and Accelerate from $1,475/mo (adds ad budget and done-for-you SEO); "Professionally Designed Website" up to 15 pages written by their team; Hibu compared as a direct Thryv competitor, pricing not visible (via search summaries) — [Zeeg: Thryv pricing](https://zeeg.me/en/blog/post/thryv-pricing); [Thryv small business website development](https://www.thryv.com/reference/small-business-website-development); [Capterra: Thryv vs Hibu](https://www.capterra.co.uk/compare/156926/207701/thryv/vs/hibu)
- Agency platforms and their resellers: Duda $19-$149/mo agency plans plus $17/site; Sitejet (born from the Websitebutler agency, 2013) with white label, ticketing and client collaboration; GoHighLevel SaaS Mode on the $497/mo Agency Pro plan reselling at $97-$297/mo with self-signup and rebilling; Broadly charging $99/mo on Duda (via search summaries) — [Duda pricing via joinsecret](https://www.joinsecret.com/duda/pricing); [Sitejet for agencies](https://www.sitejet.io/en/website-builder-for-agencies); [webdew: GHL SaaS mode](https://www.webdew.com/blog/gohighlevel-saas-mode); [ghlexperts](https://ghlexperts.com/agency-saas/sell-gohighlevel-to-local-businesses)
- Vertical SaaS with free sites and ordering: Square Online Free, DoorDash Storefront $0; Toast online ordering ~$75/mo add-on + 3.5% + $0.15; Owner.com $249-$499/mo ($100M+ ARR); Slice 5-7%; Housecall Pro and Jobber free builders for trades — sibling notes `business_model_and_market.md` Q2 and Q7, `business_types_ranking.md`
- Freelancers/local agencies: $1,500-$5,000 freelancer, $5,000-$15,000 agency, care plans $50-$500/mo, "$0 down $99-$199/mo" WaaS — sibling note `business_model_and_market.md` Q3
- Substitutes: 21% of small businesses used social media instead of a website in 2018 (46% of those mainly Facebook); "my son is doing it" is a standing objection — sibling notes `business_types_ranking.md` and `business_model_and_market.md` Q6, citing [Clutch 2018](https://clutch.co/press-releases/small-businesses-use-social-media-instead-website-despite-risks) and [SitePoint](https://www.sitepoint.com/community/t/my-son-is-doing-it/55992)
- Automated demo-first prospecting already exists as a product category ("LocalBoss"-style agents that find no-website businesses and deploy a bespoke demo) — sibling note `business_model_and_market.md` Q6, citing [site-prospector skill listing](https://claudskills.com/skills/site-prospector/SKILL.md)

### Inferences
- The founder's nearest true competitor is not Wix; it is the GoHighLevel/Duda-powered local agency selling "$0 down, $149/mo" to the same phone-book businesses, and Thryv's $99-$399 tiers. Against them the founder's edges are cost of goods (Workers at $5/account vs $17/site or $497/mo), a real commerce stack (Stripe + ShipStation + Mailchimp in production) that GHL agencies do not have, and the ability to sell a $40k custom build, which neither Thryv nor a GHL reseller can.
- Against vertical SaaS the founder should not fight: for trades, the site should wire into Housecall Pro/Jobber; for restaurants, link to Square/Toast ordering rather than rebuild it. That is why food is the last variant to build, not the first.
- Thryv's $881-$1,475 tiers show the ceiling of what phone-book businesses will pay when the bundle includes CRM, booking and ads; the founder's $249-$399 "commerce + integrations" tier sits comfortably below it.

### Gaps
- Hibu, Web.com and GoDaddy design-service prices were not retrievable this session (unverified; sibling note Q1 has the builder tiers but not the done-for-you service prices).
- No market-share data for GoHighLevel-powered agencies or for how many of the 17-29% website-less businesses have already been pitched by Thryv/Hibu sales teams.

---

## Key question 9: Viability verdict with numbers

### Takeaway
Viable, with a specific shape: this works as a productized, vertical-focused subscription studio that reaches a median-income replacement (~$9.1k/mo gross-profit equivalent) at 60-70 subscribed sites, or at 17-26 sites plus two $40k custom builds a year, within 12-18 months; it does not work as a generic "$500 website" shop (AI builders own that), and it does not work as a self-serve SaaS first (Volusion/Tictail/Selz precedent, Wix/Squarespace incumbency).

### Cited Findings
- All numbers in the tables below trace to Q1 (income and price tiers), Q2 (churn, hours per client, solo caps), Q3 (pool and closures), Q4-Q6 (failure/success factors and AI pressure), Q8 (competitor prices). Primary anchors: median income $87,460 — [Fox 9 / AP](https://www.fox9.com/news/americans-household-income-higher-data); five-year establishment survival ~50% — [BLS TED 2024](https://www.bls.gov/opub/ted/2024/34-7-percent-of-business-establishments-born-in-2013-were-still-operating-in-2023.htm); 17-29% of small businesses have no site, cost cited by only 16% — [Clutch via BusinessWire](https://www.businesswire.com/news/home/20250821848152/en/Clutch-Report-No-Code-Tools-Fuel-Website-Growth-Yet-17-of-Small-Businesses-are-Still-Offline); low churn on managed small-business subscriptions (2 of 45 over 5 years) vs 20-35%/yr on care plans — [Indie Hackers](https://www.indiehackers.com/post/two-sales-tips-for-freelance-web-devs-dd204a2e56), [kolonell](https://kolonell.com/en/blog/retain-web-clients-reduce-churn-agency-2026); 20-30 min/site/month maintenance at 30 sites — [ishouldbeyourwpguy](https://ishouldbeyourwpguy.com/how-much-do-you-need-to-spend-to-maintain-your-website/); solo caps of 20-35 high-touch clients — [Designjoy summaries](https://onepage-research.sliplane.app/articles/youtube.com/yt-dXKzST0FE-A); productised margins 60-90% vs ~40% custom — [Shiprocket](https://www.shiprocket.in/blog/productised-services/); vertical agencies 14.3% vs 8.6% margin and 2.5x referrals — [Hinge / HubSpot](https://hingemarketing.com/about-hinge/news-events/news/article/hubspot-publishes-hinge-article-practices-that-gain-client-referrals)

### Inferences (the verdict)
Numbers (gross-profit equivalent, 1.25x gross-up for SE tax and family health insurance):

| Goal | MRR-equivalent | Sites at $99 | Sites at $149 | Sites at $249 | Sites at $149 + 1 x $40k build/yr | Sites at $149 + 2 x $40k/yr |
|---|---|---|---|---|---|---|
| Median household ($87,460) | $9,100 | 93 | 62 | 37 | 39 | 17 |
| $100k take-home | $10,400 | 106 | 70 | 42 | 48 | 26 |

Capacity check: 70 sites at 0.5-1 hr/month each is 35-70 hours/month of maintenance and edits, leaving ~90-125 hours/month for sales and builds in a 160-hour month; at 6-10 hours per onboarding, 4-5 new sites a month is feasible only if demos and configs are generated from the engine, not hand-built. Churn check: at 15%/yr the founder must replace 9-11 sites a year at 62-70 sites just to hold.

Why yes:
- The founder already owns the two things that cost competitors the most: a production commerce stack (most $149/mo WaaS sellers cannot do Stripe + ShipStation + Mailchimp) and a near-zero per-site COGS on Workers.
- The family auto parts store is a real $40k-class custom client and the case study for the retail variant; one such build a year cuts the subscription count needed by a third.
- Demand is abundant and findable (8,000+ open phone-listed no-website businesses in one mid-size metro) and the hold-outs are effort-blocked, not price-blocked, which is exactly what done-for-you solves.
- Managed subscriptions to small businesses churn mostly on closure, so a base of established businesses compounds.

Why it can still fail:
- The solo constraint is real: the model tops out around 100-150 sites per person even with automation, which is fine for one family's income but means "scale without the founder" requires either the self-serve layer or a first hire at ~80 sites.
- Year one is cash-poor: 3-4 net-new sites a month at $149 is only $5-7k MRR at month 12; without the custom build line or setup fees the founder is below median income for 12-18 months.
- Every operational failure mode in Q4 (scope creep on customs, unbounded support, too many verticals, hand-built demos) is a time leak, and time is the only input the solo founder has.

Verdict in one line: a solo founder with this stack can replace a median household income in 12-18 months with 60-70 managed subscriptions or ~20-40 subscriptions plus one or two custom builds a year, provided the first 12 months are spent on two verticals, one engine variant, and a demo-to-launch pipeline that costs under 8 founder-hours per client.

### Gaps
- The 12-18 month timing is modelled (Q2), not observed; no solo WaaS operator's dated milestones were retrievable.
- Whether the founder can actually close $40k builds beyond the family store is unproven; the plan below treats the second custom build as upside, not base case.

---

## Key question 10: A 12-month plan with monthly milestones

### Takeaway
Months 1-3 sell before building (family store as paid custom client, 5 local presence sites launched in person, engine config-only onboarding proven); months 4-6 turn the Overture pipeline into 100 personalised demos a month and reach ~$2.5k MRR; months 7-9 add the retail/commerce variant with OAuth apps and reach ~$5k MRR; months 10-12 hit ~$7-10k MRR with 45-56 sites, one custom build invoiced, and the first hire or self-serve decision made on data.

### Cited Findings
- Inputs: build times "a few hours to one weekend" with content ready — [PhotoBiz](https://blog.photobiz.com/website-builder/how-fast-can-you-really-build-a-website); demo-first funnel 100 demos → 1-2 closes and Loom 15% reply rates — sibling note `business_model_and_market.md` Q6; measured metro pool (Jacksonville 8,144 open phone/no-website rows) — sibling note `no_website_discovery_pipeline.md`; first-wave segments home/property services, auto repair, barbers/salons — sibling note `business_types_ranking.md`; partner-gated connectors need weeks of lead time — the three `integrations_*.md` notes; agency structure breaks at 8-15 clients without SOPs — [SaaS Hero](https://www.saashero.net/strategy/common-growth-agency-mistakes/); specialisation yields 2.5x referrals — [Hinge / HubSpot](https://hingemarketing.com/about-hinge/news-events/news/article/hubspot-publishes-hinge-article-practices-that-gain-client-referrals)

### Inferences (the plan; MRR figures assume $129-$179 blended ARPU and 15%/yr churn, see Q2 model)

| Month | Milestone | Sites / MRR target |
|---|---|---|
| 1 | Sign the family auto parts store as a paid, fixed-scope custom build (phase 1 catalog + inventory import, written change-order policy). Apply as a platform for every partner-gated connector in the app menu (Stripe Connect, Mailchimp, ShipStation, GBP, Square, the ones the integrations notes flag as weeks-long). Publish three packages with a "what's included / what's extra" page. | 1 custom; 0 sites |
| 2 | Launch 3 presence sites for nearby home-service or salon businesses, set up in person, each producing photos, a Google review and a testimonial. Measure founder hours per site; target under 10. | 3 / ~$400 |
| 3 | Engine onboarding is config-only (per-tenant D1 row + intake form, no code per client). Dashboard app menu live with Stripe, Mailchimp, GBP via OAuth; API-key + webhook fallback screen. 5 sites total. Family-store phase 1 delivered and invoiced. | 5 / ~$700 |
| 4 | Overture → D1 prospect pipeline for one metro, Google Place Details verify per row; generate 100 demo sites from engine templates (watermarked, noindex, 14-day expiry) and send 100 Loom/email pitches. Track reply, call and close rates. | 8 / ~$1.1k |
| 5 | Iterate pitch on the month-4 data; second vertical (independent retail/parts) demo template. First referral asks to the launched clients. | 12 / ~$1.6k |
| 6 | Retail/commerce variant (Stripe Checkout + ShipStation + Mailchimp) sold to 2 retailers at $199-$249. Review: hours per site, churn, close rate; kill any package or vertical that is not converting. | 17 / ~$2.5k |
| 7 | Monthly "your site this month" email to every subscriber (visits, calls, orders, GBP views) to pre-empt the "I don't see the value" churn reason. Second custom-build proposal out (parts/hardware retailer from the retail pipeline). | 23 / ~$3.5k |
| 8 | 200 demos/month throughput with semi-automated generation; one chamber/trade-association talk; case study page for the family store live. | 29 / ~$4.3k |
| 9 | $5k MRR. Decide, on measured hours, whether booking (salons, practices) or trades-software (Housecall Pro/Jobber) integration is next; build only the one with a paying client waiting. | 34 / ~$5.1k |
| 10 | Annual-prepay option (10% off) to lower churn and pull cash forward; second custom build underway or abandoned by data. | 40 / ~$6k |
| 11 | Support SLA and request form enforced (one request at a time, async); measure maintenance hours; if over 40 hrs/month, automate or price it. | 48 / ~$7.5k |
| 12 | 45-56 sites, $7-10k MRR, 1-2 custom builds invoiced in the year. Decide between first part-time hire (onboarding/support) and the owner-facing conversational editor as the "scale without the founder" path. | 56 / ~$10k (best case) |

- The food variant (DoorDash/POS) is deliberately absent from year one; it enters only when a restaurant client pays for it, per Q4's "building before selling".
- Cash: setup fees ($500-$1,500) on ~50 sites plus the family-store build are year-one working capital; the plan does not count them as income.

### Gaps
- Close rates per channel (in-person, Loom demo, referral) for this founder are unknown until month 4; the site counts are targets, not forecasts.
- Partner approval timelines for each connector are in the integrations notes but may slip; month 3's app-menu scope depends on them.

---

## Key question 11: Top five risks with mitigations

### Takeaway
The five risks that matter are time (solo capacity), cash in the first year, the custom-build scope trap, AI-builder price compression on the entry tier, and single-metro/vertical concentration; each has a concrete mitigation in the plan above.

### Cited Findings
- Capacity and structure break at 8-15 clients without SOPs; founder dependency is the top stall cause — [SaaS Hero](https://www.saashero.net/strategy/common-growth-agency-mistakes/); solo productised operators cap at 20-35 high-touch clients — [Designjoy summaries](https://onepage-research.sliplane.app/articles/youtube.com/yt-dXKzST0FE-A)
- Scope creep and non-billable maintenance erode retainers and cause burnout — [Very Good Productized Guides](https://verygoodproductizedguides.substack.com/p/i-got-burnt-out-doing-custom-work); [Lilach Bullock](https://www.lilachbullock.com/why-growing-agencies-struggle-with-website-maintenance-demands/)
- AI builders compress the brochure tier; 73% of owners plan to use AI for web design/content by end-2026 (unverified, Hostinger 2026) — [Hostinger](https://www.hostinger.com/blog/ai-website-builder)
- Half of establishments die within five years — [BLS TED 2024](https://www.bls.gov/opub/ted/2024/34-7-percent-of-business-establishments-born-in-2013-were-still-operating-in-2023.htm)
- Cold outreach to small businesses converts at 0.37-5% replies; demos are >90% waste — sibling note `business_model_and_market.md` Q6
- Churn on care plans is driven by clients not seeing monthly value — [rocket.net](https://rocket.net/blog/4-ways-to-reduce-churn-in-your-wordpress-agency/)

### Inferences (risks and mitigations)
1. Solo capacity ceiling (the founder's hours are the only input). Mitigation: config-only onboarding with a hard target of under 8-10 founder-hours per site; async one-request-at-a-time support; measure hours monthly; plan a part-time onboarding/support hire at ~80 sites or ship the conversational editor to subscribers.
2. Year-one cash gap (3-4 sites/month gives only $5-7k MRR at month 12). Mitigation: the family-store custom build in month 1 and a second custom proposal by month 7; setup fees and an annual-prepay option as working capital; keep any current income until MRR passes ~$5k.
3. Custom-build scope creep and burnout. Mitigation: fixed-scope engine modules with written change orders, one custom build in flight at a time, and every custom feature designed as a reusable engine module so the $40k funds the platform rather than one client.
4. AI builders and free vertical sites compressing the entry tier. Mitigation: never sell a $500 site as the product; sell "$500 down then $99/mo" or "$0 down $149/mo" with the owner's work done (photos, GBP, domain, OAuth apps) and the monthly value email; wire into Housecall Pro/Jobber/Square rather than compete with them; keep the self-serve AI builder as a subscriber feature, not an acquisition product.
5. Concentration and closure churn (one metro, new-formation prospects, two verticals). Mitigation: mix new-formation leads with established-business leads from Overture (confidence and age filters), expand to a second metro once the demo pipeline runs at 200/month, and track churn by cohort so closure churn is not mistaken for product churn.

### Gaps
- No source quantifies closure-driven churn specifically for website subscriptions by business age; the 15%/yr planning figure is derived from BLS survival curves and the one indie operator's report.
- Partner-program revocation risk (a vendor removing OAuth access or changing terms, which would break the app menu for every client at once) is not covered in any source found this session; it is a real platform risk and belongs in the integrations notes' follow-up.
