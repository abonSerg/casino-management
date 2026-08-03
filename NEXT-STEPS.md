# Continuation Plan — Casino Management Platform Investigation

**Status as of 2026-07-31.** This document records exactly what was covered, what was not, and what to do in
the next session. It is written to be picked up cold — a fresh session should be able to read this file plus
`FINDINGS.md` and continue without re-deriving anything.

---

## 0. Credentials and targets

All in `.env` at the repo root. No separate credentials were needed.

| Target | URL | Login |
|---|---|---|
| Back office (ARC) | `https://white-label-admin.gammaplus.io` | `ADMIN_USER_ID` / `ADMIN_PASSWORD` |
| Player site | `https://white-label-2.gammaplus.io` | `RETAIL_USER_ID` / `RETAIL_USER_PASSWORD` |
| Admin API | `https://white-label-adminapi.gammaplus.io/api/v2` | bearer token from admin login |
| Player API | `https://white-label-api.gammaplus.io/api/v1` | HttpOnly cookie |
| Chat app | `https://white-label-chat.gammaplus.io` | separate Vite SPA, **never analyzed** |

### Ground rules observed so far (keep these)

- **Read-only.** Forms were opened to read their fields; nothing was submitted. No deposits, no bets, no data
  changed in the vendor's sandbox.
- Game launch was exercised in **DEMO** mode only.
- No direct API calls were made with the admin session token — see §4 for why that needs a decision.

### Practical browser notes (will save time next run)

- The Chrome extension's `javascript_tool` is **blocked** on these hosts by the cookie/query-string guard. Use
  `read_page`, `get_page_text`, and `find` instead.
- The player site is heavy (PixiJS preload) and **wedges tabs regularly**. Symptoms: `screenshot` returns
  "Script injection timed out", `get_page_text` returns "Page still loading". Recovery: open a new tab with
  `tabs_create_mcp` and re-navigate. Tab IDs change; re-read `tabs_context_mcp` after any failure.
- `read_network_requests` only reliably captured OPTIONS preflights and the socket polling on the admin app.
  Do not rely on it for API enumeration — the bundles were far more productive.
- Waits of 8–10s per navigation are realistic on both apps.

---

## 1. Coverage: admin back office

**94 routes exist in the bundle. 28 were visited (~30%).**

### Visited and documented

`/` (Financial tab only) · `/casino-aggregators` · `/casino-providers` (p1/11) · `/casino-games` (p1) ·
`/categories` (p1/4) · `/application-configuration` · `/application-settings` · `/payment` ·
`/payment/add-provider` · `/bonus-types` · `/bonus` · `/bonus/create` (step 1) · `/bonus/:id/deposit` (view) ·
`/wagering-template` (list) · `/tournaments` · `/tournaments/create` (step 1) · `/casino-transactions` ·
`/currencies` · `/languages` · `/cms` · `/players` · `/player-details/65` (Overview + Limits tabs) · `/staff` ·
`/staff/add` · `/kyc-management` (Labels tab) · `/withdraw-request`

Two failed and are worth retrying: `/bet-settings` threw "SOMETHING WENT WRONG"; `/form-fields` silently
redirected to the dashboard (likely a permission gate or a sportsbook dependency).

### Not visited — grouped by what each would tell us

**High value for a build (do these first):**

| Route | Why it matters |
|---|---|
| `/languages-management` | The translation-editing UI. This is a *core* white-label capability — how an operator localizes without a deploy. Not seen at all. |
| `/advance-filter`, `/advance-filter/view` | The player segmentation builder. Segments drive bonus and campaign targeting; the data model here is non-trivial. |
| `/staff/details/:id`, `/staff/edit/:id` | The actual RBAC permission matrix UI (34 modules × R/C/U/D). We inferred the model from the bundle but never saw it rendered. |
| `/wagering-template/create` | The wagering contribution model per game. Only `NO_TEMPLATE` existed in the list, so the real shape is unseen. |
| `/countries` + `/countries/restricted-{games,providers}/:id` | The country-side of geo-restriction, and which markets are enabled. |
| `/categories/addGames/:categoryId` | How games are merchandised into categories at scale. |

**Medium value:**

`/game-report` · `/player-performance` · `/transaction-banking` (the three reporting screens) ·
`/email-templates` + create/edit · `/banner-management` · `/image-gallery` · `/user-tags` ·
`/players-bulk-update` · `/kyc-requests` · `/user-history/:userId` · `/review-management` · `/support` ·
`/notifications`, `/send-notification`, `/notification-details` · `/dashboard-new` (the v3 analytics UI) ·
the four `/*/reorder` screens · `/profile` · Dashboard **Casino** and **Bonuses** tabs · player-detail
**Reports / KYC / Referrals / Notes** tabs · `/bonus/edit` steps 2–3 (Currency, Content).

**Low value here (module disabled for this tenant):**

All `/chat/*` (channels, chat-rain, offensive-words, chat-settings) and all sportsbook routes
(`/sports`, `/matches`, `/markets`, `/match/:id`, `/sports-bets`, `/sports-transactions`, `/sports/countries`,
`/sports/leagues`). Documented from the bundle; visiting them will likely 404 or error.

---

## 2. Coverage: player site

### Visited

Home (logged out and in) · `/en/login` · `/en/casino` · `/en/account/wallet/transactions` ·
`/en/account/wallet/deposit` (+ CARD method) · `/play-casino/9/19/Mad Buffalo/true` (DEMO launch) ·
`/en/bonus/all`. Confirmed 404 on this tenant: `/vip`, `/promotions`. `/en/account/profile` rendered blank.

### Not visited — **`/register` is the notable one**

**`/register` was never opened, and it is the single most important unseen screen on the player side.** It is
where jurisdiction rules, KYC triggers, responsible-gambling defaults, currency selection and the configurable
form fields all converge — and the admin has a `/form-fields` screen for exactly this that we couldn't reach.
For a regulated product this is the highest-risk flow to get wrong, and we have no picture of it.

Also unseen: Withdraw tab · My Bonuses tab · Casino Transactions tab · `/responsible-gambling` ·
`/live-casino` and the LiveGame section · `/providers` · `/tournaments` · `/all` · search · notifications ·
the embedded chat widget · forgot-password · the 2FA enrolment flows (OTP and Google Authenticator, both
toggleable in admin) · any mobile/responsive viewport.

---

## 3. Not analyzed at all

1. **The chat application** — `white-label-chat.gammaplus.io` returns 200. It is a *third* app, a separate
   Vite-built SPA embedded by iframe, with its own socket.io client. Never downloaded or analyzed.
2. **The data model** — never reconstructed. See §5, this is the top priority.
3. **Build-side cost and effort** — never estimated. See §5, this is the other top priority.
4. ~~**SoftGamings / Fundist API specifics**~~ — **done**, see `research/03-fundist-integration.md`. Fundist
   publishes *no* public documentation (`docs.fundist.org` is NXDOMAIN; `fundist.org/en/Docs` is behind
   login+2FA; the "Fundist API OneWallet v128" spec is indexed but returns HTTP 410). The contract was
   reconstructed from four independent operator-side implementations in three languages that agree
   field-for-field, plus this repo's own retail payloads. Residual gaps are listed in §12 of that report as
   questions to put to SoftGamings.
5. **API response payloads** — no direct API calls made, so field-level response shapes are unknown. See §4.
6. **The sportsbook renderer** at `http://3.220.85.69:8008` — currently unreachable (times out).

---

## 4. Open decision for the user

**Direct API probing with the admin session token.** Calling `https://white-label-adminapi.gammaplus.io/api/v2/…`
directly with the logged-in bearer token would pin down request/response shapes precisely instead of inferring
them from the client bundle. That would make the data-model reconstruction (§5.1) authoritative rather than
inferred.

It is also a step up in intrusiveness from "loading pages a browser would load anyway" against a vendor's
sandbox. **Not done, awaiting an explicit go-ahead.** If granted, restrict to GET/read endpoints only — never
the POST/PUT/DELETE mutation endpoints, and never `internal/create-credentials` or `internal/update-credentials`
(third-party integration secrets).

---

## 5. Plan status — the 2026-08-01 run

Everything in the previous plan that did **not** need sandbox access is now done. Five documents were produced
in parallel; four agents were terminated by a session limit partway through follow-up work, so the residue is
recorded honestly below.

| Was | Status | Where it landed |
|---|---|---|
| §5.1 Reconstruct the domain data model | ✅ done | `DATA-MODEL.md` — 195 `[observed]` / 49 `[inferred]` markers, six state machines, and a §13 listing seven claims it **corrects** in earlier documents |
| §5.2 Build-side estimate | ✅ done | `BUILD-ESTIMATE.md` — three scope tiers, break-even, recommendation, plus a §4.9 addendum with licence/certification/PSP figures researched afterwards |
| §5.3 Research Fundist specifically | ✅ done | `research/03-fundist-integration.md` — and see the caveat below, it is not built on primary sources |
| §5.4 `/register` and the player-side gaps | ✅ done **statically** | `src-analysis/RETAIL-REGISTRATION-ANALYSIS.md` — recovered from the bundle, no browser session needed |
| §5.4 High-value *admin* screens | ❌ **not done** | Still needs either a browser session or more bundle mining — see §5.1 below |
| §5.5 Analyze the chat application | ✅ done | `src-analysis/CHAT-APP-ANALYSIS.md` |
| §5.6 Completeness sweep | ❌ not done | Lowest priority, unchanged |

Two documents were added that were not in the plan: **`CONTEXT.md`**, a glossary fixing the domain vocabulary
(GGR vs NGR, aggregator vs studio vs RGS vs PAM, seamless vs transfer wallet), and **`docs/adr/0001`**,
recording the read-only conduct decision and the deliberate choice not to probe the admin API — which is why
`DATA-MODEL.md` carries `[inferred]` markers at all.

### 5.1 What is left, in priority order

**1. The high-value admin screens (was §5.4).** These were never visited and are still unknown. The
registration analysis proved the technique: several are recoverable from `src-analysis/admin/index.js` by
extracting `accessor:"…"` table-column strings and grouping them by the adjacent `fileName:` JSX debug
annotations the "production" build still ships. That is cheaper than a browser session and touches nothing.
Targets: `/languages-management` (the translation-editing UI — a core white-label capability, entirely unseen),
`/staff/details/:id` and `/staff/edit/:id` (the RBAC matrix as rendered — `DATA-MODEL.md` found **15 permission
verbs, not 4**, and this is the screen that assigns them), `/wagering-template/create` (load-bearing for bonus
cost), `/countries` and its restriction screens, and `/categories/addGames/:categoryId`. Also worth settling
from the bundle: whether `FINDINGS.md` §5.1 is right that there is **no affiliate module** — note that
`research/03` §1 found Fundist itself ships a full affiliate suite, so the capability may simply sit at the
aggregator layer rather than the platform layer.

**2. Get three commercial quotes.** This is now the highest-value action in the entire investigation and it is
not a research task. `BUILD-ESTIMATE.md` §5.4 shows the white-label rate and the aggregator rate move the
build-vs-buy answer roughly **six times more** than the whole engineering estimate does. A week spent getting
quotes from a white-label vendor, an aggregator and a licensing consultancy is worth more than any further
analysis of the sandbox.

**3. Request the Fundist specification under NDA.** `research/03` is reconstructed from four independent
operator implementations because **SoftGamings publishes nothing** — `docs.fundist.org`, `apidoc.fundist.org`
and `wiki.fundist.org` are all NXDOMAIN, and `fundist.org/en/API` serves a login shell. The document it is
reconstructing demonstrably exists ("Fundist API OneWallet", version 128). Its §12 lists twelve specific
questions to put to them, ranked by how much each changes the build.

**4. Completeness sweep (was §5.6).** Unchanged and still lowest priority.

### 5.2 Things later work found that the earlier documents got wrong

Read `DATA-MODEL.md` §13 for the full list. The ones that matter most:

- **There is no game round entity, no game-session entity, and no provider-supplied idempotency key** anywhere
  in the observed model. `roundId` appears zero times in the admin bundle. Given that `research/02` §3.4
  documents bets retried 3× and rollbacks up to 500×, this is the highest-risk gap found in the whole
  investigation.
- **It is two API services, not three.** The apparent `api/v3` analytics microservice is on the same host as
  `api/v2`; the separate bases are per-module destructurings of one inlined URL. `ADMIN-BUNDLE-ANALYSIS.md`
  §2.8's claim of a separate sportsbook service is also wrong.
- **RBAC is 15 permission verbs, not 4** — CRUD plus `MM` (manage money), `UW` (user wallet), `I` (issue bonus),
  `E` (export) and others. Those four are exactly what a fraud or regulatory review cares about.
- **The bonus lifecycle in `FINDINGS.md` §1.4 is not a state machine** — Active/Claimed/Forfeit are the player
  site's filter tabs. Claimed and Forfeited are alternative terminals, and Expired and Cancelled also exist.
- **The "coin" balance is not a third balance type**; the sweepstakes model is a second currency, so a second
  wallet row.
- **The KYC vendor is contested:** the admin bundle names Sumsub, the player bundle names Shufti Pro, and both
  ship.
- **`/responsible-gambling` is not missing — it is `/cms/5`.** Several routes recorded as 404s are CMS pages or
  query-parameter filters on `/casino`.
- **`/register` and `/login` were throwing server errors** at capture time (flight-payload digest `3178134716`,
  absent from pages that rendered). An HTTP 200 on this app does not mean the page rendered.

## 6. Document map

| File | Contents | Status |
|---|---|---|
| `README.md` | Entry point and index | ✅ |
| `FINDINGS.md` | Master synthesis of the sandbox investigation | ✅ (see `DATA-MODEL.md` §13 for corrections) |
| `CONTEXT.md` | Glossary — the domain vocabulary these documents use | ✅ |
| `NEXT-STEPS.md` | This file — coverage, gaps, plan | ✅ |
| `BUILD-ESTIMATE.md` | Build cost, effort, break-even, recommendation | ✅ (open items in §4.8; §4.9 backfills most of them) |
| `DATA-MODEL.md` | Reconstructed domain model | ✅ (§12 = what still needs API payloads; §13 = corrections it makes) |
| `docs/adr/0001-read-only-investigation-conduct.md` | Why this was read-only, and the cost of that choice | ✅ |
| `research/01-white-label-landscape.md` | 15 vendors, models, pricing, feature matrices | ✅ |
| `research/02-game-provider-integration.md` | Supply chain, wallet contract, launch flow, certification | ✅ |
| `research/03-fundist-integration.md` | Fundist reconstructed and diffed against Hub88 | ✅ (secondary sources only — see §0) |
| `src-analysis/ADMIN-BUNDLE-ANALYSIS.md` | ~250 endpoints, Redux actions, RBAC, i18n, security | ✅ (§2.8 corrected by `DATA-MODEL.md`) |
| `src-analysis/RETAIL-SOURCE-ANALYSIS.md` | Next.js routes, Server Actions, i18n feature tree | ✅ |
| `src-analysis/RETAIL-REGISTRATION-ANALYSIS.md` | Registration, onboarding, responsible gambling | ✅ |
| `src-analysis/CHAT-APP-ANALYSIS.md` | Chat SPA: socket protocol, cross-origin auth, moderation | ✅ |
| `src-analysis/admin/`, `retail/`, `chat/` | Downloaded bundles | ✅ |

---

## 7. Confidence notes to carry forward

- **Commercial terms are the weakest evidence in this whole investigation.** Revenue shares, minimum
  guarantees and integration fees are confidential industry-wide. Only Digitain and GR8 Tech publish real
  brackets. Everything else is consultancy-blog estimate. Do not let these numbers into a business plan without
  live vendor quotes.
- Six vendor sites blocked automated fetching (403/429); those profiles were reconstructed via search synthesis
  and a read-proxy, not read directly.
- `gammaplus.io` contradicts itself on provider count (50 vs 200+) across its own pages.
- Report 02's wallet API detail is **Hub88's**, used as a documented public stand-in. SoftSwiss, Slotegrator,
  Pariplay and EveryMatrix docs are gated and were not accessible.
- **Report 03 is not primary either, and says so.** SoftGamings publishes no API documentation at all — every
  candidate host is NXDOMAIN or behind a login. The contract is reconstructed from four unaffiliated operator
  implementations in three languages that agree field-for-field. That cross-corroboration is strong evidence of
  the *wire format* and weak evidence of *edge semantics*. Retry and timeout policy, the error-code contract,
  and record retention are simply unknown. Get the spec under NDA before building against it.
- **Two published tax figures in wide circulation are wrong**, and both were corrected only after
  `BUILD-ESTIMATE.md` §4 was drafted (see its §4.9.1). The Curaçao "2% e-zone rate" that nearly every licensing
  broker still advertises was abolished in 2020 — the live rate is 15%/22% tiered. And Malta's Type 1 gaming
  tax **triples from 5% to 15% on 1 October 2026**, which most cost models have not yet absorbed. Whether that
  15% applies to Malta-player revenue or global GGR is unresolved and materially changes the answer.
- **The licence-jurisdiction picture moved during the investigation.** Curaçao's entire regulator board resigned
  in September 2025 and the minister who authored the reform resigned in October 2025 amid a criminal complaint
  about provisional licence issuance. Anjouan's issuing bodies are disputed by the Comoros central bank, which
  says they have "no physical or legal existence". Treat both as live situations to re-check, not settled facts.
- **The engineering-cost and hosting figures in `BUILD-ESTIMATE.md` §4.6–4.7 remain the thinnest inputs**, and
  they are the largest single lever on the model after the two rev-share percentages.
- The security observations are incidental findings, **not a security audit**. No authorization testing, no
  input fuzzing, no attempt to access other tenants' data was performed, and none should be without the
  vendor's written permission.
