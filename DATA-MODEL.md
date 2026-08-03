# Reconstructed Domain Data Model — GammaStack "ARC" Casino Platform

Reconstruction of the entities, attributes and relationships behind the ARC platform, derived from the admin
SPA bundle, the player-site Next.js build, the rendered screens recorded in `FINDINGS.md`, and the
seamless-wallet contract in `research/02-game-provider-integration.md`.

**No API payloads were captured.** Everything here is recovered from client-side artifacts. Read §12 before
treating any of it as a build spec.

Date: 2026-07-31.

---

## 0. How to read this document

Every relationship and every non-obvious attribute carries an evidence marker:

| Marker | Means |
|---|---|
| `[observed]` | Directly present in a rendered screen, a literal string in a downloaded bundle, or an endpoint path. The name is real. |
| `[inferred]` | Reasoned from convention, from an endpoint's shape, or from a foreign-key-looking column. Plausible, not evidenced. |

Attribute names in `code font` are **verbatim strings from the bundles** unless marked `[inferred]`. Where a
field must exist but its name was not recovered, this document says *unknown* rather than inventing one.

### 0.1 Where the evidence came from

Five extraction techniques over `src-analysis/admin/index.js` (8.3 MB) and the retail build:

1. **Table column definitions** — `accessor:"…"` strings, grouped by the adjacent `fileName:` JSX debug
   annotation that the "production" build still ships. This maps ~190 real field names onto specific screens,
   and therefore onto specific entities. Highest-value technique by a wide margin.
2. **Formik `initialValues` + Yup schemas** — recovers create/edit form shapes and which fields are required.
3. **Enum object literals** — `{FREESPINS:"freespins",…}`-shaped constants give exact wire values.
4. **The bundled i18n resource** — a flat 2,114-key English object whose keys are frequently the field names.
5. **Endpoint paths** — the `rn` namespace map plus ~250 call sites.

### 0.2 Two corrections to the earlier reports

Both matter for §11, so they are stated up front.

- **`ADMIN-BUNDLE-ANALYSIS.md` §2.8 says base `J0` "looks like a separate sportsbook microservice." It is
  not.** `J0`, `Sr`, `Xr` and `Xx` are all separate per-module destructurings of the *same* inlined
  `VITE_APP_API_URL` — every one resolves to `https://white-label-adminapi.gammaplus.io` `[observed]`. The
  "outside `rn`" calls are simply paths written by hand instead of via the namespace constant. Likewise `Tr`
  and `Ir` are both literally `"/api/v2"`. There is no second admin host in this bundle.
- **The v3 analytics API is on the same host too.** `${Xx}/api/v3/dashboard-v2/…` resolves to
  `https://white-label-adminapi.gammaplus.io/api/v3/…` `[observed]`. v2 and v3 are path-versioned on one
  origin, not two origins. The only genuinely separate hosts are the player API, the chat app, and the
  sportsbook renderer.

---

## 1. Entity map

```
                          ┌─────────────────────────────────────────┐
  TENANCY                 │ Application / Brand (settings/application)│
                          │  layout · logo · constants · module flags │
                          └──┬────────────┬──────────────┬───────────┘
                             │            │              │
      ┌──────────────────────┘            │              └──────────────┐
      │                                   │                             │
┌─────▼──────┐  IDENTITY          ┌───────▼────────┐ MONEY      ┌───────▼───────┐ GAMING
│  Player    │◄────┬──── Tag      │    Wallet      │            │  Aggregator   │
│            │     ├──── Segment  │ (1 per         │            │      │ 1:N    │
│            │     ├──── Device   │  currency)     │            │  Provider     │
│            │     └──── Comment  └───┬────────────┘            │      │ 1:N    │
└──┬─┬─┬─┬───┘                        │                         │    Game ◄─┐   │
   │ │ │ │                            │ 1:N                     └──────┬────┼───┘
   │ │ │ │  ┌─────────────────────────▼──────────────────┐             │ N:M│
   │ │ │ │  │  Banking Txn   │  Casino Txn  │  Ledger    │             │ Category
   │ │ │ │  │ (deposit/wd)   │ (bet/win/    │  entry     │             │
   │ │ │ │  │                │  refund)     │            │             │ N:M
   │ │ │ │  └────────┬───────┴──────┬───────┴────────────┘         Country
   │ │ │ │           │              │                            (restriction)
   │ │ │ │   ┌───────▼──────┐  ┌────▼─────────┐
   │ │ │ │   │  Withdraw    │  │ Game round?  │  ← not modelled admin-side (§9.5)
   │ │ │ │   │  Request     │  └──────────────┘
   │ │ │ │   └───────┬──────┘
   │ │ │ │           │ N:1
   │ │ │ │    Payment Provider ──N:M── Country
   │ │ │ │
   │ │ │ └── COMPLIANCE:  KYC Document ──N:1── Document Label
   │ │ │                  RG Limit (10 keys)
   │ │ │                  Suspicious Activity
   │ │ │
   │ │ └──── BONUS:  Bonus Instance ──N:1── Bonus Definition ──1:N── Bonus Currency Terms
   │ │                                              │ N:1
   │ │                                      Wagering Template ──1:N── Game Contribution
   │ │
   │ └────── ENGAGEMENT: Tournament ──1:N── Leaderboard Entry
   │                     Notification · CMS Page · Banner · Email Template
   │
   └──────── Referral (player → player)
```

---

## 2. Tenancy

The weakest-evidenced group, because this tenant's back office exposes almost none of it — a white-label
platform's tenant administration is naturally *above* any single tenant's admin. What follows is mostly
inferred from the shape of what a tenant *can* configure.

### 2.1 Application / Brand

The tenant-level configuration record. Reached via `settings/application` (GET) and mutated by
`application/toggle`, `application/update-constants`, `application/update-logo`,
`application/update-value-comparisons`, and `admin/update-site-layout` `[observed]`.

| Attribute | Type | Evidence |
|---|---|---|
| layout / theme | enum `layout1` \| `layout2` \| … | `[observed]` — the player site's middleware rewrite is `x-middleware-rewrite: /en/layout2/casino`, and `admin/update-site-layout` is the mutation endpoint |
| logo | image | `[observed]` `application/update-logo`, multipart |
| "constants" | key/value bag | `[observed]` endpoint name only; contents unknown |
| "value comparisons" | unknown | `[observed]` endpoint name only. Purpose not determined — possibly the dashboard's period-over-period comparison config |
| enabled module set | set of module keys | `[inferred]` from the 34-key permission registry plus the observed fact that whole nav groups are absent for this tenant |
| enabled route set | set of routes | `[observed]` behaviourally — `/vip`, `/promotions`, `/live-casino`, `/providers`, `/responsible-gambling`, `/all` compile into the player bundle but return **404**, not a redirect. Route enablement is server-side data, not a UI flag |

**Relationships**

- Application `1:N` Player — `[inferred]`. No `brandId`/`tenantId` column was found on any player-facing
  table, but a multi-tenant platform must scope players somehow.
- Application `1:N` CMS Page, via a `portal` scope — `[observed]`. `portal` is a real column on the CMS list
  (`accessor:"portal"`), rendered as "All" on this tenant. This is the one place tenant scoping is visible in
  the UI.
- Application `1:N` Currency / Language / Country enablement — `[observed]`; each has an `isActive` toggle
  under `settings/`.

### 2.2 Registration form configuration

A genuinely tenant-scoped entity, recovered in full from `pages/RegistrationFormFields` `[observed]`. This is
the `/form-fields` screen that could not be reached in the browser.

The record carries a `disable` array of field names, and each field renders as a toggle whose handler is
`setFieldValue(name, checked ? 0 : 2)` `[observed]` — so the stored value is an integer, `0` when enabled and
`2` when disabled. The gap at `1` strongly suggests a three-state enum, most likely
`0 = required, 1 = optional, 2 = hidden` `[inferred]`.

Configurable fields, verbatim `[observed]`:

`email` · `password` · `username` · `firstName` · `lastName` · `dateOfBirth` · `phone` · `address` ·
`gender` · `preferredLanguage` · `countryCode` · `currencyCode` · `newsLetter` · `sms`

Notably absent: any jurisdiction-conditional field, any age-verification threshold, any
responsible-gambling acknowledgement, and any Brazilian CPF/CNPJ field — even though the player bundle ships
`cpfError`/`cnpjError` validation strings `[observed]`. Either those are driven by a mechanism not in this
screen, or they are hardcoded per deployment.

---

## 3. Identity

### 3.1 Player

The central entity. Field names below come from `pages/PlayerDetails/TableCol.jsx`,
`pages/Players/hooks/usePlayersListing.jsx`, the player filter schema, and the i18n resource `[observed]`.

| Attribute | Type | Notes |
|---|---|---|
| `id` / `userId` | int | `[observed]` — `/player-details/65` confirms a small integer surrogate key |
| `email` | string ≤50 | `[observed]`; carries a separate verified flag, actioned by `kyc/verify-email` |
| `username` | string | `[observed]` |
| `firstName`, `lastName` | string 3–50 | `[observed]` |
| `dateOfBirth` | date | `[observed]` |
| `phone` | string | `[observed]` |
| `address` | string | `[observed]` |
| `gender` | enum `male` \| `female` \| `unknown` | `[observed]` — note the wire value for "Other" is `unknown` |
| `countryId` | FK → Country | `[observed]` (resolved by id against a countries list in the filter formatter) |
| `preferredLanguage` | FK → Language | `[observed]` |
| `pincode` | string | `[observed]` — a player-search filter field |
| `isActive` | bool | `[observed]` — `player/toggle` |
| `kycStatus` | enum, see §7.1 | `[observed]` |
| `kycMethod` | enum `System KYC` \| `Sumsub` \| manual | `[observed]` — see §7.2 for a vendor contradiction |
| `walletAddress` | string | `[observed]` — paired with a "Metamask Registered Player" label; crypto-wallet signup is first-class |
| `trackingToken` | string | `[observed]` — affiliate attribution |
| `affiliateStatus` | unknown | `[observed]` — single occurrence |
| internal-user flag | bool | `[observed]` — `MARK_USER_AS_INTERNAL_SUCCESS`, `PUT /user/internal`. Excludes staff test accounts from revenue reporting `[inferred]` |
| `referredBy` | FK → Player | `[observed]` |
| `lastLoggedInIp` | string | `[observed]` |
| `createdAt` / registration date | timestamp | `[observed]` |
| 2FA enrolment | enum/bool | `[observed]` player-side only (`select_2fa_method`, `2fa_disabled_by_admin`); no admin field name recovered |

**Relationships**

| From | To | Card. | Evidence |
|---|---|---|---|
| Player | Wallet | 1:N | `[observed]` — `user.user.wallets[0].id` in the retail launch code; one per currency (§4.1) |
| Player | Tag | N:M | `[observed]` — `tag/attach-tag`, `tag/remove-tag`, `userTags` column |
| Player | Segment | N:M | `[inferred]` — segments are *rule-evaluated*, not stored memberships (§8.1). `segment-users` returns the evaluated set |
| Player | Device / login session | 1:N | `[observed]` — `/login-history`, `devices-users` |
| Player | Comment (staff note) | 1:N | `[observed]` — `player/create-comment`, columns `title, comment, commenterName` |
| Player | Player (duplicate link) | N:M | `[observed]` — `duplicate-players`, `duplicate-players/all`. Symmetry and whether the link is stored or computed are both unknown |
| Player | Player (referral) | 1:N | `[observed]` — `referredBy`, `referral/users`, `referral/transactions` |
| Player | KYC Document | 1:N | `[observed]` |
| Player | RG Limit | 1:N | `[observed]` — ten fixed keys, §7.3 |
| Player | Bonus Instance | 1:N | `[observed]` — `bonus/user` |

### 3.2 Admin user (staff)

Columns `fullName, email, roleName, isActive` `[observed]`; create schema
`{email, password, firstName, lastName, role, username, adminRoleId}` `[observed]`.

Two distinct role concepts coexist: a `role` string (values `Support`, and `Manager` seen in the UI) and an
`adminRoleId` FK `[observed]`. Whether `role` is denormalized from the role record or an independent
discriminator is **unknown**.

Staff form a **hierarchy**: `admin/children`, `admin/toggle-child`, `admin/update-child` `[observed]` — so
Admin `1:N` Admin (parent/child), `[inferred]` as a supervisor tree used to scope which staff a manager can
administer.

### 3.3 Permission grant — richer than previously reported

`FINDINGS.md` §1.10 and `ADMIN-BUNDLE-ANALYSIS.md` §4 describe an R/C/U/D matrix. **The real verb set has
15 members**, recovered verbatim from the `HU()` label switch `[observed]`:

| Code | Verb | Code | Verb |
|---|---|---|---|
| `C` | create | `MM` | **manageMoney** |
| `R` | read | `L` | limit |
| `U` | edit | `TE` | testEmail |
| `D` | delete | `VE` | verifyEmail |
| `TS` | toggleStatus | `RP` | resetPassword |
| `A` | apply | `I` | **issue** (bonus) |
| `CC` | createCustom | `E` | export |
| `UW` | **userWallet** | | |

Two modules relabel generic verbs `[observed]`: `gallery` renders `C` as "Upload" and `R` as "View";
`player` renders `C` as "Notify".

This matters. `MM` (manual balance adjustment), `UW` (wallet access), `I` (issue bonus) and `E` (export player
data) are exactly the permissions a regulator or an internal-fraud review cares about, and they are separable
from ordinary CRUD. A build that models permissions as four CRUD bits per module **cannot express this
platform's own access-control granularity**.

Shape: `permission.permission = { [moduleKey]: [verbCode, …] }` over 34 module keys `[observed]`, plus a
`tablePermissions` concept for per-column visibility `[observed]` whose structure was not recovered.

### 3.4 Device / login history

Columns from `pages/PlayerSessions/UserDevicesHistory.jsx` `[observed]`:
`device`, `deviceInfo`, `isSuccessfulLogin`, `ipAddress`, `location`, `createdAt`, `updatedAt`,
`isSuspicious`, `suspiciousActivities`.

One row is one login attempt, successful or not `[inferred]` — `isSuccessfulLogin` on the row means failed
attempts are retained, which is what you need for credential-stuffing detection. `suspiciousActivities` is
plural on a single row, so a login carries a collection of triggered rules `[inferred]` (§7.4).

---

## 4. Money

The most consequential group, and the one where the reconstruction is most complete.

### 4.1 Wallet

**One wallet per player per currency.** `[observed]` — the retail bundle reads `user.user.wallets[]` as an
array and picks `wallets[0].id` as a default; the player UI has a currency switcher and the string
`switchWalletToSelectedCurrency`; `walletId` is a filter field on casino transactions.

Three balances, from the i18n resource `[observed]`:

| Attribute | Meaning |
|---|---|
| `cashBalance` | Withdrawable real money |
| `bonusBalance` | Bonus funds, subject to wagering before release |
| `totalBalance` | `cashBalance + bonusBalance` `[inferred]` — whether stored or computed is unknown |

Plus `walletId`, `currencyId`, and a player FK `[observed]`.

**The "coin" balance is not a third balance type.** `FINDINGS.md` §2 observed a dual balance in the player
header ($1860.00 plus 160.00) and a "10SC Free Coins Await!" banner. The correct reading is that the
sweepstakes Gold-Coin / Sweeps-Coin model is expressed as **two currencies, hence two wallet rows**, not as a
new column — the currency entity already carries a `type` discriminator and the header has a currency
switcher `[inferred]`. No `coinBalance` field exists anywhere in either bundle `[observed]`.

### 4.2 Currency

Columns `name, code, symbol, exchangeRate, type, isActive` `[observed]`. `type` is Fiat/Crypto `[observed]`.
14 configured on this tenant, one flagged primary (USD).

`exchangeRate` is a **scalar column on the currency row**, not a time-series `[observed]`. There is no
rate-history entity and no rate-effective-date anywhere in the bundle. That is a real modelling gap: a
transaction reported months later will be re-valued at today's rate unless the rate is snapshotted onto the
transaction. There *is* a `conversionRate` field on casino transactions (§4.4), which suggests the platform
does snapshot per transaction `[inferred]` — but the two facts were never seen together.

### 4.3 Banking transaction

The deposit/withdrawal ledger. Columns `to, from, amount, bonusAmount, currency, purpose, status, createdAt`
`[observed]`; filters add `type`, `tagIds`, `currencyId` `[observed]`.

`purpose` enum, verbatim `[observed]`:

`Deposit` · `Withdraw` · `BonusDeposit` · `BonusWithdraw` · `BonusClaimed` · `BonusCashed` ·
`TournamentEnroll` · `TournamentWin` · `TournamentCancel` · `TournamentRebuy` · `ReferralDeposit`

`status` enum: `pending` · `completed` · `failed` `[observed]`.

The `from`/`to` pair is the strongest structural signal in the whole money model: this table is
**double-entry-shaped**, recording a movement between two parties rather than a signed delta on one account
`[observed]`. `BonusCashed` as a distinct purpose is the bonus→cash conversion event when wagering completes
`[inferred]`.

### 4.4 Casino transaction

The gameplay ledger. Columns `transactionId, gameId, gameName, from, to, amount, bonusAmount, currencyCode,
actionType, purpose, status, createdAt` `[observed]`. Filter schema adds `transactionType`, `walletId`,
`actioneeId`, `conversionRate`, `previousTransactionId` `[observed]`.

`actionType` enum, verbatim `[observed]` — **six values across two parallel tracks**:

| Cash track | Bonus track |
|---|---|
| `CasinoBet` | `CasinoBonusBet` |
| `CasinoWin` | `CasinoBonusWin` |
| `CasinoRefund` | `CasinoBonusRefund` |

`previousTransactionId` is the link from a win or a refund back to its originating bet `[inferred]` — it is
the structural equivalent of the seamless-wallet `reference_transaction_uuid`. `actioneeId` identifies who
caused the transaction `[inferred]` (the player, or a staff member for a manual adjustment) — it also appears
as a column named `actionee` on the KYC document table, where it means the staff member who acted, which
supports that reading `[observed]`.

### 4.5 Ledger

A **separate** concept from the two transaction tables. `transaction/get-ledgers` (GET), i18n keys `ledgerId`
and `ledgerType` `[observed]`. Only 1 call site and ~12 references — this is a thin, low-traffic screen.

The relationship between Ledger entry, Banking transaction and Casino transaction was **not recovered**. Two
readings are consistent with the evidence:

1. The ledger is the underlying double-entry journal and both transaction tables are typed views over it. The
   `from`/`to` columns on both tables support this `[inferred]`.
2. The ledger is a separate, coarser summary artifact and the transaction tables are the real records.

The bundle cannot distinguish them. §12 lists this as the single most important open question.

### 4.6 Manual balance adjustment ("Manage Money")

A real, fully-recovered form — and a compliance-sensitive one `[observed]`:

```
{ amountType, addAmount, transactionType, currencyId }
```

`addAmount` is validated as `min(0.01).max(10000)` `[observed]`. So a single staff adjustment is capped at
10,000 units, client-side. `amountType` is `[inferred]` to select cash vs bonus, and `transactionType` to
select credit vs debit; neither option list was recovered. Gated by permission verb `MM` (§3.3).

### 4.7 Withdrawal request

Columns `userId, email, name, amountWithCurr, paymentProvider, transactionId, statusText, actionableType,
updatedAt` `[observed]`. Filters: `searchString, status, fromDate, toDate, paymentProvider` `[observed]`.

`amountWithCurr` is a **pre-formatted display string** ("amount with currency"), not a numeric column
`[observed]` — a presentation concern leaking into the API contract, and a small but real sign that this
surface is built for the screen rather than for a consumer.

`actionableType` is `[inferred]` to be a polymorphic discriminator (the Rails/Sequelize `*_type` convention),
naming what kind of record the request targets. `statusText` alongside a separate `status` filter suggests the
API returns both a machine status and a rendered label `[inferred]`.

### 4.8 Payment provider and method

Two levels `[observed]`:

- **Provider** — create schema `{name, description, aggregator, category, image}`. 39 integrations available,
  10+ configured on this tenant. Note `aggregator` on a *payment* provider: PSPs are themselves aggregated.
- **Method** — columns `methodId, methodName, methodCode, minimumDeposit, minimumWithdraw, bankCode,
  bankName`. `bankCode`/`bankName` indicate bank-transfer methods carry a bank sub-selection (consistent with
  the observed `GET_OFAPAY_BANKS_SUCCESS` action).

Per-provider limits, from i18n `[observed]`: `minimumAmount`, `maximumAmount`, `processingTime`,
`processingFee`, `depositFee`, `withdrawFee`, `transactionLimit`, `dailyLimit`, `monthlyLimit`, `annualLimit`.

Provider `N:M` Country `[observed]` — the payment create wizard has a country multi-select.
Deposit and withdraw are **separate provider roles** `[observed]`; several catalogue entries are explicitly
"Withdraw Provider".

Third-party credentials live in a separate namespace: `internal/credentials`,
`internal/create-credentials`, `internal/update-credentials`, multipart `[observed]`. Modelled as its own
entity keyed by integration `[inferred]`.

---

## 5. Gaming

### 5.1 The catalogue hierarchy

```
Aggregator ──1:N──► Provider ──1:N──► Game ──N:M──► Category
```

All three levels `[observed]` in the rendered screens and in the column definitions.

| Entity | Attributes | Evidence |
|---|---|---|
| **Aggregator** | `id`, `name`, `isActive`, `mobileImage`, `desktopImage` | `[observed]` — mutation payload `{aggregatorId, mobileImage, desktopImage}` |
| **Provider** | `casinoProviderId`, `name`, `casinoAggregator` (FK), `isActive`, sort order | `[observed]` — `reorder-provider` |
| **Game** | `casinoGameId`, `uniqueId`, `name` (localized object, `name.EN`), `code`, `providerName`, `casinoProviderId`, `returnToPlayer` (RTP), `category`, `devices`, `isActive`, `isFeatured`, `operatorStatus`, `iconUrl`, `desktopImage`, `mobileImage`, `demoAvailable` | `[observed]` |
| **Category** | `id`, `nameEN`, localized `name`, `isActive`, `mobileImage`, `desktopImage`, sort order | `[observed]` — `reorder-category` |

Two attributes deserve comment:

- **`uniqueId` vs `casinoGameId`** — the player-side launch URL is built from `game.uniqueId`, while admin
  tables key on `casinoGameId` `[observed]`. These are different identifiers on the same entity: an internal
  surrogate key and the aggregator's game code `[inferred]`. Any build needs both, and needs to be clear
  about which one is stable across an aggregator catalogue refresh.
- **`operatorStatus` alongside `isActive`** — two independent enable flags on a game `[observed]`, appearing
  together on the restricted-games screens. `[inferred]`: `operatorStatus` is the aggregator's own
  availability signal and `isActive` is the operator's merchandising toggle. You cannot merchandise a game the
  supplier has withdrawn, so both are needed.

**Ordering is a stored attribute, not a computed one.** Four separate reorder endpoints and dedicated
`reorderId`/`orderId` columns `[observed]`. Sort position is persisted per game, per category, per provider
and per bonus.

### 5.2 Geo-restriction

First-class and bidirectional `[observed]`:

| Relation | Endpoints |
|---|---|
| Game `N:M` Country | `restrict-countries-for-game`, `remove-restricted-countries-for-game`, `countries/restricted-games/:id` |
| Provider `N:M` Country | `restrict-countries-for-provider`, `remove-restricted-countries-for-provider`, `countries/restricted-providers/:id` |

Both are **deny-lists** `[observed]` — the screens are "Restricted Countries" and the country-side screens
distinguish `restricted-items` from `unrestricted-items`.

This is the operator-side counterpart to the aggregator's per-game `blocked_countries` /
`restricted_countries` / `certifications` arrays documented in `research/02` §3.3. The evidence does **not**
show ARC ingesting those aggregator fields — the restriction tables look manually curated `[inferred]`. If so,
that is a compliance gap: a newly-added game inherits no restrictions until someone sets them.

### 5.3 Game session and game round — **absent**

This is a significant negative finding.

**The admin bundle contains no round entity.** `roundId` appears zero times; `round` appears only inside
unrelated minified identifiers; there is no round column on any table, no round filter, and no round endpoint
`[observed]`. There is also no game-session entity — no session id, no session start/end.

What exists instead is the **casino transaction** (§4.4), keyed on `transactionId` with a
`previousTransactionId` back-link. A round is therefore reconstructable as a chain of transactions, but it is
not a first-class record with its own identity or close flag.

`research/02` §3.4 is explicit that a `round` grouping plus an explicit `round_closed` flag is required to
settle multi-leg rounds correctly — cascading wins, bonus rounds spanning multiple spins. Either ARC stores
rounds in a table the admin never surfaces `[inferred]`, or it genuinely reconciles at transaction
granularity. From the client alone this cannot be resolved, and it is the second-most-important open question
in §12.

The v3 analytics service does expose `casino-session-length` `[observed]`, which implies *something*
session-shaped exists server-side — but nothing in the v2 admin surface reads or writes it.

---

## 6. Bonus

The deepest sub-model, and the one where the recovered enum values are most complete.

### 6.1 Bonus definition

`bonusType` enum, recovered verbatim from a single object literal `[observed]` — 12 values, against the 3 this
tenant exposes:

```js
{ FREESPINS:"freespins", DEPOSIT:"deposit", JOINING:"joining", LOOSING:"loosing",
  COUPON_CODE:"coupon_code", PROMO_CODE:"promocode", RAKEBACK:"rakeback",
  REFERRAL:"referral", BIRTHDAY:"birthday",
  DAILY_SPIN_WHEEL:"daily-spin-wheel", WEEKLY_SPIN_WHEEL:"weekly-spin-wheel",
  MONTHLY_SPIN_WHEEL:"monthly-spin-wheel" }
```

Note `loosing` — a misspelling of "losing" (cashback on losses) baked into the wire protocol, and rendered in
the UI as `DEPOSIT(CASHBACK)` `[observed]`. A reimplementation that "fixes" the spelling breaks compatibility.

Attributes, from the list columns and the create schema `[observed]`:

| Attribute | Notes |
|---|---|
| `promotionTitle` | localized |
| `bonusType` | above |
| `percentage` / `bonusPercentage` | match rate |
| `isSticky` | sticky bonus — winnings from bonus funds are not withdrawable until wagering completes `[inferred]` |
| `bonusBetOnly` | the bonus may only be wagered, not withdrawn `[inferred]` |
| `validFrom`, `validTo` | campaign window |
| `daysToClear` | wagering deadline in days |
| `wageringRequirementType` | discriminator; option list **not recovered** |
| `wageringMultiplier` | e.g. 30× |
| `quantity` | total issuance cap `[inferred]` |
| `maxCouponClaims` | per-coupon cap |
| `couponCode`, `promoCode` | two *separate* fields, matching the two separate enum values |
| `bonusImage`, `description`, T&C | content |
| `claimedCount` | denormalized counter `[observed]` |
| `isExpired`, `isActive` | state flags |
| sort order | `orderId` + `bonus/reorder` `[observed]` |

**Relationships**

- Bonus `1:N` Bonus Currency Terms — `[observed]`. The create wizard's step 2 is per-currency:
  wagering amount, max bonus claimed, min deposit, each keyed by currency. This is a child table, not columns.
- Bonus `1:N` Bonus Content — `[observed]`. Step 3 is localized title/description per language.
- Bonus `N:1` Wagering Template — `[observed]` via `wageringTemplateId`.
- Bonus `N:M` Segment — `[observed]` via `tagIds` on the bonus filter, mapped to the label "segment".
  Confusingly the *same* field name `tagIds` is used for player tags elsewhere; whether bonus targeting uses
  tags or segments is **ambiguous in the evidence**.

### 6.2 Wagering template and per-game contribution

Template columns `wageringTemplateId, name, rtp, contribution`; the per-game child rows carry
`casinoGameName, returnToPlayer, contributionPercentage, wageringContribution` and a `gameContributionPercentage`
`[observed]`.

Wagering Template `1:N` Game Contribution `N:1` Game `[observed]`. The presence of both
`contributionPercentage` and `wageringContribution` on the same row suggests two distinct numbers (a
percentage and a resolved absolute) `[inferred]`, but this was not confirmed.

Only `NO_TEMPLATE` existed on this tenant, so the populated shape was never rendered.

### 6.3 Bonus instance

The per-player grant. Endpoints `bonus/user` (GET), `bonus/issue` (POST), `bonus/cancel` (PUT) `[observed]`;
Redux actions `ISSUE_BONUS_SUCCESS`, `CANCEL_USER_BONUS_SUCCESS`, `GET_USER_BONUS_SUCCESS` `[observed]`.

Attributes recovered: `bonusAmount`, `amountWagered` / `amountToBeWagered`, `claimedAt` / `claimedDate`,
`expireDate` / `expiresOn`, status `[observed]`. Wagering progress is therefore stored as a running
**`amountWagered` against a target**, not as a percentage `[inferred]`.

Player `1:N` Bonus Instance `N:1` Bonus Definition `[observed]`.
Bonus Instance `N:1` Wallet `[inferred]` — bonus funds must land in a specific currency's wallet, and per-currency
terms exist, so the instance must be currency-scoped.

---

## 7. Compliance

### 7.1 KYC document and label

**Document Label** — the operator-configurable list of required document types. 5 on this tenant, localized.
`kyc/document-labels`, `kyc/document-label/create`, `kyc/document-label/update` `[observed]`.

**KYC Document** — the uploaded artifact. Columns `name, url, actionee, status` `[observed]`; plus
`kycRejectDescription` `[observed]`.

Document `N:1` Label `[inferred]`; Document `N:1` Player `[observed]`;
Document `N:1` Admin (via `actionee`, the staff member who verified or rejected) `[observed]`.

### 7.2 KYC method — an unresolved vendor contradiction

The admin bundle's i18n names the methods `System KYC` and **`Sumsub`** `[observed]`. The player bundle's i18n
names **Shufti Pro** (`UseShuftiProtoCompleteYourKYCVerificationInstantly`) `[observed]`, and `FINDINGS.md`
§0 records Shufti Pro as the vendor.

Both strings are real and both ship in production. `[inferred]`: the platform supports multiple KYC vendors
behind a `kycMethod` discriminator and the two front-ends were built against different ones — or one label is
stale. `FINDINGS.md`'s flat claim that the KYC vendor "is Shufti Pro" should be softened to "at least Shufti
Pro on the player side, with a Sumsub integration referenced admin-side."

### 7.3 Responsible-gambling limit

Exactly **ten fixed keys**, recovered verbatim as a complete map `[observed]`:

```js
{ selfExclusion:"self_exclusion",
  dailyBetLimit:"daily_bet_limit",       weeklyBetLimit:"weekly_bet_limit",       monthlyBetLimit:"monthly_bet_limit",
  dailyLossLimit:"daily_loss_limit",     weeklyLossLimit:"weekly_loss_limit",     monthlyLossLimit:"monthly_loss_limit",
  dailyDepositLimit:"daily_deposit_limit", weeklyDepositLimit:"weekly_deposit_limit", monthlyDepositLimit:"monthly_deposit_limit" }
```

Three limit types × three periods, plus self-exclusion. Written by three separate endpoints —
`limit/update-betting`, `limit/update-deposit-and-loss`, `limit/update-self-exclusion` `[observed]`.

Self-exclusion carries its own sub-schema `{permanent, days}` with `days` validated
`min(1).max(2000).positive().integer()` `[observed]`, and the UI offers presets 1 / 7 / 30 / 180 / 365 days
plus permanent (`-1`) and custom `[observed]`.

**The player side has an eleventh limit the admin cannot see**: `sessionLimit`, with its own
required/min validation strings `[observed]`. The admin has no session-limit field. Either it is
player-controlled only, or the admin screen is incomplete.

Also player-side only: `limitCantSetBefore`, `limitSet24Hrs`, `limit24Reset` `[observed]` — a **cooling-off
rule** where loosening a limit takes 24 hours to take effect while tightening is immediate. That is a
regulatory requirement in most licensed markets, and it means a limit record needs a pending-change field and
an effective-at timestamp `[inferred]` — neither of which appears anywhere in the admin model.

### 7.4 Suspicious activity

`suspicious-activities` (GET/POST), `suspicious-activities-settings` (GET/POST), `suspicious-player/toggle`
(POST), `admin/suspicious-login/count`, `admin/update-suspicious-login/isRead` `[observed]`. Real-time push
over socket.io on `/private/suspiciousLogin` `[observed]`.

Three entities `[inferred]` from that endpoint set: a **rule/settings** record (tenant-scoped thresholds), an
**alert** record (with an `isRead` flag), and a **flag on the player** (`suspicious-player/toggle`). The alert
attaches to a login row via `isSuspicious`/`suspiciousActivities` (§3.4) `[observed]`.

---

## 8. Engagement

### 8.1 Segment and segment rule — the non-trivial one

`NEXT-STEPS.md` flagged this as high-value and unseen. It is now fully recovered from
`pages/AdvanceFilter/hooks/useAdvanceFilter.jsx` `[observed]`.

A segment holds `{name, description, isActive, segmentation}` `[observed]` where the rule tree is:

```
segment.groups[i].fields[j] = { field, operator, value1, value2 }
```

`[observed]` — the Formik field paths are literally `groups.${i}.fields.${j}.value1`.

Boolean structure: the validation strings are `atLeastOneOrCondition` ("At least one OR condition is
required"), `atLeastOneCondition`, and `duplicateFieldOperatorCombination` ("Duplicate field-operator
combination across groups") `[observed]`. So **groups are OR-ed and the fields within a group are AND-ed**
`[inferred]` — a two-level disjunctive normal form, not arbitrary nesting. The duplicate check is enforced
*across* groups, which further supports a flat two-level model.

Sixteen operators, recovered as a complete map with their arity `[observed]`:

| Arity | Operators |
|---|---|
| 1 value (`SINGLE`) | `eq` `ne` `lt` `lte` `gt` `gte` `like` `notLike` `exists` `notExists` |
| 2 values (`DOUBLE`) | `in` `notIn` `between` `notBetween` |
| 0 values (`NONE`) | `isNull` `isNotNull` |

Operand fields confirmed by special-cased renderers: `gender`, `deviceType` (`mobile`/`desktop`/`tablet`/`other`),
`country` `[observed]`. A separate date-anchor concept `lType` takes `signup` \| `last_login` \| `last_played`
`[observed]`, and aggregate fields `total_count` \| `id` \| `username` appear in a related map `[observed]`.
The full operand-field catalogue comes from `segments/constants` at runtime and is **not in the bundle**.

Segment membership is **evaluated, not stored** `[inferred]` — `segment-users` is a query endpoint and
`segment-advance-filter` is a POST that evaluates a rule tree. This has a real consequence: a bonus targeted
at a segment re-evaluates eligibility at claim time, so a player can age out of a campaign mid-flight.

### 8.2 Tournament

Create schema `{name, description, creditPoints, startDate, endDate, registrationEndDate, image}`
`[observed]`; sortable columns add `entryFees`, `rebuyFees`, `rebuyLimit`, `minPlayerLimit`, `maxPlayerLimit`,
`tournamentPrizeType` `[observed]`; list columns add `userTags` (segment targeting) and `status`, and the
filter schema adds `isRegistrationClosed` `[observed]`.

**Leaderboard entry** — `leaderBoardId, name, amountSpent, winPoints, points, winPrize` `[observed]`.
Tournament `1:N` Leaderboard Entry `N:1` Player `[observed]`.

**Tournament transaction** — `id, userName, gameName, points, purpose, type, createdAt` `[observed]`. Points
accrual is itself transactional, not a recomputed aggregate `[inferred]`.

Tournament `N:M` Game `[observed]` — wizard step 3 selects eligible games (`code, iconUrl, id, name, category,
provider, isActive`).

Tournament money flows through the banking ledger via the `TournamentEnroll` / `TournamentWin` /
`TournamentCancel` / `TournamentRebuy` purposes (§4.3) `[observed]` — so tournaments are not a separate
money system.

### 8.3 Tag

`tag, isActive` `[observed]`. Player `N:M` Tag via `tag/attach-tag` / `tag/remove-tag` `[observed]`.

### 8.4 Content entities

| Entity | Attributes | Evidence |
|---|---|---|
| CMS Page | `title` (localized), `slug`, `portal`, `isActive`, content | `[observed]` |
| Banner | `id`, `type`, `isActive`, sort order | `[observed]` — `banner/types` implies a fixed placement enum |
| Email Template | `label`, `isDefault`, `id`, type, localized body | `[observed]` |
| Notification | `title`, `description`, `language` | `[observed]` |
| Gallery image | folder, url | `[observed]` — folders and images are separate endpoints |

**Email template type** is a 21-value enum recovered verbatim `[observed]`, and it is unusually informative
because each value names a system event that must exist:

```
welcome · active_user · inactive_user · forgot_password · reset_password · email_verification
document_rejected · document_reminder · document_received · document_verified · document_requested
kyc_activated · kyc_deactivated
withdraw_processed · withdraw_request_received · withdraw_request_approved
deposit_failed · deposit_success
gambling_registration · joining_bonus · password_updated
```

This enum is the best available evidence for two state machines (§9.2, §9.3) — the transitions had to exist
for someone to write a template for them.

---

## 9. State machines

Where a transition is drawn from the email-template enum rather than from an endpoint, it is marked as such.
**Terminal and error transitions are largely unknown**; this section says so rather than guessing.

### 9.1 Bonus instance

```
                      issue (admin) / claim (player)
        [none] ──────────────────────────────────────► ACTIVE
                                                          │
             ┌────────────────────────┬───────────────────┼──────────────────┐
             │                        │                   │                  │
     wagering complete          player forfeits      deadline passes    admin cancels
             │                        │              (daysToClear)           │
             ▼                        ▼                   ▼                  ▼
         CLAIMED                  FORFEITED           EXPIRED           CANCELLED
        (terminal)                (terminal)          (terminal)        (terminal)
```

Evidence: `active` / `claimed` / `forfeited` / `expired` are all `[observed]` — the player bonus page filters
on All / Active / Claimed / Forfeit, and the i18n carries `forfeited`, `expired`, `bonusAlreadyActive`,
`bonusAlreadyCancelled`, `bonusLapsed`. Actions `bonusActivate` and `bonusForfeit` are `[observed]`;
`bonus/issue` and `bonus/cancel` are `[observed]` endpoints.

**Unknowns:**
- `bonusLapsed` is a distinct string from `expired` `[observed]`. Whether it is a separate state or a
  synonym is unknown.
- `CANCELLED` (admin-initiated, via `PUT bonus/cancel`) vs `FORFEITED` (player-initiated) may be one state
  with two causes or two states. Not resolvable.
- What happens to bonus funds and bonus-derived winnings on each terminal transition is **not modelled
  anywhere in the client**. The `BonusCashed` and `BonusWithdraw` banking purposes are the only trace.
- `FINDINGS.md` describes the lifecycle as "Active → Claimed → Forfeit", which is the player-facing *filter
  set*, not the state machine. Claimed and Forfeit are alternative terminals, not sequential.

### 9.2 KYC document

```
   [none] ──request──► REQUESTED ──player uploads──► PENDING
                            ▲                           │
                            │                    ┌──────┴──────┐
                            └────reminder────────┤             │
                                              verify        reject(reason)
                                                 │             │
                                                 ▼             ▼
                                             APPROVED       REJECTED
                                                               │
                                                        re-upload? (unknown)
```

Status values `pending` · `approved` · `rejected` · `requested` are `[observed]` verbatim from the KYC filter
option list. The transitions map onto endpoints `kyc/request-document`, `kyc/verify-document`,
`kyc/reject-document` `[observed]` and are corroborated by the email-template enum
(`document_requested` → `document_received` → `document_verified` / `document_rejected`, plus
`document_reminder`).

`document_received` is an email event with **no corresponding status value** `[observed]` — likely the
notification fired on upload, at the `REQUESTED → PENDING` edge `[inferred]`.

**Separately**, the *player's* KYC state has an activate/inactivate axis (`kyc/activate`, `kyc/inactive`,
email types `kyc_activated` / `kyc_deactivated`) `[observed]` that is orthogonal to any individual document's
status. Whether player-level KYC status is derived from the document set or set independently is **unknown**.

Whether a rejected document can be re-uploaded in place or requires a fresh request is **unknown**.

### 9.3 Withdrawal request

```
   player requests ──► PENDING ──approve──► APPROVED ──payout executes──► COMPLETED
                          │                    │
                       cancel              (failure path unknown)
                          │
                          ▼
                     CANCELLED
```

`Pending` / `Approved` / `Cancelled` are `[observed]` from the rendered `/withdraw-request` queue.
The email-template enum contributes three distinct events `[observed]`:
`withdraw_request_received` → `withdraw_request_approved` → `withdraw_processed` — which is why `COMPLETED` is
drawn as separate from `APPROVED`. Approval is the staff decision; processing is the PSP settlement.

The transaction `status` enum (`pending`/`completed`/`failed`) `[observed]` presumably governs the payout leg.

**Unknowns:** whether a rejected-by-staff outcome is distinct from player-cancelled; what happens on PSP
failure after approval; whether funds are held/reserved on the wallet while pending (there is a
`pendingWithdrawals` i18n key `[observed]`, which suggests they are, but no reserved-balance field exists).

### 9.4 Tournament

```
   create ──► INACTIVE ──toggle──► ACTIVE ──registrationEndDate──► (registration closed, still ACTIVE)
                                     │                                        │
                                  cancel                                   endDate
                                     │                                        │
                                     ▼                                        ▼
                                 CANCELLED ◄───────────────────────────── settlement ──► SETTLED
```

Status values `ACTIVE` / `INACTIVE` / `SETTLED` / `CANCELLED` `[observed]` from `tournamentStatus*` i18n keys
and the `toggle` / `cancel` / `settlement` endpoints. `isRegistrationClosed` is a **separate boolean**, not a
status value `[observed]` — a tournament with closed registration is still running.

`SETTLED` is explicitly staff-triggered (`confirmSettleTournament` confirmation string) `[observed]`, not
automatic on `endDate`.

### 9.5 Game round — cannot be drawn

Per §5.3, no round entity exists in the admin surface. The transaction-level sequence that *stands in* for it:

```
CasinoBet ──► CasinoWin      (linked by previousTransactionId)
    │
    └───────► CasinoRefund   (linked by previousTransactionId)
```
with a parallel `CasinoBonusBet` / `CasinoBonusWin` / `CasinoBonusRefund` track `[observed]`.

There is **no observed close/settle marker**, so "is this round finished" is not answerable from this model.
That is a genuine correctness gap against `research/02` §3.4, not merely an unrecovered field.

### 9.6 Others found

- **Player account**: active ⇄ inactive via `player/toggle`; plus an orthogonal `suspicious-player/toggle`
  flag and an internal-user flag `[observed]`. Not a linear machine.
- **Sports bet** (module disabled here, enum still shipped): `pending` · `won` · `lost` · `refund` ·
  `cashout` · `half_won` · `half_lost` `[observed]`.
- **Banking / casino transaction**: `pending` → `completed` \| `failed` `[observed]`.
- **Chat Rain**: `isClosed` boolean with `validFrom` / `validFor` `[observed]`.

---

## 10. The money model in detail

### 10.1 How the three balances relate

```
  Wallet (one per player per currency)
  ├── cashBalance    ── withdrawable
  ├── bonusBalance   ── locked until wagering completes
  └── totalBalance   ── displayed to the player
```

The evidence for how funds move between them is indirect but consistent:

- Every money-carrying row has **both** an `amount` and a **separate `bonusAmount`** column — on banking
  transactions and on casino transactions alike `[observed]`. So a single transaction can touch both balances
  simultaneously. This is the mixed-funds case: a €10 bet against a wallet holding €4 cash and €6 bonus.
- `actionType` has parallel cash and bonus variants (§4.4) `[observed]`, so the platform *also* records
  wholly-bonus transactions distinctly.
- `BonusDeposit`, `BonusWithdraw`, `BonusClaimed`, `BonusCashed` exist as banking purposes `[observed]` —
  `BonusCashed` being the conversion of cleared bonus funds into cash `[inferred]`.

`[inferred]` conclusion: **bonus funds are a segregated balance on the same wallet, not a separate wallet**,
and the split of any given wager across the two balances is decided by the platform and recorded on the
transaction. The split rule itself (bonus-first? cash-first? proportional?) is **not recoverable** and is a
material behavioural question — it determines wagering-completion speed and therefore bonus cost.

### 10.2 Ledger vs banking transaction vs casino transaction

Three tables, one unresolved relationship (§4.5). What *is* established:

| | Banking transaction | Casino transaction | Ledger entry |
|---|---|---|---|
| Records | deposits, withdrawals, bonus grants, tournament flows | bets, wins, refunds | unknown |
| Party columns | `from`, `to` `[observed]` | `from`, `to` `[observed]` | unknown |
| Type column | `purpose` (11 values) | `actionType` (6) + `purpose` | `ledgerType` `[observed]` |
| Links | — | `previousTransactionId` `[observed]` | — |
| Endpoint | `transaction/banking-transactions` | `transaction/casino-transactions` | `transaction/get-ledgers` |

The shared `from`/`to` shape across both transaction tables is the strongest structural fact here: **this is a
transfer model, not a balance-delta model** `[observed]`. Each row names a source and a destination. That is
the right foundation, and it is what makes a derived-balance design possible.

**Recommendation for a build**: make the ledger the single append-only journal of record, with banking and
casino transactions as typed projections over it, and make wallet balances a materialized aggregate that can
always be rebuilt from the journal. The ARC evidence is compatible with this and does not evidence any
alternative.

### 10.3 Mapping the seamless-wallet contract onto this model

`research/02` §3.2 gives the provider-side contract. The mapping onto ARC's observed entities:

| Seamless-wallet call | ARC entity | Marker |
|---|---|---|
| `POST /user/balance` | read `Wallet.totalBalance` for the token's wallet | `[inferred]` |
| `POST /transaction/bet` | insert Casino Transaction, `actionType = CasinoBet` (or `CasinoBonusBet`) | `[inferred]`, but the enum values are `[observed]` |
| `POST /transaction/win` | insert Casino Transaction, `actionType = CasinoWin`, `previousTransactionId` → the bet | `[inferred]` |
| `POST /transaction/rollback` | insert Casino Transaction, `actionType = CasinoRefund`, `previousTransactionId` → the reversed txn | `[inferred]` |
| `transaction_uuid` (idempotency key) | **no counterpart found** | — |
| `round` + `round_closed` | **no counterpart found** (§5.3) | — |
| `is_free` (free-spin bet) | possibly the bonus `actionType` track | `[inferred]` |
| `currency` mismatch check | `Wallet.currencyId` | `[inferred]` |

Two gaps stand out, and both are the kind that cost money rather than time:

1. **No idempotency key is visible.** `transactionId` is ARC's own identifier; nothing in the model
   corresponds to a *provider-supplied* UUID that the operator must deduplicate on. `research/02` documents
   that bets are retried 3× and rollbacks up to 500× with backoff — without a unique constraint on a
   provider-supplied key, retries double-debit. Any build must add
   `provider_transaction_uuid UNIQUE NOT NULL` on the transaction table. This is the single highest-risk
   omission in the reconstructed model.
2. **No round grouping**, per §5.3 — so multi-leg rounds cannot be settled as units.

The **launch flow** maps cleanly, and matches the observed player-side behaviour `[observed]`: the Server
Action takes `{gameId, demo, walletId, tournamentId?}` and returns `{type, launchGameUrl}`; DEMO mode skips
the wallet entirely, exactly as `research/02` describes for `currency: "XXX"` with no token. The `walletId`
parameter confirms the player picks *which currency wallet* a session runs against — and `research/02` notes
currency cannot change mid-session, so a wallet switch must mint a new token `[inferred]`.

### 10.4 Represent money as integer minor units

**Recommendation: store every monetary value as a signed 64-bit integer in a fixed minor unit, with the scale
recorded per currency. Never floating point, never a language-native decimal that can silently round.**

The reasons, in order of how much they cost when ignored:

1. **The provider contract already is integer.** `research/02` §3.2 documents Hub88 transmitting amounts as
   Int64 scaled ×100,000 (`$3.56 → 356000`). If your internal representation is a float, every callback
   round-trips through a conversion that can round, and rounding in the hot path of a bet/win pair is a
   reconciliation discrepancy that surfaces as a GGR mismatch weeks later.
2. **Reconciliation is exact-match or it is nothing.** §3.5 of that report describes operators running
   parallel GGR figures against the provider's transaction feed. Sub-cent drift makes every comparison
   approximate, and approximate reconciliation cannot distinguish a rounding artifact from an actual
   integration bug.
3. **Crypto forces a wider scale than cents.** This platform carries LTCT/TRC/ERC and a `type` Fiat/Crypto
   discriminator on the currency `[observed]`. Two decimal places is wrong for those. Scale must be a
   per-currency attribute, not a global constant — and ×100,000 as a single global scale is *also* wrong for
   a satoshi-denominated balance.
4. **The bonus engine multiplies.** Wagering requirements are multipliers (30× and up) applied per
   transaction and accumulated over hundreds of spins. Float error compounds across an accumulation; integer
   arithmetic with an explicit rounding policy at a single defined point does not.
5. **Regulators audit the arithmetic.** GLI-19 certification (§5 of `research/02`) involves source review.
   "We use doubles for player balances" is a finding.

The corollary: define **one** rounding point, at the boundary where a percentage becomes an amount (bonus
match, wagering contribution, exchange conversion), make its direction explicit and always in the house's
disfavour, and record the rounded integer — never re-derive it.

Note that this recommendation is **not** evidenced as ARC's own choice. The only numeric hint is that the
manual-adjustment form validates `min(0.01)` `[observed]`, which is a UI concern and says nothing about
storage. ARC's internal representation is unknown.

---

## 11. Service decomposition asymmetry

### 11.1 What is actually separate

Correcting §0.2, the real topology is narrower than the version-number spread suggests:

| Surface | Host | Version | Genuinely separate? |
|---|---|---|---|
| Player API | `white-label-api.gammaplus.io` | `/api/v1` | **Yes** — different host `[observed]` |
| Admin API | `white-label-adminapi.gammaplus.io` | `/api/v2` | **Yes** — different host `[observed]` |
| Analytics | `white-label-adminapi.gammaplus.io` | `/api/v3/dashboard-v2` | **No** — same host, path-versioned `[observed]` |
| Sportsbook renderer | `3.220.85.69:8008` | — | Yes — bare IP, plain HTTP `[observed]` |
| Chat | `white-label-chat.gammaplus.io` | — | Yes — separate Vite SPA `[observed]` |
| Socket | `45.198.14.55:7032` | — | Yes — bare IP, bypasses the CDN `[observed]` |

So it is **two API services plus three peripheral services**, not three API services. The `rn` namespace map
(`/casino-management/`, `/player-management/`, …) is a *path convention within one origin* `[observed]`; it
does not evidence separate deployables.

### 11.2 Where entity ownership sits

`[inferred]` throughout — ownership cannot be proven from a client bundle, only argued from which service
mutates what.

| Entity | Owner | Reasoning |
|---|---|---|
| Player identity, wallet, balances | **Player service (v1)** | The player site's Server Actions read profile and wallets; game launch resolves `walletId` there. The seamless-wallet callback must terminate somewhere with authority over balances, and v1 is the only service the game layer touches |
| Casino / banking transactions | **Player service (v1) writes, admin (v2) reads** | Transactions are created by gameplay and cashier flows, which are v1 paths; the admin surface is filter-and-list only, with no create endpoint for either table `[observed]` |
| Catalogue (aggregator/provider/game/category) | **Admin service (v2)** | All mutations are v2 `[observed]`; the player site only reads |
| Bonus definitions, wagering templates | **Admin (v2)** | All CRUD is v2 `[observed]` |
| Bonus *instances* | **Ambiguous** | `bonus/issue`, `bonus/cancel`, `bonus/user` sit outside the `rn` namespace on v2 `[observed]`, while players claim and forfeit through v1. Genuinely shared write access |
| KYC documents | **Split** | Player uploads via v1; staff verify/reject via v2 `[observed]` |
| RG limits | **Split** | Both sides write, and the player side has a limit (`sessionLimit`) and a cooling-off rule the admin does not model (§7.3) `[observed]` |
| Analytics aggregates | **v3** | Read-only reporting `[observed]` |

### 11.3 What is likely duplicated or denormalized

The specific places where the split is visible in the data:

1. **Player and wallet projections.** The admin cannot plausibly hit v1 per row to render a 25-row player
   table with balances and KYC status. Either v2 reads v1's database directly, or it keeps a projection. The
   `amountWithCurr` pre-formatted string on the withdrawal queue (§4.7) `[observed]` smells like a
   report-side denormalization rather than a live join.
2. **Catalogue copy on the player side.** The player lobby filters by category and provider without an admin
   round-trip; game art comes from a third CDN (`agstatic.com`) `[observed]`. A read-model copy of the
   catalogue almost certainly exists in v1, which introduces a **staleness window on geo-restriction changes**
   — a restriction added in the back office may not take effect in the lobby immediately. Given that
   restrictions are the compliance control (§5.2), that window is worth measuring before trusting it.
3. **Bonus state.** The definition lives in v2 and the instance is written by both. This is the classic
   split-brain location for a bonus engine, and it is exactly where `bonusAlreadyActive` /
   `bonusAlreadyCancelled` / `alreadyClaimedSpinWheelBonus` error strings `[observed]` cluster on the player
   side — the shape of errors you get when two services race for the same grant.
4. **Counters.** `claimedCount` on the bonus `[observed]` and the leaderboard `points` (§8.2) are
   denormalized aggregates that must be maintained across a service boundary.
5. **Translations.** A single flat catalogue is shipped to both apps and to all locales, with observed
   cross-tenant leakage (`Tower.bet` on the ARC instance) `[observed]`. It is duplicated content with no
   per-tenant ownership.

### 11.4 The implication for a build

The asymmetry is an artifact of accretion, not a design — `FINDINGS.md` reaches the same conclusion from the
version numbers alone, and the host analysis above supports it more narrowly than the v1/v2/v3 spread
suggested.

The useful lesson is about **where the seam actually belongs**. The one boundary this platform got right is
that the player API owns the wallet, because that is where the seamless-wallet callback must land and it is
the only place with the latency budget to answer a bet in the hot path. The boundary it got wrong is running
bonus definition and bonus instance on opposite sides of a service line, because bonus state transitions need
to be transactional with wallet movements and they cannot be across that split.

If building: **put the wallet, the ledger, the bonus instance and the seamless-wallet endpoint in one service
with one database and one transaction boundary.** Everything else — catalogue, CMS, segments, reporting,
staff administration — can be split freely, because none of it is in the money path.

---

## 12. Open questions the model cannot answer without API payloads

Ordered by how much a wrong guess would cost.

1. **Is the ledger the journal of record, or a summary?** (§4.5) Everything about how to build the money core
   turns on this, and one GET of `transaction/get-ledgers` would settle it. If the ledger is authoritative,
   the two transaction tables are views and the model is clean; if it is a summary, there are three
   independently-maintained money tables and the reconciliation story is much worse.
2. **How is idempotency enforced on the wallet callback?** (§10.3) No provider-UUID field is visible anywhere.
   Either it exists and is admin-invisible, or ARC deduplicates some other way, or it does not deduplicate.
   Not answerable from the client at all — it needs either the aggregator-facing contract or a payload.
3. **Does a game round exist server-side?** (§5.3) `casino-session-length` in v3 implies something
   session-shaped is stored, but no round or session id appears in any v2 payload shape. If rounds are not
   modelled, multi-leg round settlement is unsound.
4. **What is the cash-vs-bonus debit order on a mixed-funds wager?** (§10.1) Directly determines bonus cost
   and wagering-completion time. Not inferable — it is server logic, not a field.
5. **Is `exchangeRate` snapshotted onto transactions?** (§4.2) `conversionRate` exists as a casino-transaction
   filter field, which suggests yes, but the currency's own rate is a mutable scalar with no history. If rates
   are not snapshotted, historical financial reports change retroactively.
6. **What are the option lists for `amountType` and `transactionType` on manual balance adjustment?** (§4.6)
   These are the enum values behind the single most audit-sensitive staff action on the platform.
7. **What is `wageringRequirementType`?** (§6.1) The bonus model has a discriminator whose values were never
   recovered. It likely distinguishes "wager the bonus" from "wager bonus + deposit" — a 2× difference in
   real wagering requirement.
8. **How does bonus balance behave on each terminal state?** (§9.1) What happens to bonus funds and
   bonus-derived winnings on forfeit, expiry and admin cancel is unmodelled in the client.
9. **What is `tablePermissions`?** (§3.3) Column-level access control is referenced but its structure was not
   recovered. Relevant to whether staff can be prevented from seeing player PII.
10. **Is `disable: 0|1|2` on registration fields really required/optional/hidden?** (§2.2) Only `0` and `2`
    are written by the UI; `1` is inferred from the gap.
11. **Are aggregator-supplied `blocked_countries` ingested, or is geo-restriction manually curated?** (§5.2)
    A compliance question, not an architectural one.
12. **Is player-level KYC status derived from documents or set independently?** (§9.2) And which KYC vendor is
    actually live (§7.2).
13. **Where is tenant scoping?** (§2.1) No `brandId`/`tenantId` column was found on any entity. Multi-tenancy
    may be database-per-tenant, schema-per-tenant, or a column the admin never renders — three very different
    architectures.
14. **How are segment rules evaluated at scale?** (§8.1) The rule tree is recovered but the operand catalogue
    comes from `segments/constants` at runtime, and whether evaluation is real-time or batched is unknown.

Items 1, 2, 4, 5 and 6 are all answerable by **read-only GETs against the admin API with the existing session
token** — which is exactly the decision left open in `NEXT-STEPS.md` §4. Items 2 and 3 may not be answerable
even then, since they concern the aggregator-facing surface that no client ever touches.

---

## 13. Summary of what changed versus the earlier reports

| Claim in an earlier document | Status after this reconstruction |
|---|---|
| `J0` is a separate sportsbook microservice (`ADMIN-BUNDLE-ANALYSIS.md` §2.8) | **Wrong.** Same host as everything else (§0.2) |
| Player v1 / admin v2 / analytics v3 are three separate services (`FINDINGS.md` §0) | **Partly wrong.** v2 and v3 share a host; two API services, not three (§11.1) |
| RBAC is a module × R/C/U/D matrix (`FINDINGS.md` §1.10) | **Understated.** 15 verbs including `manageMoney`, `userWallet`, `issue`, `export` (§3.3) |
| Bonus lifecycle is Active → Claimed → Forfeit (`FINDINGS.md` §1.4) | **Mis-shaped.** Those are filter tabs; Claimed and Forfeited are alternative terminals, and Expired and Cancelled also exist (§9.1) |
| KYC vendor is Shufti Pro (`FINDINGS.md` §0) | **Contested.** Admin bundle names Sumsub; both ship (§7.2) |
| Wallet has cash / bonus / coin balances (`NEXT-STEPS.md` §5.1) | **Two balances plus a total.** "Coin" is a second currency, hence a second wallet (§4.1) |
| Game round is a modelled entity (assumed by `NEXT-STEPS.md` §5.1) | **Not present** anywhere in the admin surface (§5.3) |
