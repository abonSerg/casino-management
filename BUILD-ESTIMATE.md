# Build-Side Estimate — Cost, Effort, and Break-Even

**The missing half of build-vs-buy.** `FINDINGS.md` §4 documents what *buying* costs. This document
estimates what *building* costs, and models the crossover between them.

Date: 2026-07-31. Companion to `FINDINGS.md`, `research/01-white-label-landscape.md`,
`research/02-game-provider-integration.md`, `src-analysis/ADMIN-BUNDLE-ANALYSIS.md`.

---

## 0. How to read this document (please read this section)

Every cost figure carries a basis tag:

| Tag | Meaning |
|---|---|
| **[vendor-published]** | Published by the vendor or regulator on their own site. Solid. |
| **[cited third-party]** | Industry article, consultancy, comparison site, salary survey. Directionally useful, unaudited. |
| **[my estimate]** | Mine. Always accompanied by the assumption that generated it. |

`NEXT-STEPS.md` §7 warns that **commercial terms are the weakest evidence in this entire investigation** —
revenue shares, minimum guarantees and integration fees are confidential industry-wide, and only Digitain and
GR8 Tech publish real brackets. That caveat applies with full force to everything below, and doubly to the
break-even model in §5, which multiplies two weakly-evidenced percentages against each other.

**This is a decision-framing document, not a budget.** The ranges are wide on purpose. Narrowing them requires
three live quotes — a white-label vendor, an aggregator, and a licensing consultancy — and no amount of desk
research substitutes for those. Where I have converted an estimate into a number, the number's job is to show
which *direction* the decision falls and how sensitive that is to assumptions, not to be transcribed into a
financial model.

---

## 1. Three scope tiers, because these are three different projects

The single most common error in build-vs-buy is comparing a white-label quote against the cost of Tier A while
mentally picturing Tier B or C. The vendor is selling something close to Tier C. An MVP is Tier A. They differ
by roughly **6×** in effort.

### Tier A — Genuine MVP: one brand, one market, one aggregator

The smallest thing that can hold a gambling licence and take real money. Scope:

- Player identity, registration, login, sessions, password reset, 2FA
- **Wallet + ledger** — multi-currency, cash and bonus balances segregated, double-entry, integer money
- **Seamless-wallet callback API** — `user/info`, `user/balance`, `transaction/bet`, `transaction/win`,
  `transaction/rollback` per the contract in `research/02` §3.2
- One aggregator integration; game catalogue sync; game launch (DEMO + REAL); per-game/per-country geo-restriction
- Cashier: 2–3 PSPs, deposit, withdrawal request queue with manual approval
- KYC: one vendor integration plus a manual fallback; document state machine
- Responsible gambling: deposit / loss / wager limits and self-exclusion, daily/weekly/monthly
- **One** bonus type (deposit match) with wagering requirement and an Active→Claimed→Forfeit state machine
- Back office: ~20 screens — players, transactions, withdrawals, KYC queue, games, one bonus, basic reporting
- Reconciliation against the aggregator's transaction feed; GGR/NGR reporting good enough for a regulator

Explicitly **not** in Tier A: tournaments, loyalty/VIP, spin wheel, referrals, cashback, segmentation,
CMS/banners/email templates, live chat, gamification, affiliate anything, sportsbook, multi-tenancy, in-house
games.

### Tier B — Parity with the observed ARC surface

What the sandbox actually is, minus the modules disabled for this tenant. The evidence base for the scope is
`ADMIN-BUNDLE-ANALYSIS.md`: **~251 distinct API calls across 20 service namespaces, 94 admin routes, 268 Redux
action types, and a 34-module × RCUD permission matrix**. On the player side, `RETAIL-SOURCE-ANALYSIS.md` plus
**1,796 i18n keys** across 8 shipped locales.

Tier A plus: the full bonus catalogue (deposit, freespins, joining, cashback, loyalty levels, spin wheel,
referral, coupon/promo code, birthday, cumulative) with per-currency terms and per-game wagering-contribution
templates; tournaments with leaderboard and settlement; player segmentation and an advance-filter builder;
tags; notifications and email templates; CMS pages, banners, media gallery; 10+ live PSPs behind a provider
framework; 14 currencies and 11–12 languages with an in-product translation editor; the four reporting screens
plus a separate v3 analytics service; suspicious-activity/risk tooling with duplicate-account detection, device
and IP history; full RBAC UI; Player 360 with six tabs.

Still excluded, because they were disabled for this tenant and are separate products in their own right:
**sportsbook** (its own trading, odds and settlement domain — realistically a second platform, not a module),
**live chat**, disputes/ticketing, the in-admin AI chatbot, and the first-party PixiJS Crash game (which needs
its own RGS and its own RNG certification).

### Tier C — A multi-tenant white-label *product* you could resell

Tier B plus everything that turns a casino into a platform business: a tenancy model with per-tenant feature
flags that gate **whole page routes**, a numbered-layout theming system, tenant provisioning automation,
per-tenant content and translation catalogues, group-level administration across brands, tenant-scoped data
isolation and reporting, revenue-share accounting per tenant, a second aggregator behind a pluggable adapter,
an affiliate/agent module (which the ARC build conspicuously lacks — `FINDINGS.md` §5.1), SLA and support
tooling, and the security hardening that shared infrastructure demands.

Tier C is where the vendors live. It is also the tier where you stop being an operator and start being a
software company with a support organisation, which is a different business with different economics.

**Nobody should build Tier C to run one casino.**

---

## 2. Effort by component

Person-months (PM). One PM = one competent engineer for one month, inclusive of their own testing, code review
and the meetings that make those happen. Not inclusive of management, QA specialists, or design — those are
counted in the team shape in §3. All figures **[my estimate]**, with the basis stated per row.

Tier A and Tier B columns are **absolute** (Tier B is the total, not the increment). Tier C is the **increment
over Tier B**.

| # | Component | A | B | C (increment) | Basis for the estimate |
|---:|---|---:|---:|---:|---|
| 1 | Identity, auth, sessions, RBAC core | 3–5 | 8–12 | +4–7 | B covers a 34-module × RCUD matrix with per-table column sub-permissions, enforced server-side |
| 2 | **Wallet + ledger** | 6–9 | 11–17 | +4–8 | Double-entry, multi-currency, cash/bonus/coin segregation, manual adjustment with audit trail. Correctness-critical; assumes property-based tests and a reconciliation harness |
| 3 | **Seamless-wallet callback API** | 4–6 | 6–9 | +3–5 | 5 endpoints. Small surface, high difficulty: idempotency on `transaction_uuid`, RSA-SHA256 verification, round lifecycle, rollback, sub-100ms p99 under retry storms |
| 4 | Game catalogue, launch, geo-restriction | 3–5 | 9–14 | +6–10 | B adds merchandising (categories, featured, manual reorder), bidirectional country restriction, ~105 providers of catalogue hygiene |
| 5 | **Bonus + wagering engine** | 3–5 | 20–32 | +8–14 | A is one bonus type. B is 10 types × per-currency terms × per-game contribution × segment targeting × liability reporting. The deepest rabbit hole in the product |
| 6 | Payments / cashier / PSP framework | 4–6 | 14–22 | +6–10 | ~1.5–2.5 PM per PSP after the framework exists; B assumes ~10 live of the 39 configurable, plus separate deposit/withdraw provider roles |
| 7 | KYC, AML, responsible gambling | 4–6 | 9–14 | +3–6 | Vendor integration plus manual fallback; document state machine; 4 limit types × 3 periods; self-exclusion must be enforced in the bet path, not just the UI |
| 8 | **Reporting + reconciliation** | 4–6 | 18–28 | +8–14 | B covers 4 report screens, exports, KPI dashboards, and a separate v3 analytics service (heat map, session length, bet distribution, bonus liability/expiry). Provider reconciliation is continuous, not a report |
| 9 | Back-office CRUD + RBAC UI | 5–8 | 28–42 | +8–14 | ~94 routes. Genuinely mostly CRUD — see §2.1. Costed at roughly 0.3–0.45 PM per screen |
| 10 | CMS, banners, email templates, i18n management | 1–2 | 9–14 | +6–10 | B includes the in-product translation editor, which is a core white-label capability |
| 11 | Player front end | 7–10 | 22–34 | +10–18 | B is the full surface: lobby, cashier with 5 tabs, account, bonus pages, tournaments, RG centre, search, notifications, responsive |
| 12 | CRM / engagement — segments, tags, notifications, tournaments | 0 | 15–24 | +6–10 | The segment/advance-filter builder is a non-trivial query-construction problem, not a form |
| 13 | Risk and fraud | 1–2 | 6–10 | +2–4 | Suspicious-activity rules, duplicate-account detection, device fingerprint and IP history, real-time alerting |
| 14 | Platform, infra, SRE, security engineering | 4–7 | 15–24 | +8–14 | CI/CD, environments, observability, DR, secrets, hardening. Under-budgeted in almost every estimate of this kind |
| 15 | Multi-tenancy, theming, provisioning | 0 | 2–4 | +20–32 | B assumes branded-but-single-tenant. C is where the real cost lands |
| 16 | Compliance engineering — audit logs, retention, regulatory reporting | 2–3 | 7–11 | +4–7 | Immutable audit trail, 4-month-plus transaction retention, jurisdictional reporting formats |
| 17 | Affiliate / agent module | 0 | 0 | +8–13 | Absent from the ARC build entirely; required for a resellable product |
| 18 | Tenant billing and rev-share accounting | 0 | 0 | +5–9 | Tier C only |
| | **Total** | **51–80** | **199–311** | **+107–175** | |

**Tier C absolute total: ~306–486 PM.**

### 2.1 The central point: the ~95 admin screens are the cheap part

This is the finding that should change how the estimate is read, and it is measurable rather than asserted.

Re-extracting the react-router route table from the admin bundle (`path:"…"` literals in
`src-analysis/admin/index.js`) yields **95 real admin routes**, independently confirming the count in
`NEXT-STEPS.md` §1. Classifying every one of them by URL shape:

| Shape | Count | Examples |
|---|---:|---|
| List / index screen | 55 | `/players`, `/casino-games`, `/currencies`, `/withdraw-request` |
| Detail / view | 15 | `/player-details/:id`, `/casino-games/restrict-countries/:id` |
| Create / add form | 11 | `/bonus/create`, `/categories/addGames/:categoryId`, `/staff/add` |
| Edit form | 9 | `/bonus/edit`, `/cms/edit/:cmsPageId`, `/staff/edit/:id` |
| Drag-and-drop reorder | 5 | `/banner/reorder`, `/casino-games/reorder` |

**Every single route falls into one of five shapes.** There is no sixth category. The back office is a filtered
paginated list, a Formik/Yup form, a detail page, or a drag-to-reorder widget — repeated ninety-five times over
different schemas.

Components 9, 10 and 12 — the back office, the CMS, and the CRM screens, i.e. the overwhelming majority of
those routes and the visible bulk of the product — total **52–80 PM** at Tier B: roughly **26% of the Tier B
effort for ~70% of the screens**.

They are cheap precisely because of that structural uniformity. Once the table component, the form scaffolding,
the filter/date-range widget and the RBAC gate exist, each additional screen is a schema and a route. This is
exactly the work that scaffolding, code generation, and a competent AI-assisted workflow compress hardest — and
it is the work a vendor demo shows you, because it is the work that looks like a product. Judging a platform by
its back office is judging it by its cheapest component.

The cost concentrates in four places instead, totalling **55–86 PM at Tier B — about the same as the entire
back office, for four components rather than seventy screens**:

**The ledger and wallet (11–17 PM).** Not hard to write; hard to make correct and keep correct. Multi-currency
with a bonus balance that has different withdrawal rules from the cash balance, manual adjustments by staff
that must be attributable and reversible, and a balance that is read on the hot path by an external system
that will retry aggressively. Money must be integers (×100,000 per the aggregator contract), every mutation
must be an append-only ledger entry rather than a balance update, and the invariant that the ledger sums to the
balance must be continuously verified rather than assumed. Every hour of design here is repaid; every shortcut
compounds into a reconciliation break that someone finds three months later in a regulator's sample.

**The seamless-wallet callback endpoint (6–9 PM).** Five endpoints — the smallest component in the table by
surface area, and one of the most expensive per endpoint. It sits in the hot path of every spin from every
studio behind the aggregator. Hub88's documented retry policy is the design constraint: bet/win retried 3×
with a 1-second timeout, rollbacks retried **up to 500 times with exponential backoff**
([docs.hub88.io](https://docs.hub88.io/developer-docs/operator-api-reference/wallet-api.md)). The endpoint must
therefore be safely re-callable indefinitely, idempotent on `transaction_uuid`, correctly distinguish a
legitimate retry from a genuine duplicate with different details (`RS_ERROR_DUPLICATE_TRANSACTION`), link wins
to bets via `reference_transaction_uuid`, honour round lifecycle including multi-leg cascading wins, and do all
of that fast enough that the studio does not pause traffic to you. **Correctness here is literally money**: a
double-credit bug is a direct loss, a double-debit bug is a player complaint that becomes a regulatory one.

**The bonus and wagering engine (20–32 PM).** The largest single line at Tier B and the one most often
underestimated, because its complexity is combinatorial rather than additive. Per-currency terms × per-game
wagering contribution × segment targeting × a state machine with expiry × the interaction between bonus funds
and cash funds during play. The engine must decide, on every single bet, which balance is being spent, how much
wagering progress that bet contributes given the game's contribution percentage, and whether the bonus has now
cleared, expired, or been forfeited. Then **bonus liability** — the money you owe players in uncleared bonus
funds — has to be reportable at any instant, because it is a balance-sheet item and a regulator will ask.
`FINDINGS.md` §6 already flags this as the deepest rabbit hole; the estimate reflects that.

**Reporting and reconciliation (18–28 PM).** Under-appreciated because "reporting" sounds like SELECT
statements. It is two distinct problems. The first is that operational GGR and regulatory GGR are computed
differently and must be reconciled against each other continuously (`research/02` §3.5). The second is that
your transaction log and the aggregator's must agree, every day, and when they do not, someone must be able to
find out why at the level of a single `transaction_uuid`. Building the reconciliation harness is not optional
and it is not a report — it is a piece of infrastructure that runs forever.

---

## 3. Team shape and calendar duration to a licensable v1

### 3.1 What "v1" means here

A licensable v1 is Tier A that has passed a platform certification and is attached to a granted licence. That
is a stricter bar than "the software works", and it is the bar that determines when you can take a real bet.

### 3.2 Team

Tier A, sized to deliver 51–80 PM of engineering without heroics. **[my estimate]**

| Role | FTE | Notes |
|---|---:|---|
| Engineering lead / architect | 1 | Owns the ledger and wallet design personally. Not a manager who codes occasionally |
| Backend engineers | 3–4 | One of whom is effectively the money-domain owner |
| Frontend — player site | 1.5–2 | Next.js; the cashier and game-launch flows are the hard parts |
| Frontend — back office | 1 | CRUD velocity role; scaffolding pays off fastest here |
| QA / test automation | 1 | Not optional. The wallet needs adversarial testing, not click-through |
| DevOps / SRE | 0.5–1 | Environments, CI/CD, observability, and the certification evidence trail |
| Product / BA | 0.5–1 | Translating regulatory requirements into acceptance criteria |
| Compliance lead | 0.5 | Fractional early, full-time before launch. Drives the licence application in parallel |
| **Total** | **~9–11** | |

Engineering FTE is ~7–9 of that, so 51–80 PM lands at roughly **7–10 months of engineering calendar** at Tier A.

### 3.3 Sequencing — what actually parallelises

The dependency structure is tighter than it looks, because most of the product reads from the ledger.

**Months 0–2 — foundations (low parallelism).** Ledger and wallet data model, identity, the service skeleton,
CI/CD and environments. Almost everything downstream depends on the ledger's shape, so getting this wrong is
the most expensive available mistake and rushing it to unblock parallel work is a false economy. In parallel
and off the critical path: corporate structure, licence jurisdiction decision, and the licence application
started (see §4.1 — **start this in month 0, not when the software is ready**).

**Months 2–6 — the wide phase (high parallelism).** Once the ledger and its interfaces are stable, five streams
run genuinely concurrently: (a) the seamless-wallet callback plus aggregator integration and game launch;
(b) cashier, PSP framework and withdrawal workflow; (c) KYC, RG limits and the compliance surface; (d) the back
office; (e) the player front end. This is the phase where headcount converts to velocity most efficiently.

**Months 6–9 — convergence (low parallelism again).** Bonus engine wired through the wallet, reconciliation
against real aggregator data in a sandbox, reporting, security hardening, penetration test, and the evidence
package for certification. Adding engineers here does not speed it up.

**Months 7–12+ — certification and licence, mostly waiting.** The platform audit and the regulator's process
run on their own calendar (§4). Engineering during this window is remediation of audit findings plus whatever
the regulator asks for.

**Realistic calendar to a licensable Tier A v1: 10–16 months** from a standing start, and the back half of that
range is set by the licence, not by the code. Compare with the vendor claims in `FINDINGS.md` §4.2 — BetConstruct
1–3 weeks, GammaPlus 2 weeks, GR8 Tech 3–6 weeks. Those are technical-deployment timelines on someone else's
licence. They are not comparable to this number, and treating them as comparable is the single biggest framing
error in this decision.

**Tier B is not 10–16 months.** At 199–311 PM with a team scaled to ~15–18 engineers, it is **24–36 months**,
and that assumes the team scales cleanly, which teams do not.

---

## 4. Non-engineering costs

**Frequently the larger half of the bill, and always the larger half of the calendar.** An engineering-led
estimate that stops at headcount will understate the true cost of building by roughly a factor of two and will
understate time-to-revenue by a great deal more.

**Evidence quality warning specific to this section.** The session's web-search budget was exhausted before
current fee schedules could be independently re-verified. Everything below therefore comes from one of three
places, and each figure says which: (a) the primary research already captured in
`research/02-game-provider-integration.md` §5, which is cited and dated; (b) figures I could still fetch
directly; or (c) **[my estimate]** with the generating assumption stated. Where a figure was neither available
nor safely inferable I have written **not found** rather than filling the gap — those gaps are real and are
listed together in §4.8. **Do not treat §4 as a budget.** It is a structure showing which lines exist and
roughly how large they are relative to each other.

### 4.1 Gambling licence — the calendar item, not the cost item

The fees are not what hurt. The **duration** is, because it gates revenue while the burn continues (§6.1).

| Jurisdiction | Reported duration | Notes | Basis |
|---|---|---|---|
| **Curaçao (CGA, post-2023 reform)** | ~6 weeks *claimed* | Sub-licensing ended with the reform; historically the cheapest/fastest route. The 6-week figure is a single consultancy source and is an **advertised** timeline | **[cited third-party]** [turbomates.com](https://turbomates.com/blog/gaming-licenses-comparison-of-popular-jurisdictions/), via `research/02` §5.2 |
| **Anjouan (Comoros)** | Marketed as fast | No physical office or local staff required; remote management permitted. **Lower prestige** than Malta/UK for regulated markets | **[cited third-party]** [mygaminglicense.com](https://www.mygaminglicense.com/blog/the-best-license-between-curacao-and-anjouan-for-your-igaming-business), [altenar.com](https://altenar.com/blog/anjouan-licence-vs-curacao-licence-what-are-the-main-differences-and-which-one-should-you-choose/) |
| **Malta (MGA)** | **6–12 months** vetting for a full licence | Plus a 90-day temporary licence window during which the system and compliance audits must complete. Independent third-party systems audit required **before going live**. ~10-year term, quarterly compliance reporting thereafter | **[cited third-party]** [gamblingnerd.com](https://www.gamblingnerd.com/blog/malta-gaming-authority-explained/), [csbgroup.com](https://www.csbgroup.com/igaming-services-malta/online-gaming-licence-application/) |
| **Kahnawake** | **6-month trial licence** before a 5-year full licence | Requires a Client Provider Authorization plus a Key Person Licence. **All gaming servers must be physically hosted in Kahnawake territory** — a hard infrastructure constraint, not just a fee | **[cited third-party]** [tetraconsultants.com](https://www.tetraconsultants.com/jurisdictions/kahnawake-gaming-license/) |

**Current LOK fee schedule: not found.** The Curaçao regime changed materially with the *Landsverordening op de
kansspelen*, and I could not verify current application fees, annual fees, or the cost of the mandatory local
substance requirements against a regulator-published source this session. This matters more than a missing
number usually would, because under LOK the **local substance requirement — registered office, local directors,
local staff — typically costs more than the licence fee itself**, and it is recurring. Treat this as an open
item and price it from a licensing consultancy quote.

Two structural points that do not depend on the missing fees:

- **The advertised Curaçao timeline and the Malta timeline differ by roughly an order of magnitude**, and that
  difference — not the fee difference — is what should drive the jurisdiction choice for a first licence.
- **A cheap licence your counterparties reject is not cheap.** `research/01` §6 notes white-label vendors
  universally run operators under Curaçao master licences, so Curaçao is demonstrably acceptable to the supply
  chain. Anjouan's standing with major PSPs and tier-1 studios is materially weaker, and I was **not able to
  verify** the reports that some studios decline to serve Anjouan-only operators. Confirm acceptance with your
  intended aggregator and PSP *before* choosing on price.

**Modelled in §5 [my estimate]: €150k–330k over three years**, assuming a Curaçao-tier licence with local
substance, corporate services and renewals, and excluding gaming tax. Assumption: the recurring substance and
corporate-services cost dominates, at roughly €40k–110k per year. A Malta licence would be **materially higher
on both cost and calendar** and is not what the model assumes.

### 4.2 Platform certification

The standard is **GLI-19 "Standards for Interactive Gaming Systems"** — this is the online one.
**GLI-11 is the land-based gaming-device standard and is not the relevant standard for an online platform**
([GLI-11 v2.0](https://gaminglabs.com/wp-content/uploads/2018/09/GLI-11-v2-0-Standard-FINAL.pdf) vs
[GLI-19 v3.0](https://gaminglabs.com/wp-content/uploads/2020/07/GLI-19-Interactive-Gaming-Systems-v3.0.pdf)),
a distinction `research/02` §5.1 already draws and one worth holding onto, because the two are routinely
conflated in vendor marketing.

What you are certifying as an operator building your own platform is the **system**, not the game math — the
RNG lives on the studio's RGS and is their certification obligation (`research/02` §3.4). Your obligation is to
integrate only certified games and to demonstrate that your platform handles transactions, player accounts,
responsible-gambling controls and reporting correctly.

- **Independent lab testing duration: typically 4–12 weeks** **[cited third-party]**, per `research/02` §5.3.
  This runs on the lab's calendar, so it should start well before you want to launch.
- **Lab pricing is quote-only.** GLI, BMM Testlabs, eCOGRA, iTech Labs and Gaming Associates do not publish
  rate cards for platform certification — **not found**, and I do not think a defensible figure exists publicly.
- The one hard number in the whole certification area comes from Kahnawake's RNG requirement:
  **$8,000–$15,000 for initial RNG testing and $3,000–$5,000/year for recertification** **[cited third-party]**
  ([tetraconsultants.com](https://www.tetraconsultants.com/jurisdictions/kahnawake-gaming-license/)). That is
  *RNG* testing, a narrower scope than platform certification, so read it as a **lower bound on the order of
  magnitude**, not as the price of what you need.
- Certification is recurring, not one-off: `research/02` §4 cites "tens of thousands of EUR/year in
  certification overhead per jurisdiction" **[cited third-party]**.

**This line grows with every market you enter.** Certification is closer to (system × jurisdiction) than to a
single event, and US states are fully siloed — a certification in New Jersey buys you nothing in Michigan
(`research/02` §5.2).

**Modelled in §5 [my estimate]: €60k–180k over three years** for a single-jurisdiction platform certification
plus annual recertification. Assumption: an initial engagement in the low tens of thousands, plus recurring
annual fees, in a single jurisdiction. If you build your own Crash game (the ARC platform has one), add a
separate RNG and game-math certification on top — that is a different scope and a different bill.

### 4.3 ISO/IEC 27001

Increasingly a baseline expectation for B2B credibility and for PSP and aggregator due diligence rather than a
strict legal requirement for a B2C operator. Certification runs a **three-year cycle with periodic surveillance
and recertification audits** **[cited third-party]**
([isms.online](https://www.isms.online/sectors/iso-27001-for-the-gaming-industry/),
[gamingassociates.com](https://gamingassociates.com/iso-iec-27001-certification/)).

**Specific first-time certification cost for a 20–40 person company: not found** this session.

**Modelled in §5 [my estimate]: €45k–110k over three years.** Assumption: a first-year push (gap analysis,
consultancy, policy tooling, and the certification body's Stage 1 + Stage 2 audits) that is the bulk of it,
followed by smaller annual surveillance audits. The larger real cost is usually **internal time**, which is not
in this line and is partly absorbed by the platform/SRE component in §2.

### 4.4 PSP onboarding, fees, and rolling reserves

This is a gate, not just a cost (§6.4).

**Specific figures — rolling reserve percentages and hold periods, MCC 7995 transaction fees, per-PSP setup
fees, underwriting duration, and minimum volume commitments — are not found** this session, and I am not
willing to estimate them, because they vary enormously by jurisdiction, licence, product mix and the operator's
processing history, and a plausible-looking made-up number here would be actively harmful to a plan.

What the investigation *does* establish, and what you can plan around:

- **The ARC tenant has 10+ live payment methods and 39 configurable PSP integrations** (`FINDINGS.md` §1.7),
  spanning European, Indian, North American and crypto rails. That is the shape of a mature operator's payment
  stack, and it is evidence that **payment diversity is a core operational requirement rather than a nice to
  have** — you need redundancy because individual rails fail, get closed to gambling, or are market-specific.
- **Separate deposit and withdraw provider roles** exist as first-class concepts, so the integration count is
  higher than the method count suggests.
- `research/01` §3 cites **additional payment rails at roughly $1,000–$5,000 per market** as a commonly excluded
  add-on cost **[cited third-party]**, which is the closest published anchor available.
- **PSP fees are excluded from the §5 model entirely** — deliberately, because you pay them under both the
  build and the buy path, so they cancel. What does *not* cancel is the **onboarding risk** and the
  **rolling reserve as working capital**, neither of which is a line item you can budget as a fee.

The planning conclusion stands without the numbers: **apply to multiple PSPs in parallel, start early, and
treat a rolling reserve as locked-up working capital that scales with your success.**

### 4.5 Aggregator minimum guarantees

Covered as a risk in §6.3 and as a cost in §5. The figure — **€5k–25k per month** — is from `research/02` §4
and is flagged there as **very low confidence, a single unverified secondary source**. It is nonetheless the
single most schedule-sensitive commercial term in the plan, because it starts when you contract, not when you
launch, and a licence delay converts it directly into burn.

The §5 model applies it as a floor: you pay the greater of the revenue share and the minimum. At €50k monthly
GGR a 12% share is €6k, so **the minimum binds and you are paying above your revenue-share rate** — which is
exactly the regime a new operator launches into.

### 4.6 Engineering cost rates — the input §5 is most sensitive to

The §5 model converts person-months to money at **€8,000 (low) to €14,000 (high) per person-month, fully
loaded**. Both are **[my estimate]**; the corroborating salary-survey research was **not obtainable** this
session and this is the assumption in §5 most worth replacing with real local data.

The assumptions behind them:

- **€8,000/PM ≈ €96,000/year fully loaded** — a senior backend engineer in Central/Eastern Europe or a
  comparable nearshore market, inclusive of employer taxes, benefits, equipment, software and an allocation of
  management and recruiting overhead. Roughly a gross salary in the €60–70k range with a ~1.4× loading.
- **€14,000/PM ≈ €168,000/year fully loaded** — the same seniority in Western Europe or the UK, or an
  outsourcing agency's charge-out rate for a senior engineer in a lower-cost region. Agency rates converge with
  Western European employment costs, which is why one number covers both.

Two adjustments worth making before you use these:

- **An iGaming premium is plausible but unverified.** Malta's iGaming cluster competes hard for engineers with
  payments and gambling-domain experience, and money-handling experience commands a premium generally. I could
  **not source** a quantified figure, so no premium is applied — meaning the model may understate the cost of
  hiring *specifically* for the wallet role, which is the one role you least want to compromise on (§6.2).
- **The blend matters more than the rate.** These are senior-engineer rates applied uniformly across all
  person-months. A real team mixes seniorities, which lowers the average — but the components where cost
  concentrates (§2.1) are precisely the ones you cannot staff with juniors.

### 4.7 Hosting, tooling, and the ongoing compliance function

**Hosting and observability — modelled at €110k–400k over three years [my estimate].** Specific benchmark
figures were **not found**. Assumption: a multi-AZ setup with managed Postgres and read replicas, Redis, object
storage, CDN, plus observability and error tracking — starting around €3k/month at launch volumes and growing
with traffic. Two things distinguish this from a generic SaaS workload and push it toward the upper end:

- **Gambling sites are heavily DDoS-targeted**, so enterprise-grade WAF and DDoS mitigation is a requirement
  rather than an upgrade. This is a real line item and it is not cheap.
- **Transaction volume drives observability cost superlinearly.** Every spin is several wallet calls, each
  generating traces and logs, and log-volume-priced tooling gets expensive precisely when the business is
  working. The `round`-level detail you need for reconciliation (§2.1) is the same data that makes the bill
  large — and you must retain transaction records for at least four months
  ([Hub88](https://docs.hub88.io/developer-docs/operator-api-reference/wallet-api.md)).

**Ongoing compliance and operations staffing — modelled at €330k–750k over three years [my estimate]**, i.e.
€110k–250k/year. Assumption: an MLRO or Head of Compliance plus one to two analysts covering AML transaction
monitoring, KYC review, and responsible-gambling case handling, in a mid-cost jurisdiction. Specific MLRO salary
benchmarks and AML/KYC SaaS per-check pricing were **not found** this session.

This line deserves particular attention because **it is one of the few genuinely asymmetric costs in the whole
comparison.** Under a white label, much of this function sits with the vendor, who holds the licence and carries
the regulatory relationship. Build your own and you must staff it yourself, from before launch, permanently. It
does not scale down for a small operator — a regulator's expectations of your AML function are not proportional
to your GGR. **[my estimate]: €90k–240k over three years** additionally for external legal, statutory audit and
corporate services.

### 4.8 Summary, and the gaps

Three-year non-engineering totals as modelled in §5. **Every row is [my estimate]** with the assumption stated
in the subsection above; the ranges are wide because the evidence is thin.

| Line | 3-year low | 3-year high | Evidence quality |
|---|---:|---:|---|
| Licence, substance, corporate services (§4.1) | €150k | €330k | Durations cited; **current LOK fees not found** |
| Platform certification + recertification (§4.2) | €60k | €180k | Duration cited; **lab pricing quote-only, not found** |
| ISO 27001 (§4.3) | €45k | €110k | Cycle cited; **cost not found** |
| Hosting, DDoS protection, observability (§4.7) | €110k | €400k | **Not found** |
| Compliance and AML staffing (§4.7) | €330k | €750k | **Not found** |
| Legal, audit, corporate (§4.7) | €90k | €240k | **Not found** |
| **Total non-engineering** | **€785k** | **€2.01M** | |

For scale: against a Tier A engineering build of €400k–1.12M plus €1.34M–2.35M of ongoing engineering over
years two and three, **non-engineering costs are 31–37% of the three-year total** (€2.53M at the low bound,
€5.48M at the high). They are also the part that starts earliest
(the licence application, §6.1) and the part that cannot be compressed by working harder.

**The open items, consolidated.** These are what a next session or a consultancy conversation should close, in
priority order, because they carry the most weight in §5:

1. **Engineering cost rates** for your actual hiring market (§4.6) — the single largest input to the model.
2. ~~Current Curaçao LOK fees and real observed grant durations~~ — **closed in §4.9.2.**
3. ~~PSP rolling reserve terms~~ — **closed in §4.9.6**; whether *you* can get approved remains a live gate.
4. **Aggregator minimum guarantee and ramp-up relief** (§4.5) — the term most likely to bite during a licence delay.
   Still open, and `research/03` §8 confirms nothing public exists for Fundist either.
5. ~~Platform certification quote~~ — **§4.9.4 establishes that no lab publishes pricing at all**, so this can
   only be closed by requesting three quotes directly.

> **See §4.9 below.** Dedicated research after this section was written closed most of these gaps and produced
> **two corrections that change the tax picture materially** — the Curaçao 2% e-zone rate has been dead since
> 2020, and Malta's Type 1 gaming tax triples to 15% on 1 October 2026.

Note what is *not* on that list: the engineering estimate in §2. As §5.4 shows, the commercial terms move the
answer roughly six times more than the build estimate does.

---

## 4.9 Addendum — figures obtained after §4.1–§4.8 were written

§4 was written with an exhausted search budget and 11 explicit *not found* markers. Dedicated research
subsequently closed most of them. **This addendum supersedes the corresponding rows above where they conflict.**
The §5 model was *not* re-run against these figures — see the closing note for which direction they move it.

Confidence tags: **[REG]** regulator- or law-published · **[PRO]** law firm / Big-4 · **[V]** vendor-published ·
**[3P]** cited third party, usually a licensing consultancy · **[NF]** still not found.

### 4.9.1 Two corrections that change the model, not just the numbers

**1. The "2% Curaçao tax" that nearly every broker still advertises has been dead since 2020.** The e-zone
regime was abolished on 1 January 2020, accelerated from its planned 2026 sunset under EU Code of Conduct and
OECD pressure; grandfathering ended 31 December 2022 **[PRO]**
([Chambers](https://chambers.com/articles/significant-changes-to-the-cura%C3%A7ao-profit-tax-regime-2)). The
live rate is **15% on profit up to XCG 500,000 and 22% above**, under a territorial system that has taxed only
domestic-source income since 2020 **[PRO]**
([Mondaq, 8 Oct 2025](https://www.mondaq.com/tax/1623986/corporate-tax-comparative-guide)). Curaçao levies **no
GGR tax**. Any model built on 2% understates Curaçao's tax cost by roughly an order of magnitude. The
territorial treatment is the real planning lever, and it needs local tax counsel rather than a broker page.

**2. Malta's gaming tax on Type 1 rises from 5% to 15% on 1 October 2026.** Legal Notices 84 and 86 of 2026,
published 1 April 2026, consolidate the gaming tax and device levy: **15% for Type 1** (casino/RNG), 10% for
Types 2–4 **[REG]**
([MGA](https://www.mga.org.mt/enhancements-to-maltas-vat-and-gaming-tax-frameworks-for-the-gaming-sector/)),
corroborated by [PwC Malta](https://www.pwc.com/mt/en/publications/gaming/changes-in-gaming-tax-applicable-from-october-2026.html)
and [GTG](https://gtg.com.mt/vat-recoverability-gaming-activities-malta-2026/). The same notices narrow the VAT
exemption, making most remote gaming a taxable supply and so **enabling input VAT recovery** on technology and
marketing spend — a partial offset. **Open question that materially changes the answer: whether the 15% base is
Malta-resident-player revenue or global GGR.** Sources did not resolve it; confirm with counsel before modelling
Malta.

⚠️ **Debunked:** several SEO pages apply that 15%/10% to the MGA *compliance contribution*. It is the **gaming
tax**; the compliance contribution tiers are unchanged. Separately, the widely-repeated claim that the B2C
annual licence fee is revenue-tiered at €25k/€30k/€35k is wrong — **that tiering is the B2B supply licence**.
B2C is flat €25,000 **[REG]**.

### 4.9.2 Licence fees and durations, now sourced

| | **Curaçao (LOK)** | **Anjouan** | **Malta (MGA)** | **Isle of Man** | **Kahnawake** |
|---|---|---|---|---|---|
| Application | **€4,592** (ANG 9,000) **[REG]** | €17,828 *or* €8,000 — two rival bodies | **€5,000** **[REG]** | **£5,250** **[REG]** | **$40,000** (CPA) **[REG]** |
| Annual | **€47,450** (ANG 96,000) **[REG]** | €17,828 *or* €8,000 | **€25,000 flat** + compliance contribution **[REG]** | **£36,750**, 5-yr term **[REG]** | **$20,000** **[REG]** |
| Min. share capital | none published | none | **€100,000** (Type 1) **[REG]** | **[NF]** | **[NF]** |
| GGR tax | **0%** | 0% claimed | 5% → **15% from 1 Oct 2026** | 1.5% / 0.5% / 0.1% of GGY, tiered **[REG]** | 0% claimed |
| Profit tax | **15% / 22%**, territorial | 0% claimed, unverifiable | 35% → **~5% effective** (6/7 refund) **[PRO]** | 0% | 0% claimed |
| **Advertised duration** | 4–6 weeks **[3P]** | 4–6 weeks **[3P]** | 12–16 weeks **[REG]** | 10–12 weeks **[REG]** | 8–16 weeks **[3P]** |
| **Realistic duration** | **6–12 months** | **[NF]** | **4–9 months** | 3–6 months | **[NF]** |
| Year-one all-in | **€75k–110k** | €18k–41k | **€150k–300k+** | $78k–165k | $60k+ |

Malta's **Type 1 compliance contribution** is a genuine tiered levy on top of the flat fee: 1.25% of the first
€3m of GGR, tapering to 0.40%, with a **€15,000 annual minimum and €375,000 cap** **[REG]**
([MGA guidance note](https://www.mga.org.mt/app/uploads/Guidance-Note-Licence-Fees-and-Taxation-1.pdf)).
Qualifying start-ups — group revenue under €10m in the preceding 36 months — get a **12-month moratorium**.

**Curaçao's advertised timeline is regulator-published and was demonstrably not being met.** The CGA's own
target is eight weeks per phase across two phases, extendable by four weeks each — a 16-week nominal, 24-week
statutory envelope **[REG]**
([CGA portal](https://portal.gamingcontrolcuracao.org/page/online-gaming-info)). Against that: the regulator
told iGB it was processing **~10 applications a week** having received **740 in H1 2024**, and the Curaçao
Chronicle reported **220 granted against 553 pending plus 279 unprocessed** **[3P press]**. At the stated
throughput an 832-application queue implies roughly **83 weeks of clearance** — arithmetic on cited figures, not
a published number, but it is the clearest available explanation for the 6–12 month real durations.

Note also that the licence term itself is **provisional for up to 6 months, extendable once, then a definitive
licence of indefinite duration** **[REG]** — not the "1 year" or "5 years renewable" that consultancy pages
claim.

### 4.9.3 Curaçao regime risk — new, and material

This did not appear in §4.1 at all, and it is the kind of thing that belongs in a build-vs-buy decision:

- **The entire CGA supervisory board resigned in September 2025**; oversight moved from Finance to Justice and
  the Prime Minister took control, with no publicly appointed replacement board **[3P press]**
  ([Gambling Insider](https://www.gamblinginsider.com/news/31575/prime-minister-takes-control-as-entire-curaao-gaming-authority-board-resigns)).
- **Finance Minister Javier Silvania, the architect of the LOK reform, resigned on 15 October 2025**, following
  a criminal complaint alleging fraud, embezzlement and money laundering **specifically concerning the issuance
  of provisional gambling licences during the LOK transition** **[3P press]**.
- **Wind-down on refusal is six weeks** from the rejection letter — cessation of the seal, no new players, no
  wagering, plus full accounting of player balances **[3P]**. Operators have been reported sitting in extended
  provisional periods with no confirmed end date.
- **All master licences and sub-licences are abolished** and the transitional "orange seal" expired permanently
  on 15 October 2025. Direct CGA application is now the only route.
- **Suppliers must register by 24 December 2026**, with the foreign-supplier portal expected to open October
  2026.

**Anjouan should probably be struck from consideration entirely.** On 8 January 2026 Comoros officials publicly
warned that licences from AOFA and affiliated bodies carry "no weight", and the **Central Bank of Comoros stated
they have "no physical or legal existence"** in the country — with 1,300+ licences reportedly already issued
**[3P press]** ([iGaming Today](https://www.igamingtoday.com/comoros-sounds-alarm-over-fake-anjouan-gambling-licenses/)).
Gambling is prohibited under the Comorian federal penal code, and two competing entities market rival fee
schedules differing by 2×. The strongest real-world datapoint cuts both ways: Relax Gaming (FDJ United) took an
Anjouan B2B licence in November 2025 and drew a *Le Monde* investigation in February 2026 calling the
jurisdiction's regulatory infrastructure "fictitious". A licence whose issuing authority the host state denies
exists is not a cheap licence; it is an unpriced counterparty risk sitting under your right to trade.

### 4.9.4 Certification — the structural finding matters more than the price

**Platform certification pricing is genuinely unpublished, and that is a confirmed finding rather than a search
failure.** GLI, BMM Testlabs, eCOGRA, iTech Labs, Gaming Associates, Quinel and SIQ all refuse to publish rate
cards; GLI offers only a contact form. Note SIQ is part of the GLI group, so it is not an independent price
comparison. The only public ranges — **US$40k–150k** — come from development agencies selling adjacent services
and should not enter a budget. **Get three quotes; treat any figure in the model as a placeholder.**

Three structural points that *are* solid and change how you plan:

1. **GLI-19 is two engagements, not one.** §1.6 of the standard splits it into **laboratory testing** and a
   separate **operational audit**, because system integrity depends on production procedures and network
   infrastructure. Budget and schedule both.
2. **GLI-19 v3.0 explicitly confirms §4.2's framing** — it contains a dedicated Player Account Management
   chapter and expressly excludes event wagering, which belongs to GLI-33. GLI-11 is, as §4.2 says, land-based;
   its own running header reads *"Standards for Gaming Devices in Casinos"*.
3. **"Provably fair" does not exempt an in-house Crash game from RNG certification.** UKGC, MGA and most
   European regulators require accredited-lab RNG certification regardless of algorithm. The right architecture
   is a provably-fair commit-reveal layer *on top of* a certified RNG. In-house RNG certification runs
   **US$5k–15k** for one engine in one jurisdiction, and certificates are **not portable** between
   jurisdictions. Lab engagement to certificate is 8–14 weeks, plus **4–12 weeks of regulator review** on top —
   budget ~3 months lab and up to 3 months regulator.

**MGA system audit:** performed by an MGA-**Approved Audit Service Provider** of your choosing, not by the MGA,
and due **within 60 days of go-live** — so the ASP must be engaged before launch. A System Review follows about
a year after issuance. Cost estimates conflict badly (€2,500–13,500, with one source citing €5,000–30,000 for a
larger technical audit); the approved-ASP list is portal-gated **[NF]**.

**New and worth knowing:** GLI released a **Gaming Security Framework** in October 2025 whose GLI-GSF-5 module
targets online gaming specifically. The framework documents are free, and it is accepted as an **alternative**
to ISO 27001 or NIST CSF, not an addition.

### 4.9.5 ISO 27001 for a 20–40 person company — §4.3's "cost not found" now closed

The most reliable structural finding, agreed independently by UK and US sources:

> **Surveillance audits run ≈33% of the initial audit fee per year; year-three recertification runs ≈80–100% of
> it.**

Audit fees for 20–40 staff, accredited body: **UK £6,250–£11,250**, **US $14,000–$16,000**. Add 10–15% for
auditor travel. UK fixed-fee implementation consultancy is genuinely purchasable at **£3,500–£7,500**, which has
no US equivalent in the sources — **if the entity has any UK footprint, certifying through a UKAS body is
roughly half the US cost**, driven by day rates (£1,250–1,500 vs $1,500–3,000). Compliance-automation platforms
(Vanta, Drata, Scytale, Sprinto) run **$12,000–28,000/yr** for a company this size on real contract data, and
**do not include the certification body's fee** — the platform can easily exceed the audit.

Duration is **6–9 months**, and the scheduling trap is real: **Stage 2 slots book 2–4 months out**. Book the
certification body before the ISMS is finished, not after.

One vendor is a demonstrable outlier and its numbers should be discarded: Drata quotes $15k–50k+ *per stage*
when every other source, including fellow automation vendor Vanta at $14–16k combined, puts the whole
engagement at or below Drata's floor.

Worth noting for scoping: ISO 27001 **partly substitutes for** a separate annual infosec audit. The MGA accepts
it or an equivalent security audit report for its Data Security Assessment, the UKGC accepts it as evidence for
RTS Section 4, and Greece requires it outright. Model it as one line, not two.

### 4.9.6 PSP onboarding — §4.4's gate, now quantified

**The rolling reserve is the most consistently sourced figure in all of this research: 5–10% held for 90–180
days is standard for a new gambling account**, and it is not a negotiating position. The cash-flow consequence
should be modelled explicitly as working capital, not as a cost:

> **$100k/month processing × 10% reserve × 180-day hold = $60,000 permanently frozen** — a rolling steady state,
> not a one-off.

Negotiable to ~7–8% after six clean months and ~5% after 12–24. **Crypto and open-banking rails typically carry
no reserve**, which is a genuine structural mitigation rather than a rate tweak. Settlement is T+2/3 for good
accounts and **T+7 is common for new ones**.

Card MDR for MCC 7995 runs **2.5–5%** against 1.5–2.5% for standard e-commerce; open banking is **under 1%**.
Chargeback fees are $25–100 per dispute. Scheme registration is a real pass-through cost and is rising: **Visa
VIRP $950/yr** plus a 10¢ + 10bps integrity fee, and **Mastercard's specialty merchant registration doubles to
$1,000 per merchant per year on 1 May 2026**, alongside a new **$50,000/yr high-risk acquirer licence fee** and
a $0.02 per-transaction fee.

**Two risks §4.4 flagged qualitatively and this research confirms:**

- **The no-history catch-22 is explicit.** Underwriters need processing history to assess chargeback risk; a new
  operator has none; applications are declined. There is no clean solution other than launching small, on a
  licence, with alternative rails alongside cards. Every gambling account goes through manual underwriting —
  there is no automated approval for MCC 7995.
- **Issuer-side declines are outside your control entirely.** Many issuing banks decline MCC 7995 regardless of
  whether the operator is licensed, and the UK has banned credit cards for gambling outright. **Model a deposit
  success rate well below e-commerce norms.** A platform can be built correctly and still fail here.

Caveat carried forward: almost all decline-reason commentary comes from high-risk PSPs and brokers who profit
from the problem sounding hard. The *mechanisms* are corroborated across unrelated providers and rate as
high-confidence; the *percentages* do not.

### 4.9.7 Compliance staffing and KYC unit costs — §4.7's "not found"

**The MGA forces the shape of the team.** Eight Key Functions for a B2C licence, all natural persons requiring a
Certificate of Approval. One person may hold several — **except that Internal Audit cannot be combined with
operations, compliance or risk**, a hard separation. CEO, Compliance Officer and MLRO must be in place at
licensing; the rest within six months. The **MLRO must be FIAU-registered and carries personal regulatory
liability** for filing suspicious transaction reports, which is priced into the salary.

| Role | Malta | Cyprus |
|---|---|---|
| Head of Compliance / MLRO | **€70,000 ± 10,000** | €60,000–96,000 |
| AML analyst | €30,000–35,000 | €34,000–58,000 |
| Customer support agent | €18,000–27,000 | €19,000–28,000 |

**Cyprus is not a meaningful saving at senior compliance level** — the saving is in junior tiers and support
(Georgia runs roughly €11,000–16,000 for support). No usable senior-compliance benchmark was found for Bulgaria,
Poland, Serbia, the Philippines, Costa Rica or Curaçao **[NF]**; do not extrapolate.

**Round-the-clock support has a hard floor:** three shifts means **3 FTE for bare single cover**, and realistic
staffing lands nearer 4–5 FTE once holiday, sickness and attrition load in. No credible published benchmark
exists for total compliance-plus-payments-plus-fraud headcount at a small operator **[NF]** — treat any such
figure as invented.

**KYC per-verification pricing is the best-published area in this whole research** **[V]**:

| Vendor | Base | AML screening |
|---|---|---|
| **Sumsub** | $1.35 (Basic, $149/mo min) | **$1.85 Compliance tier — screening included** |
| **Veriff** | $0.80–$1.89 by tier | **+$0.64/verification**, +$0.09 ongoing monitoring |
| **Shufti Pro** | $0.95, bundled | bundled |
| **iDenfy** | from €0.50 | not itemised |

The honest unit cost for a gambling operator is **document plus liveness plus AML screening — roughly $1.85–2.03
per verification**, not the headline base rate. And since most vendors bill only *successful* verifications,
**cost per acquired player is higher than cost per attempted check** by your KYC pass rate. Model that.

**AML transaction monitoring has a trap.** ComplyAdvantage publishes a self-serve tier at $119–353/month, but
that covers 100–1,500 *entities*. A live casino screening every player continuously lands in the enterprise
band, where real contract data shows a **median around $52,900/yr**. Do not model the cheap tier.

### 4.9.8 Net effect on the model

These figures **do not overturn §5's conclusion, and mostly reinforce it**:

- Licence costs land **within** §4.1's modelled €150k–330k over three years for a Curaçao-tier licence — €75k–110k
  year one falling to €55k–85k thereafter is roughly €185k–280k across three years, comfortably inside the range.
- **But the Curaçao tax correction is a genuine addition** the model does not carry: 15%/22% on profit rather
  than 2%, subject to the territorial treatment. Malta at 15% gaming tax from October 2026 is worse still.
- **Regime risk is a new argument, and it cuts against building on a cheap licence** rather than against
  building as such. A regulator whose board has resigned amid a criminal complaint about licence issuance, with
  a six-week wind-down on refusal, is a poor foundation for a platform you own outright — and it is exactly the
  risk a white-label vendor absorbs on your behalf, which strengthens §7.1.
- **PSP onboarding hardens from a risk into a gate.** The reserve is quantified working capital and the
  no-history catch-22 is explicit. This is the clearest evidence in the whole document for §7.1's point that the
  binding constraints are not engineering ones.

The §5 crossover of €0.9M–1.9M monthly GGR is unchanged, because it is driven by the rev-share spread rather
than by these fixed costs — which is itself the point §5.4 makes.

---

## 5. Break-even: the number the decision actually turns on

### 5.1 The modelling choice that dominates everything else

Most build-vs-buy arithmetic in this industry is wrong in the same way: it compares the white-label revenue
share against **zero**, concluding that building saves you the entire 15–30%. It does not.

**You pay for game content either way.** Under a white label, the vendor's 15–30% is normally all-in — platform,
game content, payment rails, and the sub-licence. If you build, you drop the platform fee but you still sign
your own aggregator contract at **5–15% of GGR** (`research/02` §4) and you still pay for your own licence and
compliance function out of pocket.

So the marginal saving from building is not the white-label rate. It is:

> **saving = (white-label rate) − (your own aggregator rate)**

At a white-label rate of 20% and an aggregator rate of 12%, that spread is **8 points, not 20**. This single
correction moves the break-even point by roughly 2.5×, and it is the reason the crossover sits far higher than
the "obviously we should build at scale" intuition suggests.

It also produces the sharpest result in this document: **if you negotiate a 15% white label and your own
aggregator quotes you 15%, building never pays back at any volume.** The spread, not the headline rate, is what
you are buying.

### 5.2 Assumptions

All **[my estimate]** unless tagged otherwise. Costs common to both paths — marketing, PSP transaction fees,
player support, game content at the same rate — are excluded, because they cancel.

| Input | Low | High | Basis |
|---|---:|---:|---|
| Tier A build effort | 50 PM | 80 PM | §2 |
| Fully-loaded cost per person-month | €8,000 | €14,000 | §4.6 — **[my estimate]**, the model's most sensitive input |
| Ongoing engineering team, years 2–3 | 7 FTE | 7 FTE | §3.2, reduced post-launch |
| White-label setup fee | €35k | €90k | **[vendor-published]** GR8 Tech |
| White-label revenue share | 15% | 30% | **[cited third-party]** market band, `research/01` |
| Own aggregator revenue share | 8% | 15% | **[cited third-party]** `research/02` §4, low confidence |
| Aggregator monthly minimum | €5k | €25k | **[cited third-party]** single unverified source, `research/02` §4 |
| Horizon | 36 months | | |

### 5.3 Three-year total cost by monthly GGR

Build column = 3-year fixed build cost (engineering + licence + certification + hosting + compliance staffing)
plus 36 months of aggregator content at 12%. White-label columns = setup fee plus 36 months of revenue share.

| Monthly GGR | Build (low) | Build (high) | WL @15% | WL @20% | WL @30% |
|---|---:|---:|---:|---:|---:|
| €50k | €2.75M | €6.38M | €0.31M | €0.41M | €0.63M |
| €250k | €3.61M | €6.56M | €1.39M | €1.85M | €2.79M |
| €1M | €6.85M | €9.80M | €5.43M | €7.25M | €10.89M |
| €5M | €24.13M | €27.08M | €27.04M | €36.05M | €54.09M |

**Reading it in plain language.** At €50k and €250k of monthly GGR the white label is cheaper by a factor of
two to nine, and the comparison is not close enough for any assumption to rescue it. At €1M it is a genuine
toss-up — a lean build beats a 20% white label by about €400k over three years, which is inside the error bars,
while an expensive build loses to it. At €5M building wins decisively under every assumption, saving somewhere
between €3M and €29M over three years.

### 5.4 The crossover point and its sensitivity

Monthly GGR at which the three-year totals are equal:

| White-label rate | Own aggregator rate | Crossover (lean build) | Crossover (expensive build) |
|---|---|---:|---:|
| 15% | 8% | €0.98M | €2.16M |
| 15% | 12% | €2.30M | €5.03M |
| 15% | 15% | **never** | **never** |
| 20% | 8% | €0.57M | €1.26M |
| **20%** | **12%** | **€0.86M** | **€1.89M** |
| 20% | 15% | €1.38M | €3.02M |
| 25% | 12% | €0.53M | €1.16M |
| 30% | 12% | €0.38M | €0.84M |

**The central estimate is a crossover at roughly €0.9M–1.9M of monthly GGR** — the bolded row, at a 20%
white-label rate against a 12% aggregator rate.

But look at the spread of the table rather than the bolded row. The crossover moves across a **more than
thirteen-fold range** — from €380k to €5M per month — driven entirely by two percentages that
`NEXT-STEPS.md` §7 identifies as the weakest evidence in the whole investigation. The build cost itself, which
is the number everyone argues about, only moves the answer by about 2.2×. **The commercial terms matter roughly
six times more than the engineering estimate.** That is the strongest argument in this document for spending
your next week getting three real quotes rather than refining a build plan.

### 5.5 Steady state — the cleaner way to see it

The three-year view mixes a one-off build cost with a recurring one. Once the platform exists, the build cost is
sunk and the question simplifies to: *what does it cost per year to own versus to rent?*

Annual cost of owning: €0.92M (lean) to €1.84M (expensive) — an ongoing engineering team of ~7, plus licence
renewals, recertification, hosting, and a compliance function — **plus** the aggregator's 12%.

| Monthly GGR | Rent @20%/yr | Own, lean/yr | Own, expensive/yr |
|---|---:|---:|---:|
| €250k | €0.60M | €1.28M | €2.20M |
| €500k | €1.20M | €1.64M | €2.56M |
| €1M | €2.40M | €2.36M | €3.28M |
| €2M | €4.80M | €3.80M | €4.72M |
| €5M | €12.00M | €8.12M | €9.04M |

Steady-state crossover: **€0.96M/month (lean) to €1.92M/month (expensive)** — reassuringly consistent with the
three-year figure, which tells us the answer is driven by run-rate economics rather than by how the initial
build is amortised. **The initial build cost does not determine whether building is right; it only determines
how long you wait to find out.**

### 5.6 The hybrid path costs more, and that is not an argument against it

Start on a white label, build in parallel, migrate later. Modelled at €1M monthly GGR, 20% white label, 12%
aggregator, over three years:

| Path | 3-year total |
|---|---:|
| Pure white label, never build | €7.25M |
| Pure build | €6.85M |
| Hybrid, cut over month 12 (exit fee €150k) | €8.01M |
| Hybrid, cut over month 18 (exit fee €150k) | €8.49M |
| Hybrid, cut over month 24 (exit fee €150k) | €8.97M |

**The hybrid is the most expensive option on a three-year view, under every cutover date.** That is arithmetic,
not opinion: during the overlap you pay the vendor's revenue share *and* the build cost simultaneously, then a
migration fee on top.

It is still frequently the right choice, because the three-year total is not what it optimises for. It buys
revenue from month one instead of month twelve, it de-risks the build against a licence that may not arrive,
and it lets you learn the operational business — chargebacks, bonus abuse, withdrawal fraud, what your players
actually play — before you have committed the architecture that has to handle all of it. Those are real
purchases and the premium above is their price.

One caveat that the table cannot price honestly: **the exit fee is modelled at €0–400k because nobody publishes
it.** `research/01` §3 identifies data-migration-on-exit as a cost "commonly excluded from base packages," and
`FINDINGS.md` §4 calls it "the lock-in that matters." See §6.5.

### 5.7 What this model deliberately does not include

- **Forgone revenue during the build.** The pure-build path earns nothing for 10–16 months. For a *greenfield*
  operator this barely matters, because a new brand ramps from zero either way. For an *existing* operator with
  players already depositing, twelve months of forgone contribution at €1M monthly GGR is roughly €9.6M — which
  exceeds the entire three-year build cost, and is the single strongest argument for the hybrid path.
- **The value of the asset you own at the end.** After three years the build path leaves you owning a platform;
  the rent path leaves you owning a contract. That asymmetry is real but I have not tried to price it, because
  platform valuations in this sector are not something I can source honestly.
- **Execution risk.** The model assumes the build succeeds on schedule. §6 explains why that is the assumption
  most likely to be wrong.

---

## 6. Honest risk list

Ordered by how likely each is to be the thing that actually goes wrong, not by severity.

### 6.1 The licensing calendar is the bottleneck, not the integration

This is the risk most consistently mispriced, because it is the one the white-label model exists to hide.

`FINDINGS.md` §4.2 makes the point already: vendors advertise 1–3 weeks to launch, and those numbers describe
*technical deployment on the vendor's licence*. Build your own and you inherit the regulator's calendar
instead. §4.1 below gives the researched durations; the structural point is that they are measured in **months
to quarters, not weeks**, and that they are almost entirely outside your control.

The failure mode is specific and expensive: the engineering finishes on time, and then the company burns its
run-rate for two or three quarters with a working platform, a signed aggregator contract accruing monthly
minimums, and no licence. **Start the licence application in month 0, before the first line of code.** The
application is not gated on having a finished platform, and treating it as a late-stage task is the most common
way this plan fails.

Second-order version of the same risk: the licence you can get quickly may not be the licence your counterparties
accept. See §4.1 on Anjouan.

### 6.2 The wallet is where correctness is money

Most software defects cost you a bug report. Wallet defects cost you money directly, and they are asymmetric:
a double-credit is an immediate unrecoverable loss, while a double-debit is a player complaint that becomes a
regulatory finding.

The specific hazards, all of them consequences of the contract in `research/02` §3.2:

- **Idempotency is not optional and not simple.** Retries are routine, not exceptional — rollbacks retry up to
  500 times. Returning the *original* result for a repeated `transaction_uuid` is the requirement; returning a
  fresh debit is a silent money leak that reconciliation may not surface for weeks.
- **Floating-point money.** Integer minor units (×100,000) are mandated by the contract, and a single `float`
  in a bonus-contribution calculation reintroduces rounding drift that compounds across millions of spins.
- **Concurrency on a single balance.** A player with two game sessions open produces genuinely concurrent
  debits. This needs real transactional isolation, and the naive read-modify-write is wrong under load in a way
  that testing at low volume will not reveal.
- **Bonus/cash interaction.** Deciding which balance a bet spends, and what that does to wagering progress, is
  the intersection of the two most complex components in the system. It is where the subtle bugs live.

Mitigations that are worth their cost: an append-only ledger with balance as a derived value; a continuously
running invariant check that the ledger sums to the balance; property-based and adversarial testing of the
callback endpoint specifically; and a reconciliation harness against the aggregator's transaction feed built
*before* launch rather than after the first discrepancy.

### 6.3 Aggregator minimum guarantees apply from month one

An aggregator contract typically carries a **monthly minimum guarantee** — you pay the greater of the revenue
share and the floor, whether or not you have any players. `research/02` §4 cites €5k–25k/month, flagged there
as "very low confidence — single unverified secondary mention," and that caveat stands.

The consequence for a build plan is structural rather than merely financial. You cannot integrate the
aggregator, test against it, and then start paying when you launch. Contracting starts the clock, so during a
6–12 month licence wait you may be paying a floor against zero revenue. At the €25k end of the cited range that
is €150k–300k of pure burn before your first real bet.

Two things materially change this and both are negotiable: **ramp-up periods with reduced or waived minimums
for the first 3–6 months** are described as a common concession (`research/02` §4), and a sandbox/certification
environment does not always trigger the commercial terms. Ask about both explicitly, early, in writing.

### 6.4 PSP onboarding can fail outright, and it is not a cost line — it is a gate

Every other risk here is a matter of degree. This one is binary: without payment processing you have no
business, no matter how good the platform is.

A new gambling operator with no processing history, applying under MCC 7995, is close to the worst risk profile
in card acquiring. The specifics are in §4.4, but the shape of the risk is: applications are declined rather
than priced, and the reasons are largely outside engineering's control — no processing history, a
newly-incorporated entity, a licence the acquirer's risk team does not rate, thin beneficial-ownership
disclosure, or simply an acquirer that has closed to new gambling merchants that quarter.

Three consequences for planning. **Rolling reserves are working capital, not a fee** — a percentage of turnover
held for months is money you cannot spend, and it scales with success. **Apply to more than one PSP in
parallel, early**, and treat single-PSP dependency as an availability risk as much as a commercial one. And
note the asymmetry that makes this a genuine argument for the white label: an established vendor's pre-approved
payment rails are one of the few parts of their offer that a new operator genuinely cannot replicate, at any
price, because the missing ingredient is a track record rather than money or engineering.

### 6.5 The hybrid path's exit fee is deliberately unpublished

Starting on a white label and migrating later is the sensible-sounding compromise, and it carries a cost that
the vendors have every incentive not to quote you.

`research/01` §3 lists data-migration fees on exit among the costs "commonly excluded from base packages."
`FINDINGS.md` §4 is blunter: this is "the lock-in that matters." No vendor in the fifteen surveyed publishes a
figure, which is why §5.6 models it across a €0–400k range rather than estimating a point value — the honest
answer is that this number is **not found**, and its absence is itself the finding.

What you are actually exposed to is broader than a fee:

- **The player database.** Under a white label the players may be contractually the *vendor's* — SoftGamings
  markets migration "while keeping the player database" as a differentiator (`research/01` §2), which strongly
  implies it is not the default elsewhere.
- **Historical transaction data**, which you need for regulatory retention and reconciliation, in a usable
  format rather than a PDF dump.
- **The licence itself.** Under a white label you operate on the vendor's sub-licence. Migrating means holding
  your own, so the §6.1 calendar risk applies to the migration too.
- **Player-facing disruption** — forced re-registration or re-KYC at cutover is an attrition event.

Negotiate the exit terms *in the entry contract*, when you have leverage. Specifically: a defined data-export
format and schedule, explicit ownership of the player relationship, and a capped migration fee. A vendor
unwilling to commit to those at signing has told you something useful.

### 6.6 The estimate itself

Three ways §2 and §3 could be wrong, in descending order of likelihood:

- **Scope creep from Tier A toward Tier B.** The pressure is relentless and mostly legitimate — marketing wants
  a tournament, the support team wants segmentation, a market needs another PSP. Tier A is 51–80 PM and Tier B
  is 199–311 PM; drifting from one to the other without re-planning is a 4× overrun that arrives one reasonable
  request at a time.
- **Regulatory rework.** Requirements discovered during certification or the licence review land late, in the
  low-parallelism phase, when they are most expensive.
- **The team does not scale as assumed.** §3.3's middle phase assumes five workstreams running cleanly in
  parallel off a stable ledger interface. If the ledger design is still moving in month 4, they do not.

Everything in §2 is **[my estimate]**. The ranges are roughly ±40% around the midpoint, which is normal for
estimates made from an API surface rather than a specification — and the honest way to read the numbers is as a
statement about *relative* effort between components, which I have reasonable confidence in, rather than
absolute duration, which I do not.

---

## 7. Recommendation

### 7.1 The recommendation

**Do not build. Start on a white label or light turnkey, and negotiate the exit terms on day one as though you
intend to leave — because if the business works, you will.**

This is not a hedge and it is not a permanent answer. It is what the evidence supports at the volumes any new
operator will actually see in its first two years.

The reasoning, in order of weight:

**The crossover is far above where a new operator starts.** §5.4 puts it at €0.9M–1.9M of monthly GGR at
central assumptions. A new brand does not reach that in year one, and most never reach it at all. Below €500k
monthly GGR the white label is cheaper by a factor of two or more under *every* combination of assumptions in
the sensitivity table — there is no set of inputs in the plausible range that reverses it. The decision at
that scale is not close, and treating it as close is itself an error.

**The saving from building is the spread, not the headline rate.** §5.1 is the single most important line in
this document. Building saves you the difference between the white-label rate and your own aggregator rate —
around 8 points, not 20 — and at a 15% white label against a 15% aggregator quote, it saves you nothing at any
volume, forever.

**The binding constraints are not engineering ones.** The licence calendar (§6.1) and PSP onboarding (§6.4) are
what actually determine whether you can trade, and both are things the white-label vendor has already solved by
having a track record you cannot buy. Engineering is the part of this problem you are most able to do and least
constrained by.

**You do not yet know what to build.** The ARC surface documented across this investigation is the accumulated
answer to years of operational questions — why there is a duplicate-account detector, why bonus terms are
per-currency, why geo-restriction works in both directions. Running someone else's platform for a year tells you
which of those you actually need. Building first means guessing, and the guesses get embedded in the ledger
schema, which is the most expensive place to be wrong.

### 7.2 The conditions under which I would say build

Specific and checkable, rather than "at scale":

1. **Sustained monthly GGR above ~€1M with a credible path to €2M+**, on your own traffic. Sustained, not a
   promotional spike.
2. **A white-label rate above 20% against an aggregator quote at or below 12%** — an 8-point-plus spread. Get
   both in writing before deciding; §5.4 shows this input moves the answer six times more than the build
   estimate does.
3. **Your own licence already held**, or an application already granted. This removes the largest single
   schedule risk and roughly halves the calendar in §3.3.
4. **A product reason, not just a cost reason.** In-house games, an unusual bonus mechanic, a sweepstakes or
   crypto model the vendors handle badly, a market their licence does not cover. Cost alone is a weak reason to
   build; capability the vendor cannot sell you is a strong one.
5. **An engineering organisation that already runs money-handling systems in production.** Not "can hire one."
   The wallet (§6.2) is not a place to build that muscle for the first time.

If **1, 2 and 3** all hold, build — via the hybrid path in §5.6, accepting its cost premium as the price of not
betting the company on a launch date. If **4** holds strongly, build even below the crossover, and be honest
that you are buying capability rather than saving money. If only **5** holds, that is an argument you are
*capable* of building, which is not an argument that you *should*.

### 7.3 What would change my mind

- **A real aggregator quote at 8% or below, or a white-label quote above 25%.** Either pushes the crossover
  toward €400–600k monthly GGR, which is reachable in year two and would make building a live question much
  earlier.
- **Evidence that the licence calendar is shorter than §4.1 suggests** — a granted licence in under three
  months would remove the dominant schedule risk and materially cheapen the build path.
- **A vendor refusing to commit to exit terms** (§6.5). That converts the white label from a cheap option into
  a strategic trap, and changes the calculus from cost to control.
- **A published exit/data-migration fee at the high end.** If migrating later costs €400k+ and the player
  database is contractually the vendor's, the "start cheap and migrate" path is worse than it looks and the
  choice becomes more genuinely binary at the outset.
- **A materially cheaper build than §2 assumes.** The back office is ~26% of Tier B effort and is the most
  compressible work in the estimate (§2.1). A team that can demonstrably deliver those 95 CRUD screens at a
  fraction of 0.3–0.45 PM each — through scaffolding, code generation, or AI-assisted development — moves the
  build cost toward the low bound. Note what this does *not* touch: the wallet, the callback endpoint, the
  bonus engine and reconciliation are 55–86 PM of Tier B and they do not compress, because their cost is
  design and verification rather than typing.

### 7.4 What to do next, concretely

The decision is currently blocked on commercial evidence, not analysis. In priority order:

1. **Get three quotes**: a white-label vendor (GR8 Tech and Digitain publish brackets, so they are the natural
   anchors), an aggregator (SoftGamings/Fundist, since that is what ARC runs, plus one comparator), and a
   licensing consultancy. These three numbers collapse most of the uncertainty in §5.
2. **Ask every white-label vendor the exit question in writing**, before signing: data-export format, player-
   database ownership, migration fee cap.
3. **Ask the aggregator about minimum guarantees and ramp-up relief** (§6.3) — specifically whether the floor
   applies during integration and certification.
4. **Start a PSP conversation early** (§6.4), because it may constrain the jurisdiction choice rather than
   follow from it.
5. Only then revisit this model. The engineering estimate is the part least worth refining right now.
