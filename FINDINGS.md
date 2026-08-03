# Casino Management White-Label Platform — Investigation Findings

Investigation of the **GammaStack "ARC"** sandbox (back office + player site), plus market research on how
white-label casino platforms and game-provider integrations actually work.

Date: 2026-07-31

---

## 0. What the sandbox actually is

| | |
|---|---|
| **Product name** | **ARC** — "Design and powered by GAMMASTACK" |
| **Vendor** | GammaStack / GammaPlus (gammastack.com, gammaplus.io) |
| **Back office** | `https://white-label-admin.gammaplus.io` → API `https://white-label-adminapi.gammaplus.io` **`/api/v2`** |
| **Player site** | `https://white-label-2.gammaplus.io` → API `https://white-label-api.gammaplus.io` **`/api/v1`** |
| **Live chat** | `https://white-label-chat.gammaplus.io` — a *separate* Vite SPA embedded via iframe |
| **Game supply** | A **single aggregator: SoftGamings (Fundist)** — ~105 studios behind it |
| **KYC vendor** | **Shufti Pro** (plus a manual fallback flow) |
| **Media/CDN** | `https://gammastack-casino.s3.amazonaws.com/`; game art from `agstatic.com` |
| **Licensee named in footer copy** | HNA Gaming N.V. |

Note the two APIs are **different services on different versions** — player-facing `api/v1` and back-office
`api/v2` (plus a newer `api/v3` analytics service). That is a meaningful architectural signal: this platform
grew by accretion rather than as one coherent API.

The `-2` in `white-label-2` and the player site's `[locale]/layout2` route segment are the giveaway: this is
**one tenant of a multi-tenant, multi-theme white-label deployment**. The theme is a numbered layout
(`layout1`, `layout2`, …), and each layout ships a different subset of player-facing pages — `/vip` and
`/promotions` exist in the codebase but 404 on this tenant.

**Important framing:** the back office is *not* the platform. It is a React SPA reading a REST API. The real
product is the API + the wallet/bonus/game-session engines behind it. Everything below about "features" is
really a description of that API's surface.

---

## 1. Back office (CMS) — what's there

### 1.1 Navigation actually enabled for this tenant

- **Dashboard** — Financial / Casino / Bonuses tabs, date-range presets
- **Players**
- **Reports** — Banking Transactions, Casino Transactions, Game Reports, Player Performance
- **CRM** — Notifications, Email Templates, Support
- **Content Management** — CMS, Image Gallery, Banner Management
- **Payment Management**
- **Bonuses and Rewards** — Wagering Template, Bonus
- **Advance Filter**
- **Casino Management** — Aggregators, Providers, Categories, Games
- **Staff Management** — Staff
- **Player Verification** — KYC Management

Site Configuration screens (Currencies, Languages, Countries, Application Settings/Configuration) exist and work
by direct URL but are not linked from this tenant's menu.

### 1.2 Dashboard KPIs

GGR, NGR, Total Deposit, Total Withdrawal, Daily Active Users, New Signed-up Users, Bonus Claimed Count,
Total Wallet Balance, First Time Deposit (+ count), Deposit Success/Failure Count, Total Casino Bet/Win,
Total Sports Bet/Win, Total Active Users, Revenue, Deposit vs Withdrawal, Conversion Rate.

Note the **sports** KPIs — the platform is casino+sportsbook; this tenant just has the casino vertical enabled.

### 1.3 Casino content management — the interesting part

A clean four-level hierarchy:

```
Aggregator  →  Provider  →  Game  →  Category (many-to-many)
```

- **Aggregators** (1 row): SoftGamings (Fundist). So the operator does *not* integrate studios directly.
- **Providers** (~105, 11 pages): AviatrixDirect, BGaming, 7Mojos, NovomaticGames, 7777GamingSlots,
  ICONIC21Live, Popok, ImagineLive, Galaxsys, EvoplayGames, RAWGames, … all tagged
  `Aggregator = SoftGamings (Fundist)`.
- **Games**: columns are `Is Featured` (toggle), Name, Provider, Aggregator, **RTP**, Category, Icon
  (mobile + desktop), Device Type, Status.
- **Categories** (~35): Slots, Baccarat, BlackJacks, Live Casino, Popular, Top Games, HTML5, Main page, …
  with drag-and-drop **Reorder** and an "add games to category" screen.
- **Geo-restriction is first-class and works in both directions**: restrict countries per game
  (`/casino-games/restrict-countries/:id`), per provider (`/casino-providers/restrict-countries/:id`), and from
  the country side (`/countries/restricted-games/:id`, `/countries/restricted-providers/:id`).

Merchandising (featured flags, category membership, manual ordering) is the operator's main lever — the actual
game catalogue is whatever the aggregator supplies.

### 1.4 Bonus engine

Three bonus types are live in this tenant: **deposit, freespins, joining**. The creation wizard is
**General → Currency → Content**:

- **General**: title, type, bonus percentage, validity date range, **player segment**, days-to-clear,
  active toggle, "visible in promotions" toggle, image, description, T&C.
- **Currency**: *per-currency* wagering amount, max bonus claimed, min deposit amount.
- **Content**: localized title/description per language.
- **Wagering Template** (separate entity, attached to a bonus): per-game **RTP** and **wagering contribution %**.

Player-side bonus lifecycle: **Active → Claimed → Forfeit**.

The bundle reveals considerably more bonus machinery than this tenant exposes: **loyalty levels, spin wheel
(daily/weekly/monthly), referral program, cashback, coupon/promo codes, birthday and cumulative bonuses**.

### 1.5 Player management (Player 360)

Tabs: **Overview, Limits, Reports, KYC, Referrals, Notes**.

- **Overview**: P&L report, transaction counts, game transactions, banking transactions charts; basic info
  (ID, email + verified flag, full name, username, DOB, registration date, status, gender, contact, address).
- **Account actions**: activate/deactivate, mark email verified, **Manage Money** (manual balance adjustment),
  edit user info, reset password.
- **Limits — responsible gambling**: limit types **Wager / Deposit / Loss / Self-Exclusion**, each with
  **daily / weekly / monthly** values.
- Bulk operations (`/players-bulk-update`), user tags, segments, login/device history, IP lookup, duplicate-account
  detection.

### 1.6 KYC

Three tabs: **Manual KYC Labels** (5 configurable required document labels, localized), **Manual KYC Requests**,
**KYC Settings**. Player records carry a "KYC Method" (System KYC vs manual) and a status. Document-level actions
in the API: request, verify, reject (with reason), activate/inactivate.

### 1.7 Payments

- **10+ configured methods** for this tenant: IDEAL, INTERAC, APPLEPAY, GOOGLEPAY, BLIKTRUSTED, CARD, BLIKFTD,
  OPENBANKING, SOFORT, OFAPAY, …
- **39 PSP integrations available to configure**: FonePaisa, CoinPayment, Swift, SBI Debit card, SKRILL,
  NETELLER, CoinGate, Phantom, ipay, Shift4, Pay near me, Prizeout, Triple A, Paynote, Indigrit, Sky24, Wizpay,
  Nexa, HiPay, Neutech, Accurepay, Impaya, Nodapay, CoinsPaid, Paywings, Sky, RPAYOUT, Runpay2, SambhavPay,
  EasyRupia, Saspay, BucksBus, M Paisa …
  Note the mix: European (Skrill/Neteller/Sofort/iDEAL/BLIK), Indian (SBI, FonePaisa, Paywings, SambhavPay,
  EasyRupia, M Paisa), North American (Interac, Shift4, Prizeout, Pay near me) and **crypto** (CoinPayment,
  CoinGate, CoinsPaid, Triple A, Phantom).
- Separate **deposit** and **withdraw** provider roles (several entries are explicitly "Withdraw Provider").
- **Withdraw Requests** queue with Pending / Approved / Cancelled manual review.

### 1.8 Currencies & localization

- **14 currencies**, one flagged **Primary** (USD), each with symbol, exchange rate and **Fiat/Crypto** type:
  USD, AED, INR, SGD, JPY, CAD, CNY, VND, THB, MYR, IDR + crypto LTCT, TRC, ERC.
- **11–12 languages**: EN, PT, DE, NL, FR, ES, JA, FI, IT, ZH, KO (+RU in the admin bundle).
- CMS pages: FAQ, Privacy Policy, Bonus Terms, General T&C, Responsible Gambling — each with a slug and a
  **Portal** scope ("All"), i.e. content can be targeted per brand/skin.

### 1.9 Tournaments

Wizard: **Basic Info → Currency → Games**. Fields: registration end date, start/end date, credit points,
segment, image, active flag, localized title/description across 11 languages. API adds leaderboard,
transactions, cancel and **settlement**.

### 1.10 Staff & RBAC

UI roles seen: **Manager, Support**. The real model in the bundle is much finer — a **module × permission-letter
matrix**: 34 permission-gated modules (`ai, kyc, tag, page, admin, bonus, banner, gallery, limits, player,
report, comment, country, currency, language, emailTemplate, casinoManagement, contentManagement,
applicationSetting, sportsbookManagement, kpiSummaryReport, livePlayerDetail, gameReport, kpiReport, demography,
review, tournamentManagement, paymentManagement, disputeManagement, segment, referral, gamification,
chatManagement, suspiciousLoginNotification`) × R/C/U/D letters, plus per-table column-level sub-permissions.

**These client-side checks are UI gating only** — enforcement must be server-side.

### 1.11 Risk / fraud

A **"Suspicious Activity Alerts"** panel in the top bar, fed in real time over socket.io
(`/private/suspiciousLogin`). Backed by `suspicious-activities`, `suspicious-activities-settings` and
`suspicious-player/toggle` endpoints, plus duplicate-account detection, device fingerprint history and IP lookup.

### 1.12 Modules present in the code but NOT enabled for this tenant

This is the most useful signal about the vendor's full product:

| Module | Evidence |
|---|---|
| **Sportsbook** | `/sports`, `/matches`, `/markets`, `/sports-bets`, `/sports-transactions`, `/bet-settings`; API namespace `/sportsbook-management/` with odds, custom odds, market detach |
| **Live chat** | `/chat/channels`, `/chat/chat-rain`, `/chat/offensive-words`, `/chat/chat-settings` — including **"Chat Rain"** (scheduled in-chat prize drops) |
| **Disputes / ticketing** | `/dispute-management/` with real-time `disputeThreadUpdated` socket events |
| **Gamification** | `/gamification/` — task-based rewards |
| **AI chatbot** | `/ai/chat-bot`, `ai-chat-history` — an in-admin AI assistant (the robot avatar visible bottom-right) |
| **Workflow** | `/workflow/` namespace |
| **Loyalty / VIP, spin wheel, referrals** | bonus-management endpoints; the player site carries **66 VIP/loyalty i18n keys** ("loyaltyClub", 6 levels, `loyaltyCashback`, `loyaltyPerCurrency`) even though `/vip` 404s here |
| **Analytics v3** | a separate newer microservice: `/api/v3/dashboard-v2/*` (casino heat map, session length, bet distribution, bonus liability, bonus expiry trend) |

---

## 2. Player-facing site

- **Stack**: Next.js App Router, route shape `/[locale]/…`, theme via `layout2`. PixiJS is loaded
  (`/assets/pixi-assets/js/pixi-legacy.min.js`) — for in-house instant/crash-style games.
- **Nav**: Home, Casino, Favorites, Recently Played, Transactions, Bonus (Joining / Deposit / Freespins),
  LiveGame.
- **Wallet in header**: dual balance — cash (`$1860.00`) plus a second coin balance (`160.00`) with a currency
  switcher. Banner copy says *"10SC Free Coins Await!"* — SC = **Sweeps Coins**, so the same codebase also
  serves the **US sweepstakes (Gold Coins / Sweeps Coins) model**, not just real-money.
- **Cashier** (`/account/wallet/…`): tabs **Deposit, Withdraw, Transactions, Casino Transactions, My Bonuses**.
  Deposit URL shape `/account/wallet/deposit/{id}?provider=SKRILL`. Transaction ledger shows
  date / amount / amount / transaction type (Credit/Debit) / purpose (Deposit/Withdraw) / status.
- **Casino lobby**: category tabs (All Games, Favorites, Recently Played, plus admin-defined categories) and a
  provider filter — driven directly by the admin's Categories/Providers config.
- **Bonus page**: filters All / Active / Claimed / Forfeit.
- **Footer**: Casino, Contact Us, FAQs, General T&C, Responsible Gambling, Tournaments Terms, Bonus Terms,
  Privacy Policy — these map 1:1 onto the admin's CMS pages.

### 2.1 Game launch flow (observed + confirmed in source)

1. Lobby tile → route `/play-casino/[gameId]/[providerId]/[gameName]/[demoAvailable]`
   (observed: `/play-casino/9/19/Mad%20Buffalo/true`; an optional 5th segment carries `tournamentId`).
2. An interstitial: **"Please Select Game PlayMode" → DEMO | REAL**.
3. Choosing a mode fires a **POST to the same route** — a Next.js **Server Action** (hash
   `cab91b5d…`), taking `gameId, demo, walletId, tournamentId?`.
4. It returns `{ data: { type: "HTML" | "URL", launchGameUrl } }`. `URL` is embedded as an iframe; `HTML` is
   injected into a **Blob-URL iframe**.

**Why this matters for your build:** the aggregator credentials, the operator session token and the launch URL
never touch the browser. The auth token itself lives in an **HttpOnly cookie** and is read via a dedicated
server action — client JS cannot see it. This is the correct pattern, and notably *better* than what the admin
SPA does (see §5). It also matches how the seamless-wallet contract expects session tokens to be minted
server-side.

### 2.2 In-house games

Alongside the third-party iframe titles there is a **first-party Crash game** rendered client-side with
**PixiJS** (~35 dedicated i18n keys: `crashGameTitle`, `autoCashout`, `timerTextFlewAway`, `buttonTextStartAutoBet`,
jackpot rules). Provably-fair crypto positioning runs through the copy. So the platform is not purely an
aggregator front-end — it has its own RGS-equivalent for instant games.

### 2.3 What the translation catalogue reveals

All 8 shipped locales use a **single flat `home` namespace with 1,796 keys** — effectively the whole product's
copy. Feature areas by key count: Auth/Account 316, Wallet/Payments 188, Errors/Validation 166, Bonus/Promotions
157, Sports betting 89, Responsible Gambling 84, **VIP/Loyalty 66**, KYC 52, Games/Casino 49, CMS/Legal 42,
Referral/Affiliate 34, Crash game 34, Chat/Support 29, Tournament 22.

Residual strings expose the platform's other deployments: Brazilian **CPF/CNPJ** validation, a hardcoded Indian
`"Earn ₹150 each"` referral string, sweepstakes **SC/GC** coin copy, and — notably — the brand name
**`Tower.bet` left inside a currency string on this ARC-branded instance**. Shared translation catalogues are
not fully re-authored per customer, which is a real brand-hygiene risk for the vendor's operators.

---

## 3. How white-label casinos get their games (the supply chain)

```
Game studio (RGS)  →  Aggregator  →  PAM / platform  →  Operator brand
Pragmatic, Evolution,   SoftGamings/Fundist,   ARC back office     white-label-2
NetEnt, BGaming, …      SoftSwiss, Hub88, …
```

- **Studios** own the game math and run the **RGS** — the RNG executes on *their* servers, never yours. Your
  regulatory duty is to integrate only certified games, not to certify the math.
- **Aggregators** exist because integrating 100+ studios one-by-one is 2–6 weeks *each*. One aggregator
  integration buys thousands of games. The sandbox proves the point: **one** aggregator row, ~105 providers.
- **The price of that convenience** is a slice of GGR and a hard dependency: your catalogue, your uptime and
  your commercial terms are all mediated by the aggregator.

### 3.1 The integration contract you would have to build

Two wallet models:

- **Seamless wallet** (what everyone modern uses): the balance never moves. The studio's RGS calls *your*
  wallet API in real time for every bet and win. Player sees one balance everywhere.
- **Transfer wallet** (legacy): funds are pushed into a game-session wallet and pulled back after.

If you build your own platform, **you must expose a seamless-wallet callback API** and it sits in the hot path of
every spin. Using Hub88's public docs as the concrete reference shape:

| Endpoint | Purpose |
|---|---|
| `POST /user/info` | player demographics |
| `POST /user/balance` | current balance |
| `POST /transaction/bet` | debit the wager |
| `POST /transaction/win` | credit the payout |
| `POST /transaction/rollback` | reverse a prior transaction |

Non-negotiable properties: **idempotency keyed on `transaction_uuid`** (retries must never double-debit),
`reference_transaction_uuid` linking win→bet, a `round` id with a `round_closed` flag, integer money
(×100,000, never floats), and **RSA-SHA256 request signing** rather than a shared static key.

Game launch is a separate API: `POST /game/url` with `user, game_code, operator_id, platform, lang, currency,
game_currency, country, lobby_url, deposit_url, token, sub_partner_id`. **Demo mode = pass `currency: "XXX"`
and omit `token`/`user`** — no wallet calls happen at all. That is exactly the DEMO/REAL split observed on the
player site.

Geo-compliance is data, not code: `/game/list` returns `blocked_countries`, `restricted_countries` and
`certifications` (CURACAO, MGA, IOM…) **per game**, and you are expected to honour them. The ARC back office's
per-game/per-provider/per-country restriction screens are the operator-side UI for exactly this.

### 3.2 Commercials (low confidence — this is all confidential)

Indicative only, from consultancy/blog sources rather than primary disclosure: studio share ~**7–20% of GGR**,
aggregator all-in ~**5–15%**, monthly minimums ~**€5k–25k**. Live-dealer dedicated tables and jackpot
contributions are separately negotiated and materially expensive. **Get real quotes; do not plan on these.**

---

## 4. Market context for build-vs-buy

Three models, routinely conflated:

| Model | Platform owner | License holder | Cost shape | Time to market |
|---|---|---|---|---|
| **White label** | Vendor | **Vendor** (you run under their sub-license) | Low upfront, **15–30% of GGR/NGR** | Days–weeks |
| **Turnkey** | Vendor (licensed to you) | **You** | Mid setup + monthly + smaller rev share | 2–4 months |
| **Self-service / ownership** | **You** | **You** | High upfront, no platform rev share | 3–6+ months |

Published pricing (only two vendors publish real brackets):

| Vendor | Setup | Monthly | Rev share | TTM |
|---|---|---|---|---|
| **Digitain** | €95k–€380k | €15k–€28k | 16–28% | — |
| **GR8 Tech** | €35k–€90k | — | 10–15% | 3–6 weeks |
| SoftSwiss | ~€35k+ | — | ~5–15% NGR *(est.)* | — |
| NuxGame | €15k–€50k | — | 10–18% | 1–4 weeks |
| GammaStack | $10k–$20k basic / $40k–$75k+ premium | — | not published | 2 weeks (WL) |

Cross-market year-one all-in bands: **Starter $27k–70k, Growth $65k–150k, Full-scope $140k–250k+**. Commonly
excluded from base packages: live-dealer content, extra payment rails (~$1k–5k per market), and
**data-migration fees on exit** — the last one is the lock-in that matters.

### 4.1 The licensing fork is the real decision, not the price

The pricing table above is the visible difference between the three models. The structural one is **who holds
the gambling licence**. Under white label you operate on the *vendor's* sub-licence (usually Curaçao) — which is
what makes a two-week launch possible, and also means your right to trade is contingent on their licence, their
compliance standing, and their willingness to keep you. Turnkey and ownership both require you to hold your own.
If you intend to build your own platform eventually, you will have to cross that line regardless, so it is worth
pricing the licence application into the plan from the start rather than treating it as a later step.

### 4.2 Treat the fast time-to-market claims as best-case

Vendors advertise 1–3 weeks (BetConstruct), 2 weeks (GammaPlus), 3–6 weeks (GR8 Tech). Those numbers describe
*technical* deployment. The actual bottleneck in practice is licensing and compliance approval, not integration
— which is precisely the part a white label hides by lending you theirs. Plan your own build against the
regulatory calendar, not the vendor's demo timeline.

### 4.3 Confidence caveat on the vendor research

Several vendor sites (gammaplus.io, everymatrix.com, gr8.tech, digitain.com, softswiss.com, betconstruct.com)
blocked automated fetching with 403/429, so parts of their profiles were reconstructed from search-engine
synthesis and a read-proxy rather than read directly. The Digitain and GR8 Tech brackets are vendor-published
and solid; most other figures are third-party estimates. Also worth noting as a diligence signal:
**gammaplus.io contradicts itself on its own site**, citing 50 providers on the white-label page and 200+ on the
aggregation page. Spot-check anything here against a live vendor conversation before it informs a contract.

Full detail, per-vendor profiles and sources: `research/01-white-label-landscape.md`.

---

## 5. Security observations on the sandbox

Not an audit — these surfaced incidentally. Worth knowing both as a customer and as a builder.

| Finding | Detail |
|---|---|
| **Hardcoded AES key in the admin bundle** | `VITE_APP_FE_ENCRYPTION_KEY = "rb27cry2xn2ysh7823bqxry233x9rn3682323888888q8z90"` ships to every browser and is used (CryptoJS AES) to "encrypt" the access token before writing it to `localStorage`. The key is right there in the same file — this is obfuscation, not encryption. |
| **Token in `localStorage`** | Readable by any JS on the page; no `httpOnly` protection. The AES wrapper does not mitigate XSS token theft. |
| **Player site ships raw backend IPs over plain HTTP — twice** | Live socket.io goes to `http://45.198.14.55:7032/api/socket/` (returning 503), despite the bundled config declaring `SOCKET_URL: https://white-label-api.gammaplus.io`. And `SPORTBOOK_SCRIPT_URL: "http://3.220.85.69:8008/static/js/iframeRender.min.js"` loads **executable third-party JS over unencrypted HTTP from a bare IP**. That second one is the serious finding: no TLS integrity on a script tag is a straightforward MITM script-injection path, and both bypass the Cloudflare edge that fronts everything else. |
| **No CSP, HSTS or X-Frame-Options** on the player app | Checked via `curl -I` on `/en/casino` and `/en/login`. The app embeds third-party iframes (chat widget, game frames) and injects HTML into Blob-URL iframes, so a same-origin XSS would have no backstop and there is no app-level clickjacking protection. |
| **Good practice worth copying** | The player site stores its auth token in an **HttpOnly cookie** and routes sensitive calls (game launch, profile, favourites) through **Next.js Server Actions**, so neither the token nor the real REST contract ships to the browser. Only `NEXT_PUBLIC_CHAT_URL` is inlined; the server-only API base correctly stays out of the client build. |
| **Dev artifacts in the "production" build** | Admin bundle compiles with `MODE:"production"` but `DEV: true`, ships `jsxDEV()` calls and JSX `fileName`/`lineNumber` annotations — internal source paths and component structure are readable. |
| **Full API surface is enumerable** | ~250 endpoints and the microservice namespace map recover cleanly from the client bundle, including `internal/create-credentials` / `internal/update-credentials` (third-party integration secrets) and `admin/create-user`. Normal for an SPA, but it means **all** access control has to be server-side. |
| **Public S3 bucket** | `gammastack-casino.s3.amazonaws.com` used for the CMS media gallery — worth checking ACLs/listing. |
| **Admin app also ships no CSP / HSTS / X-Frame-Options** | `curl -I` on the admin root returns only `server: cloudflare`. Same gap as the player site, on the app that holds every staff session. |
| **The plaintext sportsbook script is currently dead** | `http://3.220.85.69:8008/…/iframeRender.min.js` times out. It doesn't load today, but a bare-IP HTTP script URL shipped in a public bundle is still the wrong shape — if that IP is ever reassigned or spoofed, it executes in the player's page. |

### 5.1 A capability gap worth raising with the vendor

**There is no affiliate or agent management module in this build.** Grepping the admin bundle finds only
per-player affiliate *attribution* — an `Add Affiliate` / `Remove Affiliate` action on a player record backed by
a `trackingToken` and `affiliateStatus` field. There is no affiliate entity, no commission structure, no
affiliate reporting, and no affiliate-facing portal. GammaPlus's own marketing lists "agent/affiliate
management" as a feature of its casino management software. Either it lives in a module not provisioned for
this tenant, or the marketing overstates it — worth asking directly, because affiliate marketing is how most
new casinos actually acquire players.

Separately, a positive architectural signal: the string "Fundist"/"SoftGamings" appears **nowhere** in the admin
bundle. The aggregator is pure data, not hardcoded — which is the right way to build it if you want a second
aggregator later.

---

## 6. What I'd take away for building your own

1. **The back office is the easy part.** ~90 CRUD screens over a REST API. The hard, differentiating parts are
   the **wallet ledger**, the **bonus/wagering engine**, and the **seamless-wallet callback endpoint** — those
   are where correctness is money.
2. **Model the catalogue as `aggregator → provider → game → category` from day one**, with geo-restriction as a
   first-class relation in both directions. The sandbox validates this shape.
3. **Money must be integers**, transactions idempotent on a provider-supplied UUID, and every bet/win tied to a
   `round` with an explicit close. Rollback is a real, frequent path — not an edge case.
4. **Multi-tenancy and theming belong in the architecture from the start**, not retrofitted. The `layout2`
   pattern is a workable model: a numbered theme, plus per-tenant feature flags that disable **whole page
   routes** (not just UI chrome — `/vip` and `/promotions` genuinely 404 here), plus per-tenant module toggles
   in the back office. Also budget for per-tenant content: the vendor's shared translation catalogue leaks
   another operator's brand name into this one, which is exactly the failure mode to design against.
5. **Bonus engine design is the deepest rabbit hole.** Per-currency terms, per-game wagering contribution,
   segment targeting, and a clean Active→Claimed→Forfeit state machine are the minimum. Bonus liability
   reporting is a first-class requirement, not a report you add later.
6. **You will start on an aggregator, not direct studio deals.** Direct integrations only make sense at volume.
   Design the game-supply layer as a pluggable adapter so a second aggregator (or a direct studio) is an
   implementation, not a rewrite.
7. **Do not build client-side security theatre.** No AES keys in bundles; token in `httpOnly` cookies;
   RBAC enforced server-side with the client only hiding UI.

---

## 7. Where everything is

| File | Contents |
|---|---|
| `README.md` | entry point / index |
| `FINDINGS.md` | this synthesis |
| **`NEXT-STEPS.md`** | **coverage matrix, explicit gaps, and the prioritized plan for the next session** |
| `research/01-white-label-landscape.md` | 15 vendors, commercial models, published pricing, feature matrices, sources |
| `research/02-game-provider-integration.md` | supply chain, seamless-wallet API contract with real endpoint/field names, launch flow, certification (GLI-19/GLI-11, MGA/UKGC/Curaçao/Anjouan), commercials |
| `src-analysis/ADMIN-BUNDLE-ANALYSIS.md` | admin SPA reverse-engineering: tech stack, **~250 REST endpoints by module**, 268 Redux action types, RBAC matrix, domain enums, i18n map, security |
| `src-analysis/RETAIL-SOURCE-ANALYSIS.md` | player-site Next.js analysis: routes, API surface, game-launch code, i18n feature tree |
| `DATA-MODEL.md` | reconstructed entity model, state machines, money model, service ownership — **its §13 corrects several claims in this file** |
| **`BUILD-ESTIMATE.md`** | **the build-side counterpart to §4 above: scope tiers, effort, break-even against a revenue share, and a recommendation** |
| `CONTEXT.md` | glossary — the vocabulary used across these documents |
| `research/03-fundist-integration.md` | the aggregator this platform actually uses, reconstructed and diffed against Hub88 — **revises §3's supply-chain model: Fundist is aggregator *and* PAM** |
| `src-analysis/RETAIL-REGISTRATION-ANALYSIS.md` | registration, onboarding and responsible gambling — **corrects §2's route table: several "404" routes are CMS pages** |
| `src-analysis/CHAT-APP-ANALYSIS.md` | the third app: socket protocol, cross-origin auth handshake, moderation |
| `docs/adr/0001-…` | why this investigation was read-only, and what that cost |
| `src-analysis/admin/`, `retail/`, `chat/` | downloaded source bundles |

**Three corrections to this document, from work done after it was written.** Details in the files named above.

1. **§6.3 is right that money must be integers and transactions idempotent on a provider UUID — but ARC does
   neither.** There is no round entity, no game-session entity, and no provider-supplied idempotency key
   anywhere in the observed model. Fundist compounds it: decimal strings rather than scaled integers, and
   rollback expressed as a flag on an ordinary transaction rather than a distinct operation.
2. **§0's "three services on three versions" is wrong.** The v3 analytics API shares a host with v2; the
   apparent third service is a per-module destructuring of one URL. It is two services, not three.
3. **§5.1's positive note — that "Fundist" appears nowhere in the admin bundle, so the aggregator is pure
   data — is correct but incomplete.** The catalogue's natural keys embed the vendor
   (`SOFTGAMING-810-2834532-madbuffalo`), so aggregator identity is load-bearing in the data model even though
   it is absent from the code.
