# MBO "Modular Back-office" — Platform Investigation

**Target:** `https://mbo.kit.casino` — a second, unrelated vendor's casino back office, reached with credentials
already present in the browser session.
**Investigated:** 2026-08-23.
**Method:** Browser-only. Pages were loaded and navigated as a browser loads them, published static bundles were
read over HTTPS, and the app's own network traffic was observed while navigating. Nothing was submitted, created
or modified. See §10 for the conduct rule and its costs.

Claims are marked `[observed]` (seen directly in the running app or in a shipped bundle) or `[inferred]`
(reasoned from naming, structure or partial evidence).

This is **not** the same product as GammaStack ARC (`src-analysis/ADMIN-BUNDLE-ANALYSIS.md`). It is a different
vendor, a different architecture, and — on the evidence below — a materially more mature platform. §9 diffs the
two directly, which is the part that matters for the build-vs-buy question in `BUILD-ESTIMATE.md`.

---

## 1. What this product is

A back office branded **"Admin Panel"**, served from `mbo.kit.casino`, page title **"Modular Back-office"**.
"MBO" is the vendor's own abbreviation — it prefixes every deployed module (`@MBO/payments`, `MBO-root-config.js`).

The vendor is **not identified anywhere in the shipped artifacts** `[observed]`. There is no copyright string, no
company domain, no "powered by" marker, no vendor name in the OpenTelemetry service metadata, and the only
external domains referenced in the login bundle belong to libraries (`formatjs.io`, `github.com`, `npms.io`).
Identifying the vendor is an open question — see `NEXT-STEPS.md`.

Signals about origin `[inferred]`: the node hierarchy is built around a **"Hall"** (a gaming venue — the
CIS-market retail-gambling term, *игровой зал*), there is a first-class **Telegram Casino** module, and one
permission label contains a Cyrillic homoglyph (§8.4). This reads as a platform built for CIS/Eastern-European
operators, now selling internationally.

The account in this session sees one tenant: **Project `#000103` "Bets Maker"**.

---

## 2. Architecture — the headline finding

**The back office is a micro-frontend application: ~30 independently deployed modules composed at runtime by
single-spa, discovered through a server-served import map.** `[observed]`

```
GET /api/services/import-map
→ {"imports": {"@MBO/dashboard": "/dashboard/dist/MBO-dashboard.2fc341840284796ac319.js", ...}}
```

This is architecturally different in kind from ARC, which ships as **one 8.7 MB Vite bundle**. Here each business
domain is its own build, its own hashed artifact, and its own deployment. The root config loads SystemJS, applies
the import map, and mounts modules by route.

The practical consequences are the interesting part:

- **The module set is data, not code.** The import map is served by the API, so which modules a deployment runs
  is a server-side decision. A tenant can in principle be given or denied a whole business domain without a
  front-end release.
- **Modules deploy independently.** Every module URL carries its own content hash, and the hashes differ in
  length between modules (some 16 hex chars, some 20), which indicates genuinely separate build pipelines rather
  than one build emitting many chunks `[inferred]`.
- **The shared runtime is externalised.** React, Ant Design, Redux Toolkit, axios, moment and the rest are loaded
  once from a CDN as UMD globals, not bundled per module (§7). That is what makes ~30 modules affordable.

### 2.1 The 30 modules

| Module | Bundle size | Reachable in this tenant's menu |
|---|---:|---|
| `@MBO/ccr` | 8.3 MB | yes — "Casino Constructor" |
| `@MBO/sso` | 7.1 MB | yes — "Players", "Permissions and Roles" |
| `@MBO/bi` | 6.9 MB | yes — "Business Intelligence" |
| `@MBO/trs` | 6.8 MB | **no** |
| `@MBO/gaming` | 5.6 MB | yes — "Games" |
| `@MBO/fraud` | 5.1 MB | yes — "Risk Management" |
| `@MBO/layout` | 4.3 MB | (shell) |
| `@MBO/dashboard` | 4.3 MB | yes |
| `@MBO/crm` | 4.2 MB | **no** |
| `@MBO/settings` | 4.1 MB | yes |
| `@MBO/userchat` | 3.9 MB | **no** |
| `@MBO/messaging` | 3.7 MB | yes |
| `@MBO/bonuses` | 3.6 MB | yes |
| `@MBO/seo` | 3.5 MB | yes |
| `@MBO/audit` | 3.4 MB | **no** |
| `@MBO/payments` | 3.3 MB | yes |
| `@MBO/affiliate` | 3.3 MB | **no** |
| `@MBO/kyc` | 3.1 MB | yes |
| `@MBO/reports` | 3.1 MB | yes |
| `@MBO/loc2` | 3.0 MB | **no** |
| `@MBO/export-loader` | 2.9 MB | (utility) |
| `@MBO/tournaments` | 2.8 MB | yes |
| `@MBO/sportsbook` | 2.6 MB | yes |
| `@MBO/bet-trade` | 2.2 MB | yes — "Predictor" (access denied) |
| `@MBO/dis` | 1.9 MB | **no** |
| `@MBO/telegramCasino` | 1.8 MB | **no** |
| `@MBO/bulk-actions` | 1.5 MB | **no** |
| `@MBO/wfe` | 1.1 MB | **no** |
| `@MBO/login` | 1.0 MB | (auth) |
| `@MBO/runtime` | 180 KB | (infrastructure) |

Total shipped front-end: **~130 MB of JavaScript across 30 modules** `[observed]`.

**Eleven business modules are deployed but not reachable by this account** — `trs`, `crm`, `userchat`, `audit`,
`affiliate`, `loc2`, `dis`, `telegramCasino`, `bulk-actions`, `wfe`, and `bet-trade` (present in the menu but
returning "you don't have access"). The permission taxonomy (§5) confirms these are real product surfaces, not
dead code: it carries dedicated categories for CRM, Audit log, Telegram Casino, Affiliate Providers and Dis.

`wfe` is almost certainly a **workflow engine** and `dis` a **dispute** module `[inferred]`; `trs` is the largest
unreachable module at 6.8 MB and its only observed endpoint is `/api/trs/scheduled-job`, suggesting a
scheduling/rules service `[inferred]`.

---

## 3. Tenancy — a five-level node tree

This is the deepest structural difference from ARC, and it is worth understanding properly.

Every page in the back office is **bound to a level of a node hierarchy**, and refuses to render until a node of
that level is selected. The node picker names the five types explicitly `[observed]`:

```
Root  →  Root_folder  →  Project  →  Folder  →  Hall
```

The tenant in this session:

```
Project [#000103] Bets Maker
└── Folder [#000104] Main
    ├── Hall [#000105] Main (Default)        ← live; USD; EN, ES
    ├── Hall [#000106] testkit.kit.kit.casino   (archived)
    └── Hall [#000261] CJK Test Casino          (archived)
```

A **Hall** carries its own currency set, language set, default flag and comment, and has both a display id
(`#000105`) and a UUID exposed in the UI as **`X-Node-ID`** (`7364bd1f-9ea8-40b1-9ac5-91ee6f34224c`)
`[observed]`. The header name makes the tenancy mechanism explicit: node scope is carried per request as a
header, not baked into the URL `[inferred]`.

In CONTEXT.md terms: **Hall is the tenant**, and Project/Folder are grouping levels above it. This is a
meaningfully richer model than ARC's single "Portal" scoping field — it supports an operator running many brands
under one commercial entity, with settings inherited down the tree.

### 3.1 Every route's required scope `[observed]`

All 53 routes reachable from this account's menu, with the node level each demands:

| Scope | Count | Routes |
|---|---:|---|
| **Hall** | 25 | `settings/{hall-settings,domains,languages}`, `payments/{payment-methods,contracts,auto-withdrawal}`, `gaming/{game-provider-settings,game-list-settings,game-groups,game-types,game-sessions}`, all 7 `bonuses/*`, `constructor/lobbies`, both `tournaments/*`, both `messaging/*`, both `seo/*` |
| **Project** | 16 | `settings/{project-settings,social-networks}`, `sso/{auth,players,registrations-players,blocked-players,change-request}`, `payments/{transactions,pps-keys}`, `reports/gaming-activity`, `kyc/*` (3), `fraud/*` (2), `sportsbook/bet-list` |
| **Root** | 8 | `sso/{roles,visit-logs}`, `gaming/game-contracts`, `reports/{deposits-withdrawals,credit-consumption,bonus-reports,tournament}`, `bi/casino-metrics` |
| **Root_folder** | 2 | `settings/node-details`, `bi/players-statistics` |
| *access denied* | 2 | `payments/payment-systems`, `bet-trade/settings` |

The distribution is itself informative. **Configuration is per-Hall; identity, compliance and money movement are
per-Project; commercial reporting and role administration are per-Root.** Game *contracts* sit at Root while game
*settings* sit at Hall — i.e. the aggregator relationship is negotiated once centrally and merchandised per
brand. That is a coherent design, and it is the shape a multi-brand operator actually needs.

---

## 4. Backend service topology

The front end talks to **one origin** (`mbo.kit.casino`) under `/api/`, with the second path segment naming a
backend service. 66 distinct service prefixes appear across the bundles `[observed]`. The ones confirmed live by
observing traffic:

| Prefix | Service | Confirmed endpoints `[observed]` |
|---|---|---|
| `baa` | Bonus admin | `/admin/{bonuses,bonuses/fortuneWheels,cashbackCampaigns,quests,loyaltyAccounts,playerBonusFreeSpins,betPercents,dictionaries}` |
| `taa` | Tournament admin | `/admin/{tournaments,transactions,dictionaries}` |
| `pso` | Payment systems/orchestration | `/{methods,contracts,payprokeys,autoWithdrawals,projects/settings,paymentsystems/all/projects}` |
| `pts` | Payment transactions | `/{transactions,aggregations}` |
| `game` | Game catalogue | `/mbo/{games,providers,groups,game-types}`, `/{callback,lobby}` |
| `sso` | Identity + players | `/{players/management,projectsettings/*,dropdown/<Enum>,users/timezone}` |
| `meta` | Metadata/settings engine | `/{node/<uuid>,nodes,settings/definitions/<uuid>/instance,settings/instances/search,classifier/definitions/<uuid>/instance}` |
| `msg` | Messaging | `/{history,providers}` |
| `ccr` | Casino constructor | `/api/{lobbies,markets}` |
| `afrd` | Anti-fraud | `/{dashboard,fraud/dashboard,reports/games-ggr}` |
| `reports` | Reporting | `/dictionaries/{game-provider-names,api-provider-names,bonus-names}` |
| `templates` | Per-user UI state | `/Templates/width-<table>-<nodeUuid>-<userUuid>` |
| `idp` | Keycloak | `/realms/<uuid>/protocol/openid-connect/*` |

Also present in bundles but not exercised here: `trs`, `dis`, `bettrade`, `bulk`, `audit`, `crm`, `affiliate`,
`referral`, `segmentation`, `kyc`, `seo`, `bi`, `balance`, `player`, `sport/seamless`, `tg`, and a set of
three-letter codes (`gns`, `mps`, `jfs`, `pls`, `pto`, `mls`, `mhs`, `msp`, `mss`, `bojs`, `ccs`, `afs`)
`[observed, unmapped]`.

Two design notes worth carrying into any build:

- **`/api/meta/settings/definitions/<uuid>/instance` is a generic settings engine.** Settings screens are not
  bespoke pages; they render from a definition fetched by UUID, which is why `/settings/project-settings`
  produces a generic `Name | Description | Value` table. Several distinct definition UUIDs recur across unrelated
  modules — the same engine backs domains, SEO, fraud, auto-withdrawal and hall settings `[observed]`.
- **`/api/sso/dropdown/<EnumName>`** serves domain enumerations generically (`UserStatusEnum`,
  `RegistrationCountry` observed live; `LocalizationProjectEnum`, `ChannelNameEnum`, `TemplateTypeEnum` found in
  bundles) `[observed]`.

---

## 5. Permissions and roles

The role editor is the single richest artifact in the product `[observed]`.

- **715 permissions**, each a *resource × action* pair. Actions seen: `View`, `Create`, `Edit`, `Delete` — granted
  per resource, and genuinely varied (e.g. `Countries` offers View/Create/Delete but not Edit; `CAPTCHA Domain`
  offers View only; `Currencies icon` offers Create only).
- A **"New permissions"** counter (0 here) — the platform tracks permissions added since a role was last
  reviewed, so upgrades surface newly-introduced permissions for explicit decision. That is a mature detail.
- **11 default roles**, none with users assigned in this tenant: `SuperAdmin`, `Admin`, `ClientSupport`,
  `SEOManager`, `FinanceManager`, `ContentManager`, `Agent`, `CRMManager`, `DataAnalyst`, `LocalizationManager`,
  `Predictor`. Custom roles are supported and empty here.

The permission categories are the clearest inventory of the platform's true surface area — **22 categories**,
against the 16 sections this tenant's menu exposes:

> Settings · Meta data · Permissions and Roles · Players · Payments · Games · Reports · Business Intelligence ·
> Risk Management · Bonuses · Affiliate Providers · Dis · Casino Constructor · KYC · Tournaments · Messaging ·
> SEO · Sportsbook · OpenAPI Content/FAQ/SEO · Audit log · Telegram Casino · CRM

Resource names inside the `Node details` category alone give a sense of the granularity: `Game currencies
settings`, `Game RTP volatilities`, `Restricted content settings`, `Bonus Settings`, `CAPTCHA`, `CAPTCHA Domain`,
`Contact`, `Contact List`, `Countries`, `Black List Country`, `Crypto-Fiat`, `Currencies`, `Currencies icon`,
`Currency by Node UID and Project UID`, `Currency By Language`, `Parent Currencies`, `Currency Node`, `Currencies
network`, `Domains`, `Domains SEO Setting`, `Domains ID`, `Domains list`, `Domains main`, `Google Analytics`,
`Google Analytics List`, `IPs black`, `IPs white`, `IPs`, `Languages`, `Parental languages`, `Menu Settings`,
`Widget`, `MSP settings`, `Manage Node Child`, `Manage Node`, `Manage Sub Trees Node Structure`, …

Note `Parent Currencies` / `Parental languages` / `Manage Sub Trees Node Structure`: **configuration inherits
down the node tree** `[inferred]`, which is the natural companion to §3.

---

## 6. Domain model, as evidenced by the UI

### 6.1 Game session — the entity ARC does not have

`/gaming/game-sessions` is a first-class **game session** grid `[observed]`. Columns:

```
UUID · Start time · End time · Session activity · Player login · Player nickname · Player UUID ·
Provider · Game external ID · Game name · Currency ·
GGR [Total|Real|Bonus] · Balance [Total|Real|Bonus] · Total bet [Total|Real|Bonus] ·
Total win [Total|Real|Bonus] · Bet count [Total|Real|Bonus] ·
Average bet (real + bonus) · RTP % (real + bonus) · Jackpot count · Total jackpot · Session status
```

Two things matter here.

**First, the session is a modelled entity with a UUID, a start, an end and a status** — exactly the lifecycle
object `README.md` §2 records as *missing* from ARC. A round entity is still not directly evidenced, but a
session with an explicit close is the harder half of that problem.

**Second, every money aggregate is split three ways — total, real, bonus.** Bonus-funded and real-funded play are
separated at the session level, not reconstructed later in a report. This propagates consistently: the gaming
activity report and the bonus report both carry `Combined money / Real money / Bonus money` tabs, and
`/reports/gaming-activity` additionally splits `Win amount without promo`, `Promo amount` and `GGR without promo`
`[observed]`. A platform that gets bonus-vs-real separation right at the ledger level is solving the thing that
makes bonus liability reporting honest.

### 6.2 Banking transactions

`/payments/transactions`, tabs **Requests (0) | Issues (0) | History** `[observed]`:

```
ID · PlayerID · Direction · Amount · Currency · Initial balance · Final balance ·
Status · State · Method type · Payment system · Method name · Created · Updated · Comment
```

`Status` and `State` are **two separate lifecycle fields**, which usually means one is the payment-provider state
and the other the operator's internal workflow state `[inferred]`. `Initial balance` / `Final balance` are
recorded per transaction — a balance snapshot on every movement, which makes the grid reconcilable without
replaying the ledger. The `Requests` and `Issues` tabs are a withdrawal-approval queue and an exceptions queue
`[inferred]`.

### 6.3 Player (PAM)

List columns `[observed]`: `Login · Nickname · Player ID · External ID · Player Affiliate ID · Name · Last name ·
Email · Phone · Country · Document number · Registration date · Last Visit · Last visit IP · Test account ·
Status`.

`External ID` alongside `Player ID` is worth noting — an external identity is modelled as a first-class field,
which is the shape needed when an aggregator keeps its own player record (`README.md` §3, the Fundist problem).

The player card carries nine sections `[observed]`: **Information, Casino bonuses, Player Statistic, Tournaments,
Documents and verification, Change log, Responsible gaming, Player Fraud Profile, Player's message history** —
plus a **Duplicates** panel with a count, a header **Profile risk** badge (`Unmonitored`), verification badges for
email and phone, and header actions **Deposit / Withdraw / Logout** (force-logout of the player's session).

**Personal data is masked by default.** Login, nickname, email and date of birth render as `***` rather than
values `[observed]`. Unmasking appears to be a separate, presumably permissioned action `[inferred]`. This is a
genuinely good default for a back office and something ARC does not do.

The `Responsible gaming` tab manages player limits directly ("Player has no active limits", with an *Add new*
action) `[observed]`.

### 6.4 Bonus

`/api/baa/admin/*` backs seven screens: **List of bonuses, Cashback campaigns, Loyalty, Quest, Wheel of fortune,
Bonuses log, Module settings** `[observed]`.

- **Loyalty** is dual-track: `Loyalty level` + `Loyalty points` *and* `Experience level` + `Experience points`
  per account, with a `Bonus allowed` flag.
- **Free spins** are modelled separately from bonuses (`playerBonusFreeSpins`), with `Free spin quantity / Used
  free spins / Left free spins`, a `Free spin UUID`, a `Free spin campaign status` distinct from the instance
  `Status`, and both `Currency` and `Account Currency` — i.e. a free-spin grant can be denominated differently
  from the wallet it settles into.
- **Module settings → `betPercents`** is a `Game provider × Game type → Percent` table. In CONTEXT.md language
  this is the **wagering contribution** matrix, and it is configured per provider *and* per game type.
- **Cashback campaigns** carry `Bonus visible for player` and `Restrictions` as first-class fields.

### 6.5 Reporting

Report column sets are unusually complete `[observed]`:

- **Deposits-withdrawals** (tabs: by currency / by contracts / by players) — includes `Deposit commission`,
  `Withdrawal commission`, `In-Out`, `Hold %`, request sums separate from settled sums, `Correction Up/Down`
  count and sum, and `Refund` sum and count.
- **Gaming activity** (tabs: raw data / **cryptofiat** / by games / by providers / by currency / by players ×
  combined/real/bonus) — includes `Rollback sum`, `Jackpot count`, `Jackpot amount`, `Session count`,
  `Technology`, and the promo split noted above.
- **Credit consumption** — `Game API provider × Game provider × Currency`, splitting `Real money`, `Bonuses
  manual`, `Bonuses automatic`. This is a *supplier cost* report, i.e. what the operator owes the aggregator.
- **Tournament** — 26 columns including `Guaranteed prize pool` vs `Fact prize pool` vs `Given prize pool`,
  `Tournament strategy`, `Fee`, `Total profit`, `Manual/Auto Subscribed`.

A distinct **`Rollback sum`** column in two separate reports means rollbacks are modelled as their own operation
rather than a flag — the thing `README.md` §2 faults Fundist for `[observed]`.

The dashboard KPI set: Bets, Wins, GGR, Deposits Sum/Count, Sportsbook open bets, First Deposits Sum/Count,
First Deposit Share %, Withdrawals Sum, Payment refund count/sum, In-Out, Hold %, Pending Withdrawals, Active
Players, RTP %, Average Bet, Reg Count, Correction up/down sum, Registration to Deposit %.

### 6.6 Casino Constructor — the site builder

`/constructor/lobbies` is a **lobby builder that publishes a working player site** `[observed]`. The tenant holds
~8 lobbies; one carries a **`Published`** badge and offers *Open lobby*, the rest are drafts offering *Edit
lobby*. Each shows a last-edited timestamp.

**"Open lobby" on the published lobby opens `https://kit.kit.casino/`** — the live player-facing site for this
tenant `[observed]`. It is an unbranded template ("LOGO CASINO") with a Casino/Sport toggle and a complete
product surface: game categories, all games, live casino, card games, sportsbook (with Live and Football),
bonuses, tournaments, loyalty programme, quests, about/FAQ/T&C, and a full compliance set — **Privacy, KYC,
Self-exclusion, Responsible Gaming, Refund and Withdrawal policies**.

So the constructor is not a theming toy: the back-office lobby definition *is* the player site. Behind it,
bundles reference `/api/{pages,panels,widgets,panelsProperties,widgetProperties}`, `/api/data/{themes,
constructor-rules,default-page-types}`, `/api/lobbies/v2/` and `/api/{market-widgets,markets}` — a page/panel/
widget composition model `[observed in bundles, UI not opened]`.

Alongside it sit four asset families — `colorSchemes`, `FontSchemes`, `presets`, `scripts` — each exposing the
**same editorial lifecycle**: `submitForApproval`, `publish`, `unpublish`, `moveToDrafts`, plus `setPriority` and
`setProjects` `[observed in bundles; the workflow UI was not reachable in this tenant]`. A design system with
draft → approval → publish, scoped to chosen projects, is a CMS-grade capability and not something white-label
back offices usually have.

### 6.7 Game catalogue

`/api/game/mbo/*` `[observed]`. Providers carry `UUID · Subprovider ID · Name · Public name · Proxy enabled ·
Active on the hall`. Games carry `UUID · Game external ID · Provider · Subprovider ID · Group · Type · Device ·
Active · Restricted countries · Technology · URL`.

`Subprovider ID` on both models the `aggregator → studio` distinction that CONTEXT.md insists on. `Restricted
countries` per game is geo-restriction held as catalogue data. `Proxy enabled` per provider suggests the platform
can front the studio connection itself `[inferred]`.

---

## 7. Tech stack `[observed]`

| Layer | Choice |
|---|---|
| Composition | **single-spa 6.0.0** + **SystemJS 6.14.3** (+ AMD extra), server-served import map |
| Framework | **React 18.2.0** (UMD, external) |
| UI kit | **Ant Design 5.27.2** (UMD, external) — confirmed by `ant-*` classes and `css-var-r0` |
| State | **Redux Toolkit 1.9.5** + **react-redux 8.1.2** |
| HTTP | **axios 1.5.1** |
| Dates | **dayjs 1.11.10** *and* **moment 2.30.1** + **moment-timezone 0.5.45** (both shipped) |
| Auth | **Keycloak** OIDC — authorization code + **PKCE (S256)**, silent SSO via hidden iframe, `client_id=mbo`, realm `489d740d-4fc8-4d25-9051-96196d48fb0e`, proxied at `/api/idp/` |
| Observability | **OpenTelemetry** browser tracing (`/api/otel/v1/traces`) + **Sentry** |
| Analytics | **Google Tag Manager** `GTM-KR3FFP5S` + **GA4** `G-0PKMWBQ80M` |
| Edge | **Fastly / Varnish** |
| Dev tooling in prod | **import-map-overrides 3.1.1**, plus `unarchiver.js` from `xenova.github.io` |

Localisation is served per module and per feature: `/api/localization/{antd-frontend,uikit-frontend,sso-front,
sso-dict,settings-front,bonuses-front,sportsbook-frontend,predictor-front}/feature/en.json` — a dedicated
localisation service with per-micro-frontend dictionaries, matching the `@MBO/loc2` module.

Per-user UI state is persisted server-side: `/api/templates/Templates/width-<table>-<nodeUuid>-<userUuid>` stores
column widths per table, per node, per user. There is also a **Views** control on grids (saved filter sets)
`[observed]`.

---

## 8. Incidental observations

**These are incidental findings from ordinary use, not a security audit.** No authorization testing, input
fuzzing or cross-tenant access was attempted, and none should be without the vendor's written permission.

### 8.1 The entire runtime is loaded from public CDNs with no Subresource Integrity

Fourteen third-party scripts — including React, React-DOM, Ant Design, Redux Toolkit, axios and SystemJS — are
loaded from `cdn.jsdelivr.net` at page load, and **not one carries an `integrity` or `crossorigin` attribute**
(`integrity=` appears 0 times in the served HTML) `[observed]`. One further script, `unarchiver.min.js`, is
loaded from **`xenova.github.io`** — an individual's GitHub Pages site.

For a back office that moves money and displays player PII, that is a broad third-party trust surface: anything
that can serve those URLs can execute in an authenticated operator session. Pinning with SRI, or self-hosting,
is the standard mitigation and costs essentially nothing.

### 8.2 `import-map-overrides` ships enabled in production

`window.importMapOverrides` is live, with `addOverride`, `getOverrideMap`, `enableUI` and friends, and its custom
element is present in the DOM `[observed]`.

This is a development tool: it lets any micro-frontend module URL be repointed at an arbitrary script, and it
persists the override in `localStorage` so it survives reloads. It is not remotely exploitable on its own — an
attacker needs script execution in the user's browser already, or a user willing to paste something into a
console. What it changes is the *blast radius*: it turns a transient XSS, or a successful "paste this to fix your
account" social-engineering attempt, into a **persistent, reload-surviving replacement of a business module** in
an authenticated back-office session. Shipping it disabled in production builds is the obvious fix.

### 8.3 Google Tag Manager and GA4 run inside the back office

GTM and GA4 load on authenticated back-office pages `[observed]`. GTM is a remote-code-injection channel by
design, and back-office URLs carry player UUIDs in the path (`/sso/players/details/<uuid>/information`). Whether
any personal data actually reaches Google was **not** tested. Worth a question to the vendor.

### 8.4 A Cyrillic homoglyph in a permission label

The permission `Game сurrencies settings` uses **U+0441 CYRILLIC SMALL LETTER ES** in place of Latin `c`
`[observed, verified by code point]`. Cosmetically invisible; practically it means the permission cannot be found
by searching for "currencies", and it is the kind of defect that propagates if permission names are ever used as
identifiers.

### 8.5 A duplicated permission category

Both **`Affiliate Providers`** and **`AffiliateProviders`** appear as categories in the role editor `[observed]`.
A taxonomy that has drifted into two spellings of one category is a small but real data-quality signal.

### 8.6 The environment is a near-empty sandbox

One live Hall, 21 players (masked, several flagged as test accounts), zero payment transactions, zero pending
requests, zero users assigned to any role, and a dashboard reading `0,00` across every KPI for the current month
`[observed]`. Nothing here demonstrates the platform under real load, and no report was seen populated with real
data. That is a material limit on everything above: **this is an inventory of capability, not evidence of
correctness.**

---

## 9. MBO versus GammaStack ARC

The comparison that matters for `BUILD-ESTIMATE.md`:

| | **ARC** (GammaStack) | **MBO** |
|---|---|---|
| Front-end architecture | One 8.7 MB Vite bundle | ~30 micro-frontends, single-spa + runtime import map |
| Tenancy | A "Portal" scoping field | Five-level node tree (Root → Root_folder → Project → Folder → Hall), scope enforced per page |
| Auth | In-house, bearer token | **Keycloak OIDC, auth-code + PKCE, silent SSO** |
| Game session entity | **Absent** | **Present**, with UUID, start/end, status |
| Round entity | Absent | Not directly evidenced |
| Real vs bonus money | Reconstructed in reports | **Split at session level**, propagated through every report |
| Rollbacks | A flag on a transaction (via Fundist) | A distinct `Rollback sum` measure in two reports |
| Permissions | RBAC present | **715 permissions × 4 actions, 22 categories, 11 default roles, "new permissions" tracking** |
| PII handling | Values shown | **Masked by default** in the player card |
| Aggregator model | Single aggregator row (SoftGamings/Fundist) | `Provider` + `Subprovider ID` throughout; `Proxy enabled` per provider |
| Supplier cost reporting | — | **Credit consumption report** (what the operator owes the supplier) |
| Modules beyond casino | Sportsbook, chat | Sportsbook, CRM, affiliate, audit, workflow engine, disputes, Telegram casino, user chat, "Predictor" |

**MBO is the more mature product on every axis this investigation could reach.** The three findings that most
directly touch `README.md`'s conclusions:

1. **The game-session gap is closed.** README §2 identifies the absent session/round entity as ARC's central
   correctness failure. MBO has an explicit session with a lifecycle, and separates real from bonus money inside
   it. If a seamless-wallet callback API is the unavoidable engineering (and it is), MBO demonstrates the
   data model a correct one needs — which makes it a better reference design than ARC for the build case.
2. **Tenancy is a solved problem here, and it is expensive to build.** The five-level tree with per-page scope
   binding, inherited configuration, and per-node/per-user UI state is a large amount of un-glamorous
   infrastructure. Any build estimate that treats multi-brand tenancy as a column on a table is wrong.
3. **The permission matrix is 715 entries.** That is not a feature you add later; it is threaded through 30
   modules. It should be costed as such.

None of this changes the build-vs-buy arithmetic in `BUILD-ESTIMATE.md` §7 directly — that turns on the rev-share
spread, not on feature counts. But it raises the bar for what "parity" means, and MBO is a second data point
that the buy side of the trade is buying something substantial.

---

## 10. Conduct

Same rule as `docs/adr/0001-read-only-investigation-conduct.md`, applied to a second vendor:

- Pages were loaded and navigated as a browser loads them. Published static bundles were fetched over HTTPS.
- The app's own network calls were **observed** by instrumenting `fetch`/`XMLHttpRequest` in the page; the API
  was **not** driven directly, and no request was issued that the application did not make itself. The one
  exception is `GET /api/services/import-map` and the module bundles it names, which are the published asset
  manifest and its assets.
- Nothing was submitted, created, edited or deleted. No form was posted. No deposit, withdrawal, bonus grant or
  player modification was performed. Selecting a node and toggling "show archived" change only what the operator
  sees.
- Personal data was not extracted. The platform masks it by default and it was left masked; only field
  *structure* is recorded above.

Costs of the rule, carried honestly: request and response **shapes are unknown**, so §6 describes entities via
their UI projection rather than their API contract. Endpoint coverage is limited to what the 53 reachable pages
happened to call — 28 distinct business endpoints observed, against 66 service prefixes present in the bundles.
Eleven deployed modules were never exercised at all.

`NEXT-STEPS.md` §4 records direct read-only API probing as an open question for the user; it applies equally here.

---

## 11. Open questions

1. **Who is the vendor?** Nothing in the product identifies them. Ask, or ask whoever supplied the credentials.
2. **Commercial model** — white label, turnkey or licence? On whose gambling licence does a Hall operate? This
   is the fork that `CONTEXT.md` says decides more than the price.
3. **Which of the 11 unreachable modules are licensable**, and at what cost — CRM, affiliate and the workflow
   engine are significant products in their own right.
4. **Is there a round entity** beneath the session, and what does the seamless-wallet callback contract look
   like? This is the single most important technical question and it is not answerable from the back office.
5. **What does `trs` do?** Largest unreachable module, 6.8 MB.
6. **Does GTM/GA4 receive player identifiers** from back-office pages?
7. **Can a populated demo tenant be provided?** Everything above is capability inventory against an empty
   environment.

**Resolved during the run:** the player-facing site is **`https://kit.kit.casino/`**, reached from Casino
Constructor → the published lobby → *Open lobby* (§6.6). It was not analysed — doing so is the obvious next
session, and it is where the wallet, game-launch and registration flows actually live.
