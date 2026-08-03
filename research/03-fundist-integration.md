# SoftGamings / Fundist — The Actual Integration Contract

**Purpose:** Document the *real* game-aggregator contract used by the platform under investigation, as opposed to the generic seamless-wallet pattern documented in [`02-game-provider-integration.md`](./02-game-provider-integration.md). That report used **Hub88** as a documented public stand-in because every major aggregator's technical docs are gated. This report closes that gap for the one aggregator this platform actually runs: **SoftGamings, whose platform and API are branded *Fundist***.

---

## 0. Source discipline and confidence conventions — read first

**Fundist publishes no public API documentation.** This was verified, not assumed:

| URL tried | Result |
|---|---|
| `docs.fundist.org` | **NXDOMAIN** — host does not exist |
| `apidoc.fundist.org` | **NXDOMAIN** |
| `wiki.fundist.org` | **NXDOMAIN** |
| `docs.softgamings.com` | connection failed, no host |
| `https://www.fundist.org/en/Docs` | HTTP 200, but serves the **login shell** (`<title>System</title>`, username/password + two-factor prompt) — gated |
| `https://www.fundist.org/en/API` | same login shell — gated |
| `https://www.softgamings.com/casino-api/` | reachable, but **marketing page only**; explicitly references "Detailed Documentation" without linking any |
| [`scribd.com/document/740119351/Fundist-API-OneWallet-v128`](https://www.scribd.com/document/740119351/Fundist-API-OneWallet-v128) | **HTTP 410 Gone** — a document titled *"Fundist API OneWallet v128"* is indexed by search engines but has been removed from the host |

So the vendor's own specification — which demonstrably exists, is titled **"Fundist API OneWallet"**, and was at version **128** — **could not be obtained**. Nothing in this report is quoted from it.

**What this report is built on instead.** The contract below is reconstructed from **five independent artifacts that were obtained**, and its confidence rests on *cross-corroboration*: four unrelated operator-side implementations, written in three languages (PHP, TypeScript, C#), by four unaffiliated teams, that agree field-for-field on the wire format. Two of them (the C# and one TypeScript codebase) carry structured comments that are clearly transcribed verbatim from Fundist's official specification — e.g. `Stats/Bets/[CASINO_SERVER_IP]/[TID]/[KEY]/[DATE]/[PWD]` and `// Unique Transaction ID, alphanumeric max 32 chars` — which is how field-length and format constraints below are known.

| # | Artifact | Language | What it establishes |
|---|---|---|---|
| A | [`qsmartisp/casino-backend`](https://github.com/qsmartisp/casino-backend) — `app/Services/Fundist/*`, `app/Http/Controllers/Webhook/OneWalletController.php` | PHP / Laravel | Full outbound client + OneWallet callback handler, HMAC verification, idempotency |
| B | [`ansul-rathi/unifitz-api-main`](https://github.com/ansul-rathi/unifitz-api-main) — `src/interfaces/fundist.interface.ts`, `src/services/fundist.service.ts` | TypeScript | Typed request/response interfaces **with doc-transcribed field constraints**; catalogue field mapping |
| C | [`levi-tech-guru/casineo-main-api-v2`](https://github.com/levi-tech-guru/casineo-main-api-v2) — `src/utils/softgames/**` | TypeScript | The most complete callback set: `ping, balance, debit, credit, roundinfo, rollback, cancel, promotions` |
| D | [`stoura90/fengshuistone`](https://github.com/stoura90/fengshuistone) — `Labixa/Common/SDKApiFundist.cs`, `Labixa/HashMD/HashMD5.cs` | C# | **Widest endpoint surface** (19 methods), with per-method pre-hash strings documented in comments |
| E | [`WebRabbits/checkboxAddDataFunMerchant`](https://github.com/WebRabbits/checkboxAddDataFunMerchant) — `index4.html` | HTML capture | A saved copy of the **Fundist back-office UI itself** (`fundist-logo`, `fundist-bar__uat`), exposing 89 back-office routes |
| F | `src-analysis/retail/pages/casino.html` (this investigation) | JSON payload | The ARC platform's **live catalogue data**, showing exactly how the Fundist feed lands in an operator's model |

**Confidence labels used per section:**

- **[primary]** — verified directly against a Fundist- or SoftGamings-owned host, or against this investigation's own captured artifacts.
- **[secondary — corroborated]** — agreed upon by **two or more** of implementations A–D independently. Treat as high-confidence for field names and wire shape; treat exact optionality/edge semantics as probable, not guaranteed.
- **[secondary — single source]** — appears in only one implementation. Directionally useful, unverified.
- **[not obtainable]** — could not be established; the URL tried is named.

A standing caveat for everything in §2–§8: these are *observed implementations of* the contract, not the contract. Where an implementation is wrong, this report inherits the error. Before building against it, get the real spec from SoftGamings — it exists, and is handed out under NDA.

---

## 1. What Fundist actually is — and why that reframes the integration

Report 02 modelled the industry as four layers: studio → aggregator → PAM → operator. **Fundist is not just the aggregator layer. It is aggregator *and* PAM.** The back-office capture (artifact E) yields 89 routes, and their spread is decisive **[secondary — single source, but a direct product artifact]**:

| Fundist back-office area | Routes observed |
|---|---|
| Player accounts / KYC | `Users`, `Users/Documents`, `Users/Categories`, `Users/AutoCategories`, `Users/Unconfirmed`, `Users/VerificationSettings`, `Users/ManualWithdrawalVerification`, `Users/RegistrationRestrictions`, `DocumentTypes` |
| Money | `Banking`, `Payments/Systems`, `Payments/Config`, `Payments/DepositLimits`, `Payments/AutoWithdrawalRules`, `Payments/RoutingRuleGroups`, `Payments/TrafficRules`, `Payments/Cashback`, `Payments/IncompleteDeposits`, `Payments/Pays/Summary` |
| Game supply | `Merchants/Games`, `Merchants/Games/Permissions`, `Merchants/Countries`, `Merchants/Currencies`, `Merchants/Sorting`, `Merchants/RefundReports`, `Games/Categories`, `Games/Tables`, `Games/BettingLimits`, `Games/RtpExclusions`, `ClientsProviders/Providers` |
| Bonusing / engagement | `Users/Freerounds`, `Loyalty/Levels`, `Jackpots/JackpotList/JackpotBettingStatistics`, `Banners`, `News`, `Users/Lists/Mailing` |
| Reporting | `Reports/Turnover`, `Reports/BetsHistory`, `Reports/Games`, `Reports/BonusTurnovers`, `Reports/MajorWins`, `Reports/PlayerIntelligence`, `Reports/BetRadarBetting`, `Reports/SouthAfrica/TaxReturns` |
| Affiliates | `Affiliates`, `Affiliates/Commissions`, `Affiliates/Referrals`, `Affiliates/Reports`, `Affiliates/Settings`, `Affiliates/External/AffilkaAdminsFee` |
| Compliance / ops | `Licenses`, `Countries`, `IPBlocks`, `GeoipPatch`, `Acl`, `TwoFactorAuth`, `RestrictedNicknames`, `Api/DomainBlocking`, `Api/Licenses`, `Api/Logs`, `Api/RequestViewer` |

**Why this matters for a build decision.** Fundist maintains **its own player record** — that is why the integration has a `User/Add` endpoint at all, and why the OneWallet callbacks identify the player by a `userid` that is a *Fundist login*, not an arbitrary operator token. An operator integrating Fundist is not just consuming a game catalogue; it is **shadow-registering every one of its players into the vendor's system** and keeping that mirror in sync. Hub88 has no equivalent — it takes an opaque `user` string and a `token` and stores no player of its own.

This has direct consequences that recur throughout the diff in §10: player identity, currency binding, and even *account enable/disable state* are replicated across a trust boundary. It is the single largest structural difference from the Hub88 pattern, and it is not visible from the wallet API alone.

The layering also explains the `Merchants/*` vocabulary: in Fundist's model the **game studio is a "merchant"**, and merchant IDs are the join key across the catalogue, the country restrictions, and the launch call.

---

## 2. Authentication — two different schemes, one per direction **[secondary — corroborated, A+B+C+D]**

Fundist uses **asymmetrically different mechanisms** for the two directions of traffic. This is unusual and is the first thing to get right.

### 2.1 Operator → Fundist: positional MD5 pre-hash

Every outbound API call is a `GET`/`POST` to a URL of the form:

```
{base}/System/Api/{API_KEY}/{Method}/?&{Param1=..}&{Param2=..}&TID={tid}&Hash={hash}
```

The `Hash` is `md5()` of a **slash-joined, strictly ordered, positional string**:

```
{Method}/{CASINO_SERVER_IP}/{TID}/{API_KEY}[/{method-specific fields in fixed order}]/{API_PASSWORD}
```

Three credentials are involved, plus a fourth implicit one:

| Credential | Role |
|---|---|
| `API_KEY` | Public-ish identifier. Appears **in the URL path** *and* inside the hashed string. |
| `API_PASSWORD` ("system password") | Shared secret. **Never transmitted** — only ever the trailing segment of the pre-hash. |
| `CASINO_SERVER_IP` | The operator's own outbound server IP, hashed into every request. Effectively binds requests to an IP allowlist **cryptographically**, not just at the firewall. |
| `TID` | Per-request nonce, alphanumeric, **max 32 chars**. |

Verbatim pre-hash constructions, from artifact D's doc-transcribed comments, cross-checked against A and B:

| Method | Pre-hash string |
|---|---|
| `Game/FullList` | `Game/FullList/[IP]/[TID]/[KEY]/[PWD]` |
| `Game/List` | `Game/List/[IP]/[TID]/[KEY]/[PWD]` |
| `Game/Categories` | `Game/Categories/[IP]/[TID]/[KEY]/[PWD]` |
| `User/Add` | `User/Add/[IP]/[TID]/[KEY]/[LOGIN]/[PASSWORD]/[CURRENCY]/[PWD]` |
| `User/AuthHTML` | `User/AuthHTML/[IP]/[TID]/[KEY]/[LOGIN]/[PASSWORD]/[SYSTEM]/[PWD]` |
| `User/DirectAuth` | `User/DirectAuth/[IP]/[TID]/[KEY]/[LOGIN]/[PASSWORD]/[SYSTEM]/[PWD]` |
| `User/KillAuth` | `User/KillAuth/[IP]/[TID]/[KEY]/[LOGIN]/[PWD]` |
| `User/Enable` | `User/Enable/[IP]/[TID]/[KEY]/[LOGIN]/[PWD]` |
| `Balance/Get` | `Balance/Get/[IP]/[TID]/[KEY]/[SYSTEM]/[LOGIN]/[PWD]` |
| `Stats/Bets` | `Stats/Bets/[IP]/[TID]/[KEY]/[DATE]/[PWD]` |
| `Stats/BetsSummary` | `Stats/BetsSummary/[IP]/[TID]/[KEY]/[DATE]/[PWD]` |
| `Stats/Detailed` | `Stats/Detailed/[IP]/[TID]/[KEY]/[DATE]/[PWD]` |
| `Stats/GameDetails` | `Stats/GameDetails/[IP]/[TID]/[KEY]/[GAMEID]/[PWD]` |
| `Tables/LobbyState` | `Tables/LobbyState/[IP]/[TID]/[KEY]/[PWD]` |
| `Providers/TablesInfo` | `Providers/TablesInfo/[IP]/[TID]/[KEY]/[PWD]` |
| `Loyalty/Payment` | `Loyalty/Payment[IP]/[TID]/[KEY]/[SYSTEM]/[AMOUNT]/[LOGIN]/[PWD]` |

Artifact A implements the ordering **generically** rather than per-method, and the resulting rule is worth stating because it is a trap:

> Optional fields are appended in a fixed canonical order — `system, amount, login, password, currency, gameId` — **except that `system` moves to the end for `User/AuthHTML`**.

That is a genuine special case in the signing algorithm, not a quirk of one implementation: artifacts B and D both place `system` after `password` for `AuthHTML` and before `login` elsewhere.

**Assessment.** This is a weak scheme by 2026 standards, and materially weaker than Hub88's. It is **MD5** (broken for collision resistance; still adequate as a pre-image-resistant MAC construction here, but not a defensible choice); it is a **home-rolled positional concatenation** rather than a canonicalised MAC, so it is vulnerable to ambiguity if any field can contain a `/`; it is a **shared secret**, so it gives authentication but **no non-repudiation** — either party could have produced any request. Hub88's RSA-SHA256 asymmetric signing (report 02 §3.2) is strictly better on all three counts. Any risk assessment of a Fundist integration should record this.

### 2.2 Fundist → Operator: HMAC-SHA256 over sorted values

The inbound OneWallet callbacks use a completely different scheme. Artifacts A (PHP) and C (TypeScript) implement it **identically and independently**:

```
key      = sha256_raw( HMAC_SECRET )            // raw 32-byte digest, NOT hex
payload  = concat( values of all params except `hmac`, ordered by key name ascending )
hmac     = hmac_sha256( payload, key )          // lowercase hex
```

with one special case, present in both implementations:

> If the body contains an `actions` array (the `roundinfo` callback), each action object's **values are sorted by that object's key names and concatenated**, all action-strings are concatenated together, and the result replaces `actions` as a single string value before the outer sort.

The double-hashing of the secret (`HMAC-SHA256` keyed by `SHA-256(secret)` rather than by the secret itself) is unusual but consistent across both codebases, so it is part of the contract, not a shared bug.

**The same HMAC is applied to the operator's response.** The operator computes it over its own outgoing fields (again excluding `hmac`) and returns it. This is genuine bidirectional message authentication and is a point where Fundist is *better* than a naive integration — the operator's balance responses cannot be tampered with in transit either.

**Note the concatenation-without-delimiters weakness:** because values are joined with no separator, `{a:"1", b:"23"}` and `{a:"12", b:"3"}` produce the same MAC input. Exploitability is limited by the fixed field set, but it is a real canonicalisation flaw and worth flagging in a security review.

### 2.3 Live verification of the API host **[primary]**

```
$ curl -i https://api.fundist.org/
HTTP/1.1 422 Unprocessable Entity
server: server
content-type: text/html; charset=utf-8

34,Invalid Request
```

Two facts established directly, no intermediary:

1. `api.fundist.org` (178.16.18.162) is live and is the production API host; `apitest.fundist.org` (a CloudFront alias) is the sandbox host, per artifact A's config default.
2. **Fundist's error envelope is a bare CSV string — `{code},{message}` — not JSON.** This matches artifact A's response parser exactly, which splits on the first comma and treats code `1` as success. See §6.1.

---

## 3. The OneWallet callback API — what Fundist calls on the operator **[secondary — corroborated, A+B+C]**

This is the contract the operator must implement. Unlike Hub88, which exposes **five distinct endpoint paths** (`/user/info`, `/user/balance`, `/transaction/bet`, `/transaction/win`, `/transaction/rollback`), Fundist posts everything to **one operator-supplied callback URL** and discriminates on a `type` field in the body.

### 3.1 Request types

| `type` | Purpose | Hub88 equivalent |
|---|---|---|
| `ping` | Health check / connectivity probe | *(none — Hub88 has no ping)* |
| `balance` | Read current balance | `/user/balance` |
| `debit` | Place a wager | `/transaction/bet` |
| `credit` | Pay a win | `/transaction/win` |
| `roundinfo` | Post-hoc summary of a completed round's actions | *(none)* |

Rollback is **not a type**. See §3.4 — this is the most consequential difference in the whole contract.

### 3.2 Request fields

Field constraints marked *(spec)* are from artifact B's doc-transcribed comments.

| Field | Types it appears on | Notes |
|---|---|---|
| `type` | all | one of the five above |
| `hmac` | all | see §2.2 |
| `userid` | balance, debit, credit | The **Fundist login**, i.e. the value the operator passed to `User/Add`. Not an opaque token. |
| `currency` | balance, debit, credit | |
| `amount` | debit, credit | **Decimal string**, e.g. `"10.50"` — *not* a scaled integer |
| `tid` | debit, credit | Fundist's transaction ID. *(spec: alphanumeric, max 32 chars)* — **the idempotency key** |
| `i_gameid` | debit, credit | **Round ID**, despite the name. *(spec: alphanumeric, max 32 chars)* |
| `i_actionid` | debit, credit | Individual game-event reference within the round |
| `i_gamedesc` | balance, debit, credit | *(spec: `"{SystemID}:{GameType}"`, max 64 chars)* |
| `i_extparam` | balance, debit, credit | **Echoed back from the `ExtParam` the operator set at launch.** The operator's own correlation handle. Artifact A packs `"{gameId}|{gameName}"` into it. |
| `i_rollback` | debit, credit | **Presence flips the semantics** — points at the `tid` being reversed |
| `i_reference_actionid` | credit | Links a bonus credit to its originating action |
| `subtype` | debit, credit | debit: `cancel`\|`debit`. credit: `cancel`\|`credit`\|`promotion` |
| `round_start` | debit, credit | boolean — first transaction of the round |
| `round_ended` | debit, credit | boolean — round is closed |
| `jackpot_win` | credit | `1` when the credit is a jackpot payout |
| `game_extra` | debit, credit | *(spec: "PageCode for Evo or `FREEROUNDS_XXX`")* — see §7 |
| `i_flag` | credit | *(spec: "Promo award flag")* **[secondary — single source, B]** |
| `action_details` | debit, credit | *(spec: "Additional round information")* **[secondary — single source, B]** |
| `actions[]` | roundinfo | array of `{actid, type, amount, timestamp}` **[secondary — single source, C]** |
| `mrid` | — | present in artifact A's validation with the comment `// todo: ???` — **purpose unknown** |

### 3.3 Response envelope

Success responses vary by type — the operator must return *only* the fields for that type, because the HMAC covers exactly the fields sent:

| `type` | Response body |
|---|---|
| `ping` | `{ "status": "OK", "hmac": "…" }` |
| `balance` | `{ "status": "OK", "balance": "1234.56", "hmac": "…" }` |
| `debit` / `credit` | `{ "status": "OK", "tid": "…", "balance": "1234.56", "hmac": "…" }` |
| *any error* | `{ "error": "<message>", "hmac": "…" }` — **note: no `status` field at all** |

`balance` is a **decimal string formatted to exactly 2 dp** (artifact A: `number_format($b, 2, '.', '')`; artifact C: `.toFixed(2).toString()`).

**Errors are returned with HTTP 200 and a free-text `error` string.** Artifacts A and C both do this. There is no documented enum of error codes anywhere in the obtained material — the observed strings (`Invalid HMAC`, `Invalid currency`, `Invalid userid`, `Insufficient balance`, `Tid invalid`, `Transaction parameter mismatch`) are each implementation's **own invention**, and they differ between codebases. Whether Fundist parses these strings, or merely treats "response contains `error`" as failure, is **[not obtainable]** — it is the kind of detail that only the real spec settles, and it matters, because it determines whether "insufficient funds" is distinguishable from "your server is broken" on Fundist's side. **Ask SoftGamings this question explicitly during onboarding.**

### 3.4 Rollback is a modifier, not an operation — the big one

In Hub88, reversing a transaction means calling a **different endpoint**, `/transaction/rollback`, with a `reference_transaction_uuid`.

In Fundist, a reversal arrives as an **ordinary `debit` or `credit`** that happens to carry an `i_rollback` field naming the `tid` to reverse. Artifact C's router makes this explicit — it checks for the *presence of the field* before it ever looks at `type`:

```ts
if (Object.prototype.hasOwnProperty.call(data, 'i_rollback')) {
  await handleRollout(data, res)          // rollback path
} else if (Object.prototype.hasOwnProperty.call(data, 'subtype')) {
  …
} else {
  switch (data.type) { case 'ping': … case 'debit': … }
}
```

And the direction inverts: a `type: "credit"` carrying `i_rollback` means *reverse a debit*, so the operator **adds** funds; a `type: "debit"` carrying `i_rollback` means *reverse a credit*, so the operator **subtracts** them.

**This is a live foot-gun.** An implementation that dispatches on `type` first — the obvious design, and the one the Hub88 contract trains you toward — will process a rollback as a fresh bet or a fresh win, doubling the error instead of undoing it. Any adapter interface that normalises Fundist to a Hub88-shaped `{bet, win, rollback}` operation set must branch on `i_rollback` *before* `type`, and must invert the sign. Note also that `subtype: "cancel"` is a **third**, separate reversal-ish path in artifact C, distinct from `i_rollback` — the precise difference between a `cancel` subtype and an `i_rollback` reversal is **[not obtainable]** from the material gathered.

### 3.5 Idempotency and replay

Artifact A implements a two-level scheme, and its structure implies the contract's requirements:

1. **Replay detection by `hmac`.** The full request HMAC is stored. If a request arrives with an HMAC already seen *and* an identical body, the **stored prior response is replayed verbatim** — no re-application. `type: "balance"` is excluded (balance is not idempotent in a meaningful sense; it is just re-read).
2. **Conflict detection by `tid`.** If a `tid` has been seen but with a *different* `type`, `userid`, `currency`, or `amount`, the operator raises **transaction parameter mismatch** rather than applying anything. This is Fundist's analogue of Hub88's `RS_ERROR_DUPLICATE_TRANSACTION`.

**Retry policy is [not obtainable].** Hub88 publishes its retry semantics precisely (3 retries at 1s for bet/win; up to 500 rollback attempts with exponential backoff — report 02 §3.2). Nothing equivalent was found for Fundist in any obtained artifact. Since retry aggressiveness directly determines how hard the operator's idempotency layer is exercised, **this is a required question for onboarding**, not a nice-to-have.

Likewise **[not obtainable]**: any minimum retention period for transaction records (Hub88 mandates 4 months), and any documented callback timeout.

---

## 4. Game launch — `User/AuthHTML` **[secondary — corroborated, A+B+C+D]**

Fundist has no `/game/url`-style JSON endpoint. Launch goes through `User/AuthHTML`, which authenticates the player **and** returns the game frame in one call.

```
GET {base}/System/Api/{KEY}/User/AuthHTML/?&Login={login}&Password={password}
    &System={systemId}&Page={pageCode}&UserIP={ip}&Currency={cur}
    [&Demo=1][&Country=..][&Language=..][&Nick=..][&Timezone=..]
    [&ExtParam=..][&UserAutoCreate=1][&UniversalLaunch=1][&IsMobile=1]
    &TID={tid}&Hash={md5}
```

| Parameter | Meaning |
|---|---|
| `Login` / `Password` | **The player's Fundist credentials.** Not a session token — see below. |
| `System` | The **MerchantID** — which studio's system to launch into |
| `Page` | The `PageCode` from the catalogue — which specific game |
| `UserIP` | Player's IP, forwarded for the studio's own geo/fraud checks |
| `Currency` | Wallet currency |
| `Demo` | `1` for fun mode |
| `ExtParam` | Operator-defined opaque string, **echoed back on every wallet callback as `i_extparam`** |
| `UserAutoCreate` | Create the Fundist user on the fly if absent |
| `UniversalLaunch` | Present in artifacts A and B; semantics **[not obtainable]** |
| `IsMobile` | Device hint (artifact A) |

**The credential model is the second big structural difference.** Hub88 launches a game with an operator-minted, short-lived `token` that the operator itself validates on each wallet callback. Fundist launches with a **`Login` + `Password` pair for a persistent Fundist account** — which is why artifact A maintains a `Fundist\User\Credential` model and a `CredentialGenerator`/`CredentialManager`: the operator must **mint, store, and be able to reproduce a password for every player**, for the lifetime of that player. That is a standing secret-management obligation with no Hub88 counterpart, and it is why `User/Add` exists at all.

Artifact A's config also reveals a **login-prefix convention** (`prefix: 'GG'`, `delim: '_'`), i.e. operator player IDs are namespaced into Fundist's global login space as `GG_<id>`. *(spec, artifact B: login max 29 chars, alphanumeric plus `_-`; password min 6 chars and must not contain the login.)*

**Demo mode** is `Demo=1`. Artifact A additionally shows a **shared demo identity** — login `$DemoUser$`, password `Demo` — used instead of a real player. Contrast Hub88, where demo is expressed by `currency: "XXX"` *and* omitting both `token` and `user`.

**Response.** `User/AuthHTML` returns the CSV envelope: `1,<html>`. Artifact A splits on the first comma and returns the HTML remainder. This lines up exactly with the ARC platform's observed launch behaviour (`FINDINGS.md` §2.1): the server action returns `{ type: "HTML" | "URL", launchGameUrl }`, and the `HTML` variant is injected into a **Blob-URL iframe** — that is `User/AuthHTML`'s output, passed through.

Related session endpoints from artifact D: **`User/DirectAuth`** (same signature, presumably returning a redirect URL rather than HTML) and **`User/KillAuth`** (terminate a session). Neither is corroborated elsewhere; both are **[secondary — single source, D]**.

---

## 5. Catalogue feed — `Game/FullList` **[secondary — corroborated, A+B+D]**

One call, no pagination observed, returning five top-level collections (artifact B):

```json
{ "games": [...], "categories": [...], "merchants": [...],
  "merchantsCurrencies": [...], "countriesRestrictions": [...] }
```

Companion endpoints: **`Game/List`** (available games only), **`Game/Categories`**, **`Game/Sorting`** (`type=new|popular`, artifact B), **`Stats/GameDetails`** (per-game, by `gameId`), **`Tables/LobbyState`** and **`Providers/TablesInfo`** (live-dealer table state).

### 5.1 Per-game fields

Raw keys, from artifact B's field-by-field mapping, corroborated by artifact A:

| Field | Type / meaning |
|---|---|
| `ID` | Fundist global game ID |
| `Name` | **Map of locale → string** (`{"en": "...", ...}`) — localisation is in the feed |
| `Description` | Map of locale → string |
| `Image`, `ImageFullPath` | Thumbnail; `ImageFullPath` is absolute |
| `Url`, `MobileUrl` | Presence of each **is** the device-support signal — there is no `devices` enum |
| `PageCode`, `MobilePageCode` | The launch handle passed as `Page` |
| `MerchantID`, `SubMerchantID` | Studio, and sub-studio where a studio is resold |
| `CategoryID` | **Array** of category IDs |
| `RTP` | Return to player |
| `MinBetDefault`, `MaxBetDefault` | Default staking bounds |
| `MaxMultiplier` | Max win multiplier |
| `hasDemo` | Demo availability (0/1) |
| `Branded` | Operator-branded title flag |
| `IsVirtual` | Virtual-sports flag |
| `TableID` | Live-dealer table identifier |
| `AR` | Aspect ratio, for frame sizing |
| `Freeround` | Free-round eligibility/handle |
| `BonusBuy`, `Megaways`, `Freespins`, `FreeBonus` | **Mechanic flags** |

**Fundist's catalogue is richer than Hub88's on merchandising and staking** — `RTP`, `MinBetDefault`/`MaxBetDefault`, `MaxMultiplier`, `AR`, and the four mechanic flags have no Hub88 `/game/list` counterpart. `BonusBuy` in particular is directly actionable: report 02 §5.2 covers the UKGC's effective ban on bonus-buy mechanics since 2019, and this flag is how you'd filter for it without maintaining a manual title list.

### 5.2 Country restrictions are **per-merchant, not per-game**

This is the third structural difference, and it is a real capability gap. Artifact B's persisted model:

```ts
{ merchantId, restrictionId, name, bannedCountries: string[], bannedSubdivisions: string[] }
```

keyed **uniquely on `merchantId`**. Hub88 carries `blocked_countries` and `restricted_countries` **per game**, plus a per-game `certifications` array (`CURACAO`, `MGA`, `IOM`).

Consequences:

- Fundist geo-blocking is **studio-wide granularity by default**. You cannot express "this one title is unavailable in Country X" through `countriesRestrictions` — you block the whole studio or nothing.
- `bannedSubdivisions` is a capability Hub88 lacks — **sub-national** blocking, which matters for US states and Canadian provinces (report 02 §5.2: US markets are fully state-siloed).
- **No per-game certification field was found anywhere.** Hub88 publishes which regulator each title is certified under; nothing equivalent appears in any obtained Fundist artifact. If an operator needs per-jurisdiction certification status per title — and in a regulated market it does — that appears to come from outside the API. **[not obtainable]** — flag as an onboarding question.

Interestingly, the ARC platform **does** carry per-game `restrictedCountries` (§9), so either the operator derives them from the merchant-level feed, or a per-game source exists that these implementations don't use.

---

## 6. Reconciliation and reporting **[secondary — corroborated, B+D]**

| Endpoint | Parameters | Purpose |
|---|---|---|
| `Stats/Bets` | `DATE` | Per-bet transaction feed for a date |
| `Stats/BetsSummary` | `DATE`, `AffiliateID` | Aggregated betting summary |
| `Stats/Detailed` | `DATE`, `System`, `AffiliateID` | Detailed breakdown |
| `Stats/GameDetails` | `GAMEID` | Per-game statistics |

**Response field shapes for all four are [not obtainable].** Artifact D calls them but discards the payloads into a generic string return; artifact B's `getCasinoTransactions` reads the operator's *own* database, not Fundist. So we know the reconciliation feeds exist and how to authenticate to them, but **not what they return.**

The structural difference from Hub88 is nonetheless clear: Hub88's `/transactions/list` is **cursor-paginated and filterable**, with a documented per-record schema (`transaction_uuid`, `transaction_status`, `kind`, `round`, …). Fundist's `Stats/*` are **date-bucketed report pulls** — a batch/reporting idiom rather than a queryable ledger. For an operator that wants continuous automated reconciliation rather than a daily job, that is a meaningful difference in how you'd build the reconciliation worker.

Fundist's own back office additionally exposes `Reports/Turnover`, `Reports/BetsHistory`, `Reports/Games`, `Reports/MajorWins` and `Merchants/RefundReports` (artifact E), so the operator can reconcile through the UI regardless.

### 6.1 The response envelope is inconsistent — a real integration hazard

Artifact A's response base class has to handle **two mutually incompatible formats** and sniffs `Content-Type` to decide:

```php
private function isJson(): bool {
    return str_contains($this->getBaseResponse()->getHeaderLine('Content-Type'), 'application/json');
}
public function isOk(): bool {
    return $this->isJson()
        ? $this->getBaseResponse()->getStatusCode() === 200   // JSON endpoints
        : $this->getFundistCode() === 1;                      // CSV endpoints: "1,..." = OK
}
```

- **JSON** — `Game/FullList`, `Game/List`, `Game/Categories`
- **CSV `{code},{payload}`** — `User/Add` (success is the literal body `1`), `User/AuthHTML` (`1,<html>`), and errors generally (verified live: `34,Invalid Request`)

An HTTP 200 does not mean success on the CSV endpoints, and an HTTP 422 carries a meaningful application code in its body. Any client must handle both. **No enumeration of the numeric codes was obtainable** — code `1` = OK and code `34` = "Invalid Request" are the only two established, the latter by direct observation.

---

## 7. Free rounds, jackpots, tournaments, bonuses

**Free rounds — [secondary, partial].** Three signals, no API:
- The catalogue carries a per-game `Freeround` field (§5.1).
- Fundist's back office has a **`Users/Freerounds`** screen (artifact E) — so free rounds are granted **through the vendor's UI**, and plausibly through an API not seen here.
- At play time, free rounds surface on the wallet callback via **`game_extra` = `FREEROUNDS_XXX`** (artifact B's doc-transcribed comment), and a promotional credit arrives as `subtype: "promotion"` (artifact C handles exactly this).

**No free-round granting endpoint was found.** Searches for `Freerounds/Add`, `Freeround/Set`, `Users/FreeRounds` and similar as *API methods* returned nothing across all indexed public code. This is a sharp contrast with Hub88, which publishes a full **Freebets API** — seven endpoints covering create, bulk-create (up to 100 players), list, cancel, campaign templates and prepaids, with a documented `FRS_*` status lifecycle (report 02 §3.4). **[not obtainable]** whether Fundist has a programmatic equivalent at all.

Note also the mechanism difference: Hub88 flags a free bet **on the transaction itself** (`is_free: true` on the bet), so the operator's wallet knows not to debit real cash. Fundist appears to signal it via the **`game_extra` string** and `subtype`, which is a weaker, stringly-typed contract.

**Jackpots — [secondary, partial].** A jackpot payout is an ordinary `credit` carrying **`jackpot_win: 1`** (corroborated A+B). Fundist's back office has `Jackpots/JackpotList/JackpotBettingStatistics`. **No jackpot contribution or pool API was found** — same gap as report 02 found for Hub88, so this is not a Fundist-specific deficiency.

**Tournaments — [not obtainable].** No tournament endpoint appears in any artifact. The ARC platform has a full tournaments module and its launch route carries an optional `tournamentId` segment (`FINDINGS.md` §2.1), which suggests **tournaments are implemented operator-side**, over the top of ordinary game rounds, rather than by the aggregator. That is a reasonable inference, not an established fact.

**Loyalty — [secondary — single source, D].** `Loyalty/Payment` (`System`, `Amount`, `Login`) credits a loyalty payout, and `Loyalty/Levels` exists in the back office. There is also a `WLCInfo` method used for bonus select/cancel in artifact D, whose semantics are unclear.

---

## 8. Commercial and onboarding requirements **[primary for the quoted wording; low confidence on all numbers]**

From SoftGamings' own pages:

- **Integration timeline.** [softgamings.com/casino-api](https://www.softgamings.com/casino-api/): *"A typical casino games API integration takes from 3 days to a week."* The [FAQ](https://www.softgamings.com/faq/) is more hedged: *"The fastest turnaround time is 48 hours, but you need to contact our team for specific details and exact TAT."*
- **Prerequisites — this is the useful one.** The FAQ states an operator needs *"a globally acclaimed gaming license, a registered gambling company with its address domain and bank accounts, payment gateways, and certified software providers."* In other words: **licence and PSP before integration, not after.** This corroborates report 02's and `BUILD-ESTIMATE.md`'s conclusion that the licensing calendar, not the integration, is the critical path.
- **Pricing.** *"SoftGamings offers transparent pricing through revenue share or a fixed monthly fee."* **No rates are published.** Quotes within 24 hours on request.
- **Catalogue scale.** SoftGamings markets 10,000+ games from 250+ providers on the casino-API page; other 2026 material claims 16,000+ from 300+. Vendor-marketed, unaudited, and mutually inconsistent — same caveat as report 02's aggregator table.
- **Five-phase onboarding**: Interface Setup → Testing & Go-Live → Integration Monitoring → Update Handling → Issue Response. A sandbox is provided (`apitest.fundist.org`, confirmed live).
- **Technical certification of the operator's integration:** no published requirement was found. **[not obtainable]**

**Minimum guarantees: [not obtainable].** Nothing specific to SoftGamings. The €5,000–€25,000/month range that circulates in comparison-site material is the same generic industry figure already flagged as very-low-confidence in report 02 §4, not a SoftGamings quote. Setup-fee figures found in secondary comparison articles (~$15,000 entry tier, $30,000–$80,000 typical) are third-party estimates about SoftGamings' *turnkey/white-label* products, not its games API, and should not be used. Per `NEXT-STEPS.md` §7, none of these belong in a financial model without a live quote.

---

## 9. What the ARC sandbox itself proves **[primary — this investigation's own artifacts]**

The retail bundle contains real catalogue payloads, and they let us watch the Fundist feed land in an operator's model.

**A refinement to an earlier finding.** `FINDINGS.md` §5 notes that "Fundist"/"SoftGamings" appears **nowhere in the admin bundle**. That is confirmed — zero hits across `src-analysis/admin/`. But the strings do appear in the **retail** bundle, in two different roles, and the distinction matters:

**(a) As pure data — the good case.** The aggregator is a plain row with a localised display name:

```json
"casinoAggregator": { "id": "1",
  "name": { "EN": "SoftGamings (Fundist)", "DE": "SoftGamings (Fundist)", … },
  "isActive": true }
```

Eleven locales, all identical, for a vendor's proper noun. The aggregator is modelled as a **translatable label on a lookup table** — no behaviour, no code path. This is exactly the shape you want.

**(b) Baked into natural keys — the leak.** Game and category identifiers carry the aggregator's name and Fundist's own numbering:

```json
"uniqueId": "SOFTGAMING-810-2834532-madbuffalo"
"uniqueId": "SOFTGAMING-CAT-1364"
```

Decomposing the game key against the contract in §5, and testing it across the catalogue: **`810` is RAWGames (26 of 27 titles) and `817` is AvatarUX (1 title)** — a stable per-provider constant, i.e. Fundist's **`MerchantID`**. The third segment (`2834532`, and a run of siblings at `2834492`, `2834496`, `2834500`… spaced by 4) is Fundist's global **game `ID`**. So the operator's primary catalogue key is `{AGGREGATOR}-{MerchantID}-{GameID}-{slug}` — **aggregator-scoped by construction.** The 31 distinct `SOFTGAMING-CAT-*` category IDs (`4`, `41`, `54`, `1364`, `3604`, `7239`, …) are likewise Fundist's own `Game/Categories` numbering, wearing a prefix.

**The full field mapping, feed → operator model**, from a real record (Mad Buffalo):

| ARC field | Value | Fundist source (§5.1) |
|---|---|---|
| `uniqueId` | `SOFTGAMING-810-2834532-madbuffalo` | `MerchantID` + `ID` |
| `returnToPlayer` | `95.57` | `RTP` |
| `demoAvailable` | `true` | `hasDemo` |
| `iconUrl` / `mobileImageUrl` / `desktopImageUrl` | `agstatic.com/games/rawgames/mad_buffalo.jpg` | `ImageFullPath` — **still served from the vendor's CDN** |
| `restrictedCountries` | `["AU","DK","SE","GB","US"]` | `countriesRestrictions`, flattened from merchant to game |
| `casinoProvider` | RAWGames (id 19) | `merchants[].Alias` |
| `devices` | `null` | would derive from `Url`/`MobileUrl` |
| `name` | 11 locales, all `"Mad Buffalo"` | `Name` locale map, **collapsed** — Fundist's per-locale names were not used |

Two things stand out. The operator **re-projects Fundist's merchant-level country restrictions down to per-game** — which is why ARC can show per-game `restrictedCountries` despite §5.2. And game images are **hot-linked to `agstatic.com`**, the vendor's CDN: a live runtime dependency on the aggregator for every lobby tile, beyond the API itself.

---

## 10. Diff table: Fundist vs. Hub88

The main deliverable. Hub88 column per report 02 §3; Fundist column per §2–§7 above.

| Contract area | Hub88 | Fundist | Same / Different / Unknown |
|---|---|---|---|
| **Outbound auth** (operator → vendor) | RSA-SHA256, asymmetric, operator holds private key | MD5 of positional slash-joined string; shared `API_PASSWORD`; operator IP hashed in | **Different — materially weaker.** No non-repudiation, broken hash primitive, home-rolled canonicalisation |
| **Inbound auth** (vendor → operator) | RSA-SHA256, `X-Hub88-Signature` header, over exact body bytes | HMAC-SHA256 over key-sorted concatenated **values**, key = `sha256_raw(secret)`; in the `hmac` **body field** | **Different.** Symmetric not asymmetric; body field not header; no delimiters between values |
| **Response signing** | Not required | **Required** — operator HMACs its own response | **Different — Fundist stricter (better)** |
| **Callback routing** | 5 distinct URL paths | **1 URL**, discriminated by `type` body field | **Different** |
| **Callback operations** | `user/info`, `user/balance`, `transaction/bet`, `transaction/win`, `transaction/rollback` | `ping`, `balance`, `debit`, `credit`, `roundinfo` | **Different.** Fundist adds `ping` + `roundinfo`; has no `user/info`; **has no rollback operation** |
| **Rollback** | Dedicated endpoint + `reference_transaction_uuid` | **`i_rollback` field on an ordinary debit/credit, with inverted sign** | **Different — highest-risk difference.** See §3.4 |
| **Money representation** | **Int64 scaled ×100,000** (`$3.56` → `356000`) | **Decimal strings**, 2 dp (`"3.56"`) | **Different.** Fundist reintroduces the float/rounding risk the scaled-integer convention exists to remove |
| **Idempotency key** | `transaction_uuid` | `tid` (alphanum, ≤32) — plus replay-detect on full request `hmac` | **Same concept, different name** |
| **Round grouping** | `round` + `round_closed` | `i_gameid` (round) + `round_start` / `round_ended` | **Same concept.** Fundist adds an explicit round-*start* marker and a `roundinfo` summary callback |
| **Per-request trace ID** | `request_uuid`, separate from business txn | Not present on inbound; `TID` on outbound only | **Different — Fundist lacks it** |
| **Error signalling** | Enum of 14 `RS_ERROR_*` codes, documented | Free-text `error` string, HTTP 200, **no documented enum** | **Different — significantly worse.** Each implementation invents its own strings |
| **Retry semantics** | Documented (3× @1s; rollback ≤500× backoff) | **Not documented anywhere obtainable** | **Unknown — must ask** |
| **Record retention** | 4 months, both sides | Not documented | **Unknown — must ask** |
| **Launch call** | `POST /game/url` → JSON with launch URL | `GET User/AuthHTML` → CSV `1,<html>` | **Different.** Returns a rendered frame, not a URL |
| **Launch credential** | Operator-minted session `token`, short-lived, operator-validated | **Persistent Fundist `Login` + `Password` per player** | **Different — largest structural gap.** Operator must mint/store/reproduce a password per player forever |
| **Player provisioning** | None — `user` is an opaque string | **`User/Add` required**; also `User/Update`, `User/Enable`, `User/KillAuth` | **Different.** Fundist keeps its own player record; every player is shadow-registered |
| **Demo mode** | `currency: "XXX"`, omit `token`/`user` | `Demo=1`; plus a shared `$DemoUser$` identity | **Different mechanism, same outcome** |
| **Correlation handle** | `game_code` + operator's own `token` | **`ExtParam` at launch → echoed as `i_extparam` on every callback** | **Different — Fundist better.** A first-class round-trip correlation slot |
| **Catalogue call** | `/game/list` | `Game/FullList` (+ `Game/List`, `Game/Categories`, `Game/Sorting`) | **Same concept** |
| **Catalogue: localisation** | Not in feed | `Name`/`Description` as **locale maps** | **Different — Fundist richer** |
| **Catalogue: RTP & staking** | Not exposed | `RTP`, `MinBetDefault`, `MaxBetDefault`, `MaxMultiplier` | **Different — Fundist richer** |
| **Catalogue: mechanic flags** | Not exposed | `BonusBuy`, `Megaways`, `Freespins`, `FreeBonus`, `Branded`, `IsVirtual` | **Different — Fundist richer.** `BonusBuy` is directly useful for UKGC filtering |
| **Catalogue: device support** | `platform` enum at launch | Implied by presence of `Url` / `MobileUrl` | **Different encoding, same information** |
| **Geo-restriction granularity** | **Per game** (`blocked_countries`, `restricted_countries`) | **Per merchant** (`countriesRestrictions` keyed on `merchantId`) + `bannedSubdivisions` | **Different — Fundist coarser on games, finer on geography.** Sub-national blocking is a Fundist-only capability |
| **Per-game certification** | `certifications[]` (`CURACAO`, `MGA`, `IOM`) | **Not found in any artifact** | **Unknown — likely absent.** Real compliance gap if so |
| **Free rounds** | Full **Freebets API**: 7 endpoints, bulk grant, campaigns, `FRS_*` lifecycle; `is_free` on the bet | **No API found.** Back-office screen only; surfaces as `game_extra=FREEROUNDS_XXX` + `subtype: "promotion"` | **Unknown/likely absent — biggest functional gap** |
| **Jackpots** | `phoenix_jackpot_support` catalogue flag; no contribution API | `jackpot_win: 1` on credit; no contribution API | **Same limitation on both sides** |
| **Tournaments** | Not documented | Not found; ARC appears to implement operator-side | **Unknown on both** |
| **Reconciliation feed** | `/transactions/list` — paginated, filterable, documented per-record schema | `Stats/Bets`, `Stats/BetsSummary`, `Stats/Detailed` — **date-bucketed reports**, schema unknown | **Different idiom.** Query-a-ledger vs. pull-a-report |
| **Response format** | JSON throughout | **Mixed** — JSON for catalogue, CSV `{code},{payload}` for auth/launch/errors | **Different — worse.** Two parsers required |
| **Live-dealer support** | Not specifically documented | `Tables/LobbyState`, `Providers/TablesInfo`, per-game `TableID` | **Different — Fundist explicit** |
| **Vendor scope** | Aggregator only | **Aggregator + full PAM** (players, payments, KYC, affiliates, loyalty, reporting) | **Different — the deepest difference of all** |
| **Public documentation** | Fully public at `docs.hub88.io` | **None.** All hosts NXDOMAIN or gated behind login+2FA | **Different** |

---

## 11. What a second aggregator would actually cost

The prior finding — *"the aggregator is pure data, not hardcoded, which is the right way to build it"* — is **correct but incomplete**, and §9 shows where. The admin models the aggregator as data. The **catalogue's natural keys do not**: `SOFTGAMING-810-2834532-madbuffalo` and `SOFTGAMING-CAT-1364` embed the vendor's name, the vendor's merchant numbering, and the vendor's game and category IDs into identifiers that are almost certainly referenced by bonus eligibility rules, wagering-contribution templates, tournament game lists, and category memberships.

That is not fatal — prefixing by aggregator is a *defensible* way to keep a global key space collision-free across vendors, and it is far better than assuming one vendor's IDs are globally unique. But it does mean the aggregator identity is load-bearing in the data model, not merely a label.

### Is a clean adapter interface achievable?

**Yes for the catalogue. Yes with care for launch. Only partly for the wallet.** Taking the §10 diff seriously:

**Cleanly abstractable** — these are pure shape differences over the same concepts:
- Catalogue ingestion. Field names and richness differ, but every aggregator emits "games with providers, categories, images, device support, geo rules". Normalise on ingest into your own schema, keep the vendor's raw record alongside, and the differences never leak further. Fundist's extra fields (`RTP`, staking bounds, mechanic flags) are a superset — model them as nullable, populated where the vendor supplies them.
- Money representation. Int64-×100,000 vs. decimal-string is an adapter-boundary conversion, provided your ledger uses one internal representation. Use scaled integers or a decimal type internally and convert at the edge — never let a vendor's float reach the ledger.
- Reporting/reconciliation. Query-a-ledger vs. pull-a-report is a difference in the *worker*, not the domain. Both produce transaction records to diff against your own.
- Signing. Every vendor has its own scheme; that is precisely what an adapter is for. One `sign(request)` / `verify(callback)` pair per vendor.

**Abstractable only with a deliberately-chosen interface:**
- **Rollback semantics (§3.4).** If your internal wallet interface is `{bet, win, rollback}` — the Hub88 shape, and the natural one — then the Fundist adapter must **detect `i_rollback` before dispatching on `type`, and invert the sign**. This is the one place where a naive adapter silently corrupts balances rather than failing loudly. Make the internal operation set explicit about direction (`credit(amount)` / `debit(amount)` plus a separate `reverses: txn_id`), and write the test that feeds a `type:"credit"` + `i_rollback` payload through and asserts the player's balance *decreases*.
- **Round lifecycle.** `round_closed` vs. `round_start`/`round_ended` + a `roundinfo` callback. Model rounds internally with explicit open/close events and let Fundist's extra `roundinfo` be an optional enrichment.
- **Error taxonomy.** You need an internal enum (insufficient funds, unknown player, bad signature, currency mismatch, duplicate transaction). Hub88 maps onto it directly. Fundist requires you to *choose* the strings you emit, and to confirm with SoftGamings which — if any — it interprets. Until that is answered, this is the adapter's least-defined edge.

**Not abstractable — these force operator-side changes, not just adapter code:**
- **Player provisioning.** Fundist requires `User/Add` and a persistent per-player `Login`+`Password`; Hub88 requires nothing. You cannot hide "this vendor needs every one of your players mirrored into its system, with a credential you must be able to reproduce" behind a game-supply interface. It touches registration, credential storage, and account-state sync (`User/Enable`/`KillAuth`). Budget it as a **vendor-specific lifecycle hook** invoked at player registration, not as part of the game adapter.
- **Geo-restriction granularity.** Merchant-level (Fundist) vs. game-level (Hub88) is an *expressiveness* difference. Your model must be per-game (the finer granularity) and the Fundist adapter must fan merchant restrictions out across that merchant's games — which is exactly what ARC does (§9). That is a lossy direction, so it works; the reverse would not.
- **Per-game certification.** If Fundist genuinely does not supply it and Hub88 does, no adapter invents it. In a regulated market this becomes a manual compliance dataset the operator maintains.
- **Free rounds.** Hub88 has a rich programmatic API; Fundist appears not to. An operator-side bonus engine that assumes it can grant free rounds via API will simply not work on Fundist without a manual back-office step. **Resolve this before designing the bonus engine's free-round feature** — it is the difference between a supported flow and a human in the loop.

### Bottom line

A clean adapter interface **is** achievable for the game-supply layer proper — catalogue, launch, and wallet — and is worth building. The seam should be drawn to include catalogue normalisation, launch-parameter construction, signing/verification, and wallet-callback translation.

It should **not** be drawn to include player provisioning, which is a genuine per-vendor lifecycle concern, nor free-round granting, whose availability differs in kind rather than in shape.

The realistic cost of adding a second aggregator, given a first integration built this way: **the adapter itself is the small part.** The expensive parts are (1) whatever the second vendor demands that the first did not — player mirroring being the canonical example, (2) backfilling capability gaps the first vendor hid (per-game certification, free-round APIs), and (3) the commercial and certification calendar, which as report 02 and §8 both conclude, dominates the engineering timeline regardless.

---

## 12. Open questions to put to SoftGamings

Ranked by how much they change the build. Every one is a documented gap above, not speculation.

1. **The specification itself** — request "Fundist API OneWallet" (v128 or later) plus the games/reporting API reference under NDA. Everything below is in it.
2. **Retry and timeout policy** on OneWallet callbacks — how many attempts, what backoff, what timeout? Determines idempotency-layer design (§3.5).
3. **Error contract** — is there an enumerated error code set, or is `error` free text? Does Fundist distinguish "insufficient funds" from "operator down"? (§3.3)
4. **`i_rollback` vs. `subtype: "cancel"`** — what is the precise difference, and when is each sent? (§3.4)
5. **Free rounds** — is there a programmatic grant/cancel/query API, or is `Users/Freerounds` back-office-only? (§7)
6. **Per-game certification** — how does an operator determine which jurisdictions a given title is certified for? (§5.2)
7. **Per-game geo-restriction** — is merchant-level truly the only granularity? (§5.2)
8. **`Stats/*` response schemas**, and whether a queryable/paginated transaction endpoint exists alongside the date-bucketed reports. (§6)
9. **Record retention** — minimum period each side stores transaction records. (§3.5)
10. **`mrid`, `UniversalLaunch`, `WLCInfo`** — undocumented parameters/methods seen in implementations. (§3.2, §4, §7)
11. **Commercials** — revenue share, minimum guarantee, setup fee, ramp-up terms. Nothing public exists. (§8)
12. **Operator certification** — is there a technical certification step before go-live?

---

## Sources

**Primary — Fundist / SoftGamings owned, verified directly in this session:**
- `https://api.fundist.org/` — production API host (178.16.18.162); returns `34,Invalid Request` with HTTP 422, establishing the CSV error envelope
- `https://apitest.fundist.org/` — sandbox host (CloudFront alias `d1g0zr2udzjr0t.cloudfront.net`)
- [`https://www.fundist.org/`](https://www.fundist.org/) — operator back-office login (username/password + two-factor), "©2008–2026, Copyright Fundist.org"
- `https://www.fundist.org/en/Docs`, `https://www.fundist.org/en/API` — both serve the login shell; **documentation is gated**
- `docs.fundist.org`, `apidoc.fundist.org`, `wiki.fundist.org`, `docs.softgamings.com` — **NXDOMAIN / unreachable**
- [SoftGamings — Online Casino API](https://www.softgamings.com/casino-api/) — integration phases, 3-day-to-a-week timeline, "revenue share or a fixed monthly fee"
- [SoftGamings — FAQs](https://www.softgamings.com/faq/) — 48-hour fastest TAT; pre-integration prerequisites (licence, company, PSP, certified providers)
- [SoftGamings — Products](https://www.softgamings.com/products/)

**Primary — this investigation's own captured artifacts:**
- `src-analysis/retail/pages/casino.html` — live ARC catalogue payload: aggregator record, 27 game `uniqueId`s, 31 category IDs, per-game RTP and restricted countries
- `src-analysis/retail/index.html` and sibling pages — `SOFTGAMING-CAT-*` taxonomy
- `src-analysis/admin/**` — **zero** occurrences of "fundist"/"softgamings", confirming `FINDINGS.md` §5

**Secondary — independent operator-side implementations (basis for §2–§7):**
- [`qsmartisp/casino-backend`](https://github.com/qsmartisp/casino-backend) — PHP/Laravel. `app/Services/Fundist/{FundistService,Sender,Responder,Response,WebhookService,WebhookDtoStringify}.php`, `app/Services/Fundist/DTO/Webhook/*.php`, `app/Http/Controllers/Webhook/OneWalletController.php`, `app/Http/Requests/Fundist/WebhookRequest.php`, `app/Console/Commands/Fundist/GameUpdate.php`, `config/fundist.php`
- [`ansul-rathi/unifitz-api-main`](https://github.com/ansul-rathi/unifitz-api-main) — TypeScript. `src/interfaces/fundist.interface.ts` (doc-transcribed field constraints), `src/services/fundist.service.ts`, `src/services/fundist-onewallet.service.ts`, `src/models/fundist-game.model.ts`, `src/models/fundist-restricted-country.model.ts`
- [`levi-tech-guru/casineo-main-api-v2`](https://github.com/levi-tech-guru/casineo-main-api-v2) — TypeScript. `src/utils/softgames/softgames.ts`, `src/utils/softgames/callbacks/{ping,balance,debit,credit,rollback,cancel,promotions,roundinfo,invalidType}.ts`, `src/utils/gameHelpers.ts`, `src/api/v1/controllers/games/softgame/softgameCallback.controller.ts`
- [`stoura90/fengshuistone`](https://github.com/stoura90/fengshuistone) — C#. `Labixa/Common/SDKApiFundist.cs` (19 API methods), `Labixa/HashMD/HashMD5.cs` (per-method pre-hash strings, doc-transcribed)
- [`WebRabbits/checkboxAddDataFunMerchant`](https://github.com/WebRabbits/checkboxAddDataFunMerchant) — `index4.html`, a saved copy of the Fundist back-office UI; source of the 89-route map in §1

*Credential values are present in several of these repositories' committed configuration files. They are deliberately not reproduced here, and were not used to make any authenticated request.*

**Indexed but unobtainable:**
- ["Fundist API OneWallet v128" (Scribd)](https://www.scribd.com/document/740119351/Fundist-API-OneWallet-v128) — **HTTP 410 Gone**. Establishes that the official spec is titled *Fundist API OneWallet* and had reached v128, but no content was retrieved. No mirror was found on pdfcoffee, vdocuments, idoc or dokumen.
- [Fundist.org Postman workspace](https://www.postman.com/cloudy-flare-16729/workspace/fundist-org/overview) — renders as an empty SPA shell to non-interactive fetch; no collection data retrieved.
- `http://docs.dev.bettings.ch/` — surfaced in searches for Fundist docs; on inspection its SPA bundle points at `dev.api.bettings.ch/api/documentation/{admin,frontend}` and contains **zero** references to Fundist. Unrelated platform, discarded.

**Cross-reference:**
- [`research/02-game-provider-integration.md`](./02-game-provider-integration.md) — the Hub88 baseline this report diffs against
- [`FINDINGS.md`](../FINDINGS.md) §2.1 (game-launch flow), §5 (aggregator-as-data)
- [`NEXT-STEPS.md`](../NEXT-STEPS.md) §5.3 (this task), §7 (confidence conventions)
