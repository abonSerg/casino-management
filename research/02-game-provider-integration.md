# Online Casino Game-Supply Chain: Technical & Commercial Integration Reference

**Purpose:** Reference document for engineering/product decisions about building or integrating with a casino platform's game-supply layer. Covers the supply chain, direct vs. aggregator integration trade-offs, the seamless-wallet technical contract, commercials, and certification/compliance.

**Confidence note (read first):** The single strongest primary source obtained during this research is **Hub88's public developer documentation** (`docs.hub88.io`), which is unusually open for the industry — most aggregators and studios (SoftSwiss, Slotegrator, Pariplay, EveryMatrix, Pragmatic Play, Evolution, Relax Gaming, etc.) gate their actual API references behind signed partner agreements. Where this document quotes endpoint names, field names, or status codes, they are **verbatim/paraphrased from Hub88's real public docs** unless explicitly marked otherwise. Because seamless-wallet APIs across the industry converge on the same conceptual pattern (this is confirmed by every secondary source consulted), Hub88's documented contract is used as a **representative concrete example of the pattern**, not a claim that other vendors use identical field names. Commercial figures (revenue share %, minimum guarantees, integration fees) are almost universally confidential; the numbers below are drawn from industry blogs, consultancy sites, and comparison articles, and are explicitly flagged as estimates/ranges, not verified contract terms.

---

## 1. The supply chain

The online casino content chain has four layers. Each exists to solve a specific problem the layer below it can't solve alone.

```
Game Studio  →  Aggregator  →  Platform / PAM  →  Operator (casino brand)
(RNG/RGS,       (single API,     (player accounts,   (brand, license,
 game math,      contract,       wallet, KYC,         marketing,
 certification)  settlement)     bonusing, CMS)       player relationship)
```

### 1.1 Game studio
Companies like **Pragmatic Play, Evolution, NetEnt (Evolution-owned), Play'n GO, Hacksaw Gaming, Nolimit City, Endorphina, BGaming, Spribe, Relax Gaming, Yggdrasil, Booming Games, Onlyplay** build the actual game — the math model, the RNG (or, for live casino, the physical studio and dealers/game presenters), the client (HTML5 game frame), and the **Remote Game Server (RGS)** that runs the round logic and holds the certified RNG. The studio owns:
- The RNG/math and its regulatory certification (GLI-19, per-jurisdiction RTP variants).
- The RGS that executes each game round.
- The game client/frontend served into the operator's page via iframe or SDK.

Studios do **not** hold player funds — they call out to the operator's (or aggregator's) wallet to debit/credit in real time. This is the single most important architectural fact in the whole chain (see §3).

### 1.2 Aggregator
Aggregators — **SoftSwiss Game Aggregator, Slotegrator (APIgrator), Pariplay (Fusion), EveryMatrix (CasinoEngine), Hub88, GoldenRace, SoftGamings, Bragg (via Salsa Technology's marketplace), Relax Gaming (Silver Bullet), Yggdrasil (YGS Masters/GATI), Booming Games, Onlyplay** and others — sit between many studios and many operators. Why they exist:

- **Single integration point.** Instead of an operator building and maintaining N separate bespoke wallet integrations (one per studio, each with its own field names, retry semantics, and quirks), the operator builds **one** seamless-wallet callback API to the aggregator's spec, and the aggregator fans that out to every studio behind it. Hub88, for example, exposes ~12,000+ games from 150+ studios through one Operator API ([Hub88 casino API integration](https://hub88.io/casino-api-integration/); [Hub88 docs](https://docs.hub88.io/)).
- **Single commercial contract.** One master agreement and one settlement relationship instead of dozens of studio contracts, each with its own revenue share, minimum guarantee, and legal review.
- **Single wallet/session model.** The aggregator normalizes every studio's game-launch and transaction pattern into one consistent request/response contract (see §3), even though each studio's own RGS behaves differently internally.
- **Certification pass-through.** Aggregators bundle jurisdictional certifications for the games they carry (Pariplay's Fusion claims certification/compliance "in over 25 regulated markets" — [iGB](https://igamingbusiness.com/gaming/pariplay-fusion-aggregation-platform-transforms-gaming/)), sparing the operator from chasing per-studio certification status manually.
- **Reporting/reconciliation normalization.** One transaction feed format and one GGR reconciliation process instead of N different ones.

Some aggregators are themselves studios that opened their pipe to third parties: Relax Gaming's **Silver Bullet** (50+ partner studios, commercial + compliance representation) and **Powered By Relax** (technical-only) programs, and Yggdrasil's **YGS Masters** program (third-party studios build on Yggdrasil's **GATI** technical framework and get access to Yggdrasil's certifications and operator network — [CasinoBeats](https://casinobeats.com/2018/12/05/yggdrasil-ygs-masters-from-concept-to-reality/), [goodluckmate.com](https://goodluckmate.com/blog/all-you-need-to-know-about-the-yggdrasil-ygs-masters-program)) are the clearest examples of this hybrid studio/aggregator role.

### 1.3 Platform / PAM (Player Account Management)
The platform (sometimes the operator's own stack, sometimes a turnkey/white-label vendor like EveryMatrix, SoftSwiss, or a PAM specialist) owns:
- The **player wallet of record** — actual real-money balance, currency, bonus balance.
- KYC/AML, responsible-gambling controls, deposit/withdrawal/payments.
- The seamless-wallet callback endpoints the aggregator/studio calls into (§3).
- CMS/lobby, bonusing engine, CRM/segmentation hooks.

In many real deployments the "platform" and "aggregator" are the same vendor (e.g., SoftSwiss and EveryMatrix sell a combined PAM + aggregator product), but architecturally they are distinct concerns: the aggregator is a *content distribution and transaction-routing layer*; the PAM is the *system of record for player funds*.

### 1.4 Operator
The licensed casino brand — holds the gambling license(s), owns the player relationship, marketing, and ultimately bears the regulatory and commercial risk. The operator chooses which aggregator(s)/studios to integrate, decides its jurisdictional footprint, and is the party regulators hold accountable even though the actual game math executes on a studio's RGS.

### Named aggregator comparison

| Aggregator | Approx. content scale (as marketed) | Notable strengths | Source |
|---|---|---|---|
| SoftSwiss Game Aggregator | 40,000+ games, 300+ providers | Crypto-friendly, high processing volume, EU regulation focus | [softswiss.com](https://www.softswiss.com/game-aggregator/), [onlyigaming.com](https://onlyigaming.com/categories/game-aggregators) |
| Slotegrator (APIgrator) | 20,000–30,000+ games, 150–180 providers | Fast (claimed same-day) integration, strength in CIS/Africa/Asia markets | [slotegrator.pro](https://slotegrator.pro/apigrator.html) |
| Hub88 | 12,000+ games, 150+ suppliers | Publicly documented API, fastest go-live claims (as little as ~3 days, typical ~2 weeks), transparent docs (unusual for the sector) | [docs.hub88.io](https://docs.hub88.io/), [hub88.io](https://hub88.io/casino-api-integration/) |
| Pariplay (Fusion) | 14,000+ games, 120+ suppliers | Certified in 25+ regulated markets (UK, Malta, Romania, Ontario, several US states) | [iGB](https://igamingbusiness.com/gaming/pariplay-fusion-aggregation-platform-transforms-gaming/) |
| EveryMatrix (CasinoEngine) | 27,800+ games, 317 suppliers (largest claimed catalogue) | Modular, API-driven; also sells full PAM/turnkey stack; live-table statistics API | [everymatrix.com](https://everymatrix.com/casinoengine/) |
| SoftGamings | 10,000+ games, 250+ providers | Broad aggregation + consulting/turnkey services | [softgamings.com](https://www.softgamings.com/casino-api/) |
| Relax Gaming (Silver Bullet / Powered By) | 2,000+ games via one integration, 50+ partner studios | Studio-run aggregator; Silver Bullet adds commercial/compliance representation for smaller studios | [relax-gaming.com](https://www.relax-gaming.com/partners/silverbullet) |
| Yggdrasil (YGS Masters) | Yggdrasil catalogue + ~20 partner studios (as of the most recent addition found) | Studio-run aggregator via the GATI technical framework | [CasinoBeats](https://casinobeats.com/2018/12/05/yggdrasil-ygs-masters-from-concept-to-reality/), [igamingtoday.com](https://www.igamingtoday.com/yggdrasil-adds-lakka-studios-as-20th-ygg-masters-partner-expanding-nordic-led-content-pipeline/) |
| GoldenRace | Virtual sports/number games specialist | Distributed through multiple aggregators (SoftGamings, Digitain, BtoBet, EveryMatrix, SBTech) rather than being a general slots aggregator itself | [softgamings.com](https://www.softgamings.com/online-gambling-software-providers/golden-race/) |

*Game/provider counts are vendor-marketed figures from each company's own marketing pages and should be treated as directional, not audited.*

---

## 2. Direct integration vs. aggregator

| Dimension | Direct studio integration | Aggregator integration |
|---|---|---|
| Engineering time per content source | Est. **6–10 weeks per studio** for a mid-size operator running many direct integrations (engineering + QA + legal/cert overhead), per industry commentary — **this is an estimate from a secondary source, not a verified benchmark** ([track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026)) | One integration covers the whole catalogue behind the aggregator; Hub88 reports most operators go live in **~2 weeks** from API-key generation, with some live in **under 3 days** ([hub88.io](https://hub88.io/casino-api-integration/)) |
| Time to full catalogue | Months, scaling roughly linearly with number of studios contracted | Days once one integration is certified — claims of 3–7 days to reach 10,000+ games via an aggregator appear repeatedly in vendor/industry material ([track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026)) |
| Ongoing maintenance | Each studio evolves its own API independently — N maintenance surfaces, N sets of breaking changes, N support relationships | One maintenance surface; aggregator absorbs studio-side API churn (mostly) behind a stable operator-facing contract |
| Certification burden | Operator/studio must certify per jurisdiction studio-by-studio | Aggregator often carries pre-existing certifications for its catalogue in major regulated markets, though the operator's own license/market entry is still separate |
| Commercial leverage | Direct relationship = ability to negotiate exclusive content, dedicated/branded tables, custom RTP configs, and potentially better rates at scale — but only studio-by-studio | Aggregator negotiates on behalf of many operators; individual operator has little leverage over per-studio terms, and the aggregator adds its own margin on top |
| Effective content cost | Studio's headline revenue share only | Studio's revenue share **plus** an aggregator margin layered on top — commonly cited as **1–4 percentage points** or bundled into a **5–15% of GGR** all-in aggregator rate depending on source (see §4) |
| Minimum guarantees | Negotiated per studio directly; can be substantial for a high-volume operator wanting a top-tier studio | Aggregator often passes through the underlying studio's minimum guarantee, sometimes without making that pass-through obvious upfront ([henkwolff.com](https://henkwolff.com/insights/choosing-game-aggregator/) as summarized via search) |
| When direct makes sense | Very large operator, single dominant market, wants bespoke/branded/dedicated-table deals, has the legal/engineering bandwidth to run many contracts and integrations in parallel | — |
| When aggregator makes sense | New operator, multi-market/multi-jurisdiction ambitions, wants breadth of catalogue fast, limited engineering headcount to spend on wallet-integration maintenance | — |

**Cost claim, flagged as an estimate:** one industry source states aggregation can cut integration cost "by up to 70%" versus direct multi-studio contracting ([track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026); [groovetech.com](https://www.groovetech.com/blog/benefits-of-game-aggregators-for-online-casino-operators)). No primary/audited source was found for this figure — treat as a marketing-adjacent industry estimate, directionally plausible but not verifiable.

The core trade-off in one sentence: **you pay a thicker margin per unit of GGR through an aggregator, in exchange for dramatically less engineering/legal/certification overhead and much faster time-to-catalogue.** For a platform team building from scratch, aggregator-first is the default industry pattern; direct studio deals typically come later, once volume justifies the negotiation and maintenance cost for a specific must-have studio (most commonly Evolution for live casino, or Pragmatic Play for slots, given their market weight).

---

## 3. The technical integration contract

### 3.1 Seamless wallet vs. transfer wallet

Two wallet models exist, and the choice has real product and engineering consequences.

**Seamless (real-time) wallet.** The player's balance never physically moves. The studio's RGS calls the operator's (or aggregator's) wallet API in real time for every bet and every win: debit on bet, credit on win, against the operator's live wallet. There is no separate "game session balance" — what the player sees in the game client is the same number as their site-wide account balance, kept in sync transaction-by-transaction.

- **Pros:** No stranded funds in a game session; player can move between games/sportsbook/site instantly with one balance; bonus wallet and cash wallet segregation stays authoritative on the operator's side; simpler AML/RG (responsible gambling) enforcement because every wager is visible to the operator's systems the instant it happens.
- **Cons:** The operator's backend must be reachable, fast, and correct 24/7 for every single spin/hand across every integrated studio — it becomes a hard real-time dependency in the critical path of gameplay. Outages or slow responses directly degrade player experience and can cause studios to pause traffic to the operator.

**Transfer wallet.** Funds are moved from the main operator wallet into a separate game/session-specific wallet (held by the aggregator or studio) before play begins, and moved back out when the session ends.

- **Pros:** Decouples the operator's core wallet from the real-time availability requirements of every studio's RGS; simpler for operators without the engineering capacity to expose a robust real-time seamless API; useful stop-gap for early-stage or resource-constrained operators.
- **Cons:** Player experience friction (explicit "load funds into game" step); risk of stuck/orphaned balances in the game wallet if a session terminates abnormally; harder unified view of player balance across products; generally regarded as the legacy/fallback model — most modern regulated-market operators run seamless.

Hub88's docs describe both explicitly: "Seamless Wallet" (operator handles live balance updates through wallet API calls) vs. "Transfer Wallet" (Hub88 handles all balance updates on its own side after transfer) ([docs.hub88.io – Demo and Real Gameplay](https://docs.hub88.io/developer-docs/operator-api-reference/operator-api-overview/demo-and-real-gameplay.md)). This binary is standard across the industry — SoftSwiss, Slotegrator and others describe the same two-model split in their own marketing copy, though their gated technical docs could not be accessed directly to confirm field-level details.

### 3.2 Seamless-wallet callback API — the standard operator-side contract

This is the part every operator engineering team must build correctly, because it sits in the hot path of real-money gameplay and must survive retries, timeouts, and duplicate delivery without ever double-crediting or double-debiting a player.

**Provenance note:** everything in this subsection quoting endpoint paths, field names, and status codes is drawn directly from **Hub88's public documentation** (`docs.hub88.io`), fetched during this research. It is presented as a **concrete, real, representative example** of the seamless-wallet pattern used across the industry (SoftSwiss, Slotegrator, Pariplay, EveryMatrix and studio-direct integrations follow the same conceptual shape — request from provider to operator, HMAC/RSA-signed, idempotent by UUID — but their exact field names could not be independently verified because their technical docs are gated). Anywhere a field/endpoint below is *not* from Hub88's docs, it is explicitly marked "illustrative."

#### Authentication / request signing
- Hub88 uses **RSA-SHA256** asymmetric signing. Operators generate a key pair (`openssl genrsa` / `openssl rsa -pubout`), give Hub88 their public key, and every Wallet API request Hub88 sends to the operator is signed with Hub88's private key and carries an `X-Hub88-Signature` header (BASE64-encoded RSA-SHA256 signature of the exact request body — no whitespace/reformatting allowed, or signature validation fails). Conversely, for the Games API (operator → Hub88), the operator signs and Hub88 verifies with the operator's public key. ([docs.hub88.io – Getting Started](https://docs.hub88.io/developer-docs/operator-api-reference/getting-started.md), [Private and public keys](https://docs.hub88.io/developer-docs/hub88-apis/private-and-public-keys.md))
- This asymmetric-signature pattern (rather than a shared static API key) is common in the industry precisely because the wallet callback endpoint is a bearer of real-money transactions and needs non-repudiation.

#### The operator endpoints the aggregator/provider calls (Hub88 Wallet API — real, from public docs)

| Endpoint | Method | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| `/user/info` | POST | Fetch player demographic/account data | `user`, `request_uuid` | `user`, `status`, `country`, optional `jurisdiction`, `birth_date`, `registration_date` |
| `/user/balance` | POST | Get current balance | `user`, `game_code`, `token`, `request_uuid` | `user`, `status`, `balance` (int64, ×100,000), `currency` |
| `/transaction/bet` | POST | Debit wager | `user`, `game_code`, `token`, `transaction_uuid`, `supplier_transaction_id`, `request_uuid`, `round`, `amount`, `currency`, `is_free`, `is_supplier_promo` | `status`, new `balance` |
| `/transaction/win` | POST | Credit payout | Same as bet, plus `reference_transaction_uuid` (points back to the bet's `transaction_uuid`), optional `reward_uuid` | `status`, new `balance` |
| `/transaction/rollback` | POST | Reverse a prior transaction | `user`, `game_code`, `token`, `round`, `transaction_uuid`, `supplier_transaction_id`, `request_uuid`, `reference_transaction_uuid` | `status`, new `balance` |

Source: [docs.hub88.io – Wallet API (Operator)](https://docs.hub88.io/developer-docs/operator-api-reference/wallet-api.md)

#### Idempotency and transaction/round IDs

- **`transaction_uuid`** is the idempotency key for every bet/win/rollback call. The operator must check for a previously-seen `transaction_uuid` before applying a debit/credit, and if it's a repeat, return the same result rather than re-applying the transaction.
- **`request_uuid`** is a separate per-HTTP-request tracking ID (distinct from the business transaction ID) used for request-level tracing/idempotency of the call itself.
- **`round`** groups all bet/win/rollback events belonging to a single game round together, and `round_closed` marks when a round is finished (important for game types like slots with tumbling/cascading wins where a round has multiple bet/win legs).
- **`reference_transaction_uuid`** links a win back to its originating bet, and a rollback back to the transaction it's cancelling.
- Both operator and provider are expected to **store the transaction ID for at least 4 months** for reconciliation purposes.
- **Retry policy (as documented):** bet/win calls are retried "3 times with 1 second timeout" on network failure; failed bets can trigger rollback retries up to **500 times with exponential backoff** — i.e., the system is built assuming transient failures are common and the operator endpoint must be safely re-callable indefinitely without side effects beyond the first successful application.
- Money amounts are transmitted as **Int64, scaled ×100,000** (e.g. $3.56 → `356000`) — a common pattern industry-wide to avoid floating-point rounding errors in financial fields.

Source: [docs.hub88.io – Wallet API (Operator)](https://docs.hub88.io/developer-docs/operator-api-reference/wallet-api.md), [Getting Started](https://docs.hub88.io/developer-docs/operator-api-reference/getting-started.md)

#### Response status codes the operator must implement (real, verbatim from Hub88 docs)

| Status | Meaning |
|---|---|
| `RS_OK` | Success — request accepted and processed |
| `RS_ERROR_UNKNOWN` | General/unclassified error |
| `RS_ERROR_INVALID_PARTNER` | Operator or sub-partner disabled, or bad `sub_partner_id` |
| `RS_ERROR_INVALID_TOKEN` | Token unknown to operator's system |
| `RS_ERROR_INVALID_GAME` | Unknown `game_code` |
| `RS_ERROR_WRONG_CURRENCY` | Transaction currency ≠ user's wallet currency |
| `RS_ERROR_NOT_ENOUGH_MONEY` | Insufficient balance to place bet |
| `RS_ERROR_USER_DISABLED` | User locked/disabled, can't place new bets |
| `RS_ERROR_INVALID_SIGNATURE` | Signature verification failed |
| `RS_ERROR_TOKEN_EXPIRED` | Session token expired |
| `RS_ERROR_WRONG_SYNTAX` | Request doesn't match expected schema |
| `RS_ERROR_WRONG_TYPES` | Field types don't match expected types |
| `RS_ERROR_DUPLICATE_TRANSACTION` | Same `transaction_uuid` resent with *different* transaction details (as opposed to a legitimate idempotent retry) |
| `RS_ERROR_TRANSACTION_DOES_NOT_EXIST` | Referenced bet (for a win) not found on operator's side |

Source: [docs.hub88.io – Seamless Wallet response statuses (Operator API)](https://docs.hub88.io/developer-docs/operator-api-reference/operator-api-overview/seamless-wallet-response-statuses-operator-api.md)

#### Illustrative example payloads

These payloads are **illustrative reconstructions** built from the documented field names above (not a copy-pasted real request/response body — Hub88's rendered docs describe fields in prose rather than exposing raw JSON samples via this fetch path). Field names are real; exact nesting/formatting is a reasonable typical shape, not verified byte-for-byte.

```json
// POST /user/balance  (Hub88 → Operator)
// illustrative, field names per docs.hub88.io
{
  "user": "op-player-88213",
  "game_code": "prgm_sweetbonanza",
  "token": "8f0a7c2e-91b4-4a3d-9c77-2a6e0d55f1aa",
  "request_uuid": "b1e2c3d4-0000-1111-2222-333344445555"
}
```

```json
// 200 OK response (Operator → Hub88)
{
  "user": "op-player-88213",
  "status": "RS_OK",
  "request_uuid": "b1e2c3d4-0000-1111-2222-333344445555",
  "balance": 356000,
  "currency": "EUR"
}
```

```json
// POST /transaction/bet  (Hub88 → Operator)
{
  "user": "op-player-88213",
  "game_code": "prgm_sweetbonanza",
  "token": "8f0a7c2e-91b4-4a3d-9c77-2a6e0d55f1aa",
  "transaction_uuid": "8b2a1f00-aaaa-bbbb-cccc-ddddeeeeffff",
  "supplier_transaction_id": "prgm-bet-99231122",
  "request_uuid": "b1e2c3d4-1111-2222-3333-444455556666",
  "round": "round-7788990011",
  "round_closed": false,
  "amount": 100000,
  "currency": "EUR",
  "is_free": false,
  "is_supplier_promo": false
}
```

```json
// 200 OK response
{
  "user": "op-player-88213",
  "status": "RS_OK",
  "request_uuid": "b1e2c3d4-1111-2222-3333-444455556666",
  "balance": 256000
}
```

```json
// POST /transaction/win  (Hub88 → Operator)
{
  "user": "op-player-88213",
  "game_code": "prgm_sweetbonanza",
  "token": "8f0a7c2e-91b4-4a3d-9c77-2a6e0d55f1aa",
  "transaction_uuid": "3fa1c2d3-eeee-ffff-0000-111122223333",
  "reference_transaction_uuid": "8b2a1f00-aaaa-bbbb-cccc-ddddeeeeffff",
  "supplier_transaction_id": "prgm-win-99231123",
  "request_uuid": "b1e2c3d4-2222-3333-4444-555566667777",
  "round": "round-7788990011",
  "round_closed": true,
  "amount": 450000,
  "currency": "EUR"
}
```

```json
// POST /transaction/rollback  (Hub88 → Operator)
{
  "user": "op-player-88213",
  "game_code": "prgm_sweetbonanza",
  "token": "8f0a7c2e-91b4-4a3d-9c77-2a6e0d55f1aa",
  "transaction_uuid": "9c3d4e5f-6666-7777-8888-999900001111",
  "reference_transaction_uuid": "8b2a1f00-aaaa-bbbb-cccc-ddddeeeeffff",
  "supplier_transaction_id": "prgm-rollback-99231122",
  "request_uuid": "b1e2c3d4-3333-4444-5555-666677778888",
  "round": "round-7788990011",
  "round_closed": true
}
```

```json
// Error case: retried bet with same transaction_uuid, operator already applied it
{
  "user": "op-player-88213",
  "status": "RS_OK",
  "request_uuid": "b1e2c3d4-1111-2222-3333-444455556666",
  "balance": 256000
}
// NOTE: correct idempotent behavior — same transaction_uuid returns
// the SAME balance/result as the original application, not a fresh debit.
```

#### The critical sequencing rule

Hub88's docs state explicitly: **suppliers must not make any Wallet API calls before returning a successful `/game/url` response to Hub88** — the session token is not considered active by Hub88 until Hub88 has received and processed a valid game URL response. This ordering rule (launch handshake completes *before* any wallet traffic starts) is a common industry safety property: it prevents a race where a studio starts debiting a player before the operator has even confirmed the session/token is valid.

### 3.3 Game launch flow

The launch sequence (again, using Hub88's documented shape as the concrete example — the same conceptual flow: operator requests URL → aggregator forwards to supplier → supplier returns embeddable URL → operator embeds/redirects, is described in essentially every aggregator's marketing/integration material):

1. Player clicks "Play" in the operator's lobby.
2. Operator's backend generates a **session token** — a value the operator itself creates and will later validate on every wallet callback — and calls the aggregator: `POST /operator/generic/v2/game/url`.
3. Aggregator forwards to the studio's own `POST /game/url` on the Supplier API.
4. Studio returns a launch URL to the aggregator, which relays it to the operator.
5. Operator's frontend embeds that URL (iframe or redirect) — this is what actually loads the studio's game client into the operator's page.
6. Only *after* that URL exchange succeeds may the studio's RGS start calling the operator's Wallet API (balance → bet → win → …) using the token from step 2.

**Request fields for `game/url`** (real, from Hub88 Games API docs — [docs.hub88.io](https://docs.hub88.io/developer-docs/operator-api-reference/games-api.md)):

| Field | Required | Purpose |
|---|---|---|
| `user` | Yes (real money) | Unique operator player ID (≥3 chars) |
| `game_code` | Yes | Which game to launch, from `/game/list` |
| `operator_id` | Yes | Operator's account ID in the aggregator's system |
| `platform` | Yes | `GPL_DESKTOP` or `GPL_MOBILE` |
| `lang` | Yes | ISO 639-1 language code |
| `currency` | Yes | ISO 4217 wallet currency (or `XXX` for demo) |
| `game_currency` | Yes | In-game display currency |
| `country` | Yes | ISO 3166-1 country code (used for jurisdictional content filtering) |
| `lobby_url` | Yes | Return URL when player clicks "Home"/lobby |
| `deposit_url` | Optional | Deep link to the deposit page (shown if the player runs low on funds mid-session) |
| `token` | Optional | Session token; **omit for demo mode** |
| `sub_partner_id` | Optional | Sub-brand/white-label identifier for multi-brand operators |
| `meta` | Optional | Supplier-specific extra parameters |

Demo/fun mode is triggered simply by passing `currency: "XXX"` and omitting `token`/`user` — in that mode, **no wallet calls occur at all**; the studio manages a fake in-memory balance client-side or on its own RGS. Currency cannot change mid-session — a currency switch requires issuing a brand-new session token.

Jurisdiction/geo-filtering is enforced at the catalogue level: the Games API's `/game/list` response carries `blocked_countries` and `restricted_countries` (ISO codes) per game, and `certifications` per game (e.g. `CURACAO`, `MGA`, `IOM`), which the operator is expected to respect when deciding what to show a given player ([docs.hub88.io – Games API](https://docs.hub88.io/developer-docs/operator-api-reference/games-api.md)).

### 3.4 RGS role, RNG certification, round lifecycle, free spins, jackpots

**RGS location.** The actual game math and RNG execution run **on the studio's Remote Game Server, not the operator's infrastructure.** The operator only ever sees the *outcome* of a round via the wallet debit/credit callbacks — it never computes or has visibility into the RNG itself. This is architecturally load-bearing: the operator's regulatory obligation is to integrate only with studios/games that are independently certified, not to certify game math itself. Industry sources describe the RGS as hosting "game logic, RNG output processing, transaction handling, and player session integrity," essentially acting as the central nervous system for gameplay while exposing only wallet-callback and launch/reporting APIs to the outside world ([capermint.com](https://www.capermint.com/how-a-remote-gaming-server-works/), [gamingassociates.com](https://gamingassociates.com/blog/remote-gaming-server-rgs-certification/)).

**RNG certification.** Every regulated jurisdiction requires the RGS's RNG to be independently tested before a game can go live for real money. GLI-19 §3.2.1 specifically requires the testing lab to review **source code** for core randomness/shuffling/scaling algorithms — not just black-box statistical testing ([gaminglabs.com GLI-19 PDF](https://gaminglabs.com/wp-content/uploads/2020/07/GLI-19-Interactive-Gaming-Systems-v3.0.pdf), summarized via [gamingcompliance.io](https://gamingcompliance.io/gli-19-v3-0-what-every-online-casino-game-must-meet-to-pass-certification/)). See §5 for the full certification landscape.

**Round ID lifecycle.** A `round` groups all the transactions (bet, possible multiple wins for cascading/tumbling mechanics, possible rollback) belonging to one play of the game, and a `round_closed` boolean on the final transaction tells the operator the round is finished and safe to consider settled for reconciliation purposes (Hub88 Wallet API, §3.2 above). This matters for games with multi-step rounds (e.g., a slot with a cascading-wins feature that produces several sequential win transactions before the round is truly over, or a bonus round that spans multiple spins).

**Free spins / free-round bonus campaigns.** These are run through a dedicated API layer, separate from core wallet transactions but flagged into them. Hub88's **Freebets API** (real endpoints, from public docs):

| Endpoint | Method | Purpose |
|---|---|---|
| `/operator/generic/v2/freebet/rewards/create` | POST | Award a single free-bet/free-spin reward to one player |
| `/operator/generic/v2/freebet/rewards/create_bulk` | POST | Award to up to 100 players in one call |
| `/operator/generic/v2/freebet/rewards/list` | POST | Query reward status (paginated) |
| `/operator/generic/v2/freebet/rewards/cancel` | POST | Cancel an unclaimed reward |
| `/operator/generic/v2/freebet/campaigns/create` | POST | Create a reusable campaign template |
| `/operator/generic/v2/freebet/campaigns/list` / `/campaigns/info` | POST | Query campaigns |
| `/operator/generic/v2/freebet/prepaids/list` | POST | List available reward "prepaid" templates (bet value, currency, game) |

Reward statuses: `FRS_NEW`, `FRS_ACTIVE`, `FRS_CANCELLED`, `FRS_FAILED`, `FRS_FINISHED`, `FRS_EXPIRED`. When a player uses a free-spin reward, the resulting bet transaction carries `is_free: true` so the operator's wallet knows not to actually debit real cash for that wager ([docs.hub88.io – Freebets API](https://docs.hub88.io/developer-docs/operator-api-reference/freebets-api.md); corroborated conceptually by other vendors, e.g. St8's "Bonus API" description of free spins/cash bonuses/bonus-buy "through one API" — [st8.io](https://st8.io/bonus-api/)).

**Jackpot contribution/feed mechanics.** A small slice of every qualifying wager — commonly cited as roughly **0.5%–2%** of stake, sometimes framed as up to 5% depending on the game/network — is diverted into a shared prize pool rather than returned as base-game RTP; this is why progressive jackpot slots typically show a lower headline base RTP (often ~92–94%) than non-jackpot slots ([multiple industry consumer-facing sources synthesized, e.g. track360.io jackpot guide](https://track360.io/blog/jackpot-slots-progressive-must-drop-operator-guide-2026); these are consumer-education sources, not primary provider documentation — treat exact percentages as approximate/illustrative]). Jackpot pools come in three shapes:
- **Standalone** — lives inside one game on one operator.
- **Local progressive** — pooled across all instances of a game within a single operator.
- **Wide-area progressive** — pooled across many operators running the same game through a shared studio network (this is how eight/nine-figure jackpots like NetEnt's Mega Fortune or Microgaming's Mega Moolah are funded — contributions flow from every operator carrying the game into one shared pool the studio administers).
- Studios sometimes seed the pool back to a minimum after a win to keep it attractive; who funds that seed (studio vs. operator) is a negotiated/contractual detail not found in public sources during this research — flagged as **unknown/gated**.

Hub88's Games API changelog also references a `phoenix_jackpot_support` field (later deprecated) on its game-list response, confirming jackpot participation is at minimum surfaced as a per-game flag in aggregator catalogues ([docs.hub88.io changelog, June 2025](https://docs.hub88.io/api-changelog/june-2025/deprecation-of-phoenix_jackpot_support-in-operator-api-game-list-response.md)) — exact contribution-feed mechanics (a dedicated jackpot-contribution wallet API) were not found in accessible public docs and should be treated as **gated/unknown** beyond the general industry description above.

### 3.5 Reporting / reconciliation and GGR calculation

- Hub88 exposes a **Transactions API** (`POST /operator/generic/v2/transactions/list`) returning per-transaction records with `transaction_uuid`, `transaction_status` (`TS_SUCCESS`, `TS_DECLINED`, `TS_ROLLEDBACK`, `TS_RETRY_ATTEMPTS_LIMIT_EXCEEDED`), `kind` (`TK_BET`, `TK_WIN`, `TK_ROLLBACK`, `TK_FREE_BET`, `TK_FREE_WIN`, `TK_JP_WIN`, `TK_NEG_BET`, `TK_BONUS_WIN`), `amount`, `currency`, `round`, `reference_transaction_uuid`, `inserted_at`, `game_code` — explicitly positioned as the feed operators use for **troubleshooting and reconciliation/GGR calculation** ([docs.hub88.io – Transactions API](https://docs.hub88.io/developer-docs/operator-api-reference/transactions-api.md)). This is a real, concrete example of what a provider-side settlement feed looks like.
- **GGR** in this context = total bets − total wins (before bonus cost and before operator overhead) for the games traffic behind that feed, computed independently on both sides (operator's own transaction log vs. the provider's Transactions API/settlement report) and reconciled — discrepancies are investigated as either systematic (a business-rule mismatch, e.g. a misapplied revenue-share tier or bonus-cost attribution) or random (data-quality/timing issues), per general industry reconciliation practice ([track360.io commission reconciliation](https://track360.io/blog/ngr-vs-ggr-commission-calculation-operator-deep-dive-2026)).
- Standard practice: both sides retain transaction records for a minimum retention window for reconciliation purposes — Hub88 documents **4 months minimum** for `transaction_uuid` storage on both operator and Hub88 sides.
- Many operators run **two parallel GGR figures**: an "operational" GGR computed live off the transaction stream for dashboards, and a "regulatory" GGR computed for statutory filings, reconciled against each other — a pattern described generically in industry commentary rather than tied to one named vendor ([journalentrieshub.com](https://www.journalentrieshub.com/entries/casino-igaming-online-revenue), synthesized).
- Settlement/invoicing itself (who pays whom, on what cadence) happens **outside** the transaction API — typically monthly invoicing from studio/aggregator to operator based on the reconciled GGR figure and the contracted revenue-share formula (§4).

---

## 4. Commercials

**Confidence: low-to-moderate.** Actual contracted rates are confidential; every figure below is sourced from industry blogs, consultancies, or comparison sites rather than a disclosed contract or regulatory filing, and ranges vary by source. Treat all percentages as **indicative ranges**, not verified facts, and always confirm with current vendor commercial teams before using in a real financial model.

| Commercial term | Range found | Source(s) | Confidence |
|---|---|---|---|
| Studio → operator (direct) revenue share | ~7%–20% of GGR, varying by studio brand weight/content exclusivity; large/established studios sit at the fixed, less-negotiable end | [track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026), gamingsoft.com negotiation article (fetch failed directly, summarized via search) | Low — directional only |
| Aggregator all-in rate (studio share + aggregator margin bundled) | ~5%–15% of GGR | [onlyigaming.com](https://onlyigaming.com/categories/game-aggregators) (search-summarized) | Low |
| Aggregator margin layered on top of pass-through studio rates | ~1%–4 percentage points (sometimes described as a flat "technical gateway" fee of 1–3%) | [track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026), [onlyigaming.com](https://onlyigaming.com/categories/game-aggregators) | Low |
| Aggregator monthly minimum guarantee | Roughly €5,000–€25,000/month cited in one secondary source | search-summarized secondary source (henkwolff.com content, could not verify by direct fetch — 403) | Very low — single unverified secondary mention |
| Integration/setup fees | Not disclosed publicly by any vendor found; several vendors (Slotegrator, Hub88) market **no/low setup friction** as a selling point rather than publishing a fee schedule | — | Not found |
| Direct-integration engineering cost (secondary estimate) | 6–10 weeks engineering time per studio for a mid-size operator running many direct integrations, plus tens of thousands of EUR/year in certification overhead per jurisdiction | [track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026) | Low — single-source estimate |
| Jackpot contribution rate | ~0.5%–2% of stake into the pool (some consumer sources cite up to 5%) | synthesized from consumer-facing gambling-guide sources, e.g. [track360.io](https://track360.io/blog/jackpot-slots-progressive-must-drop-operator-guide-2026) | Low — consumer education sources, not provider disclosures |
| Who funds jackpot seeding after a win | Not established from public sources — some jackpots are "partially funded by the casino/studio itself," exact split not disclosed | consumer-facing sources | Unknown/gated |
| Live-casino dedicated/branded table cost | No public pricing found for either Evolution's fully-dedicated studio/table model or Pragmatic Play Live's shared "Smart Studio" model; only that Evolution has built 600+ client-dedicated setups by end of 2024, and Pragmatic's Smart Studio is explicitly marketed as the lower-cost, multi-brand-per-table alternative to a fully dedicated studio | [Evolution](https://www.evolution.com/about/solutions/dedicated-tables), [Pragmatic Play Smart Studio](https://www.pragmaticplay.com/en/live-casino/smart-studio/), [Newsroom Panama comparison](https://newsroompanama.com/2026/07/21/comparing-evolution-pragmatic-play-live-and-gaming-for-streaming-and-table-depth/) | Not found — no numeric pricing publicly available |

**What's confirmed by multiple independent sources, even without exact numbers:**
- Revenue share is layered: **studio rate + aggregator margin**, both ultimately borne by the operator, both deducted before NGR is calculated ([Soft2Bet](https://www.soft2bet.com/news/igaming-operator-economics-what-makes-up-ggr-ngr-and-where-margin-leaks), [track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026)).
- Minimum guarantees from individual studios are commonly **passed through** an aggregator to the operator, and this pass-through is not always transparent in the aggregator's headline commercial pitch.
- Tiered/volume-based pricing (rate falls as GGR volume rises) is a standard structure across multiple aggregators, per comparison-site summaries.
- Ramp-up periods with reduced minimums for the first 3–6 months of a new operator relationship are a common negotiated concession, per industry commentary.

---

## 5. Certification & compliance

### 5.1 Standards bodies and what they cover

| Standard/body | Scope | Notes |
|---|---|---|
| **GLI-19** (Gaming Laboratories International) | "Standards for Interactive Gaming Systems" — the primary standard referenced for online casino/RGS certification globally | Section 3.2.1 requires **source-code review** of core RNG/shuffling/scaling algorithms, not just black-box statistical testing ([GLI-19 v3.0 PDF](https://gaminglabs.com/wp-content/uploads/2020/07/GLI-19-Interactive-Gaming-Systems-v3.0.pdf)) |
| **GLI-11** | "Standards for Gaming Devices in Casinos" — physical/land-based gaming devices, not online systems | Distinct from GLI-19; referenced here for contrast since the prompt named it, but it is **not** the relevant standard for online RGS/game content ([GLI-11 v2.0 PDF](https://gaminglabs.com/wp-content/uploads/2018/09/GLI-11-v2-0-Standard-FINAL.pdf)) |
| **GLI-33** | Referenced alongside GLI-19 in secondary sources as another relevant interactive-gaming-adjacent standard for some jurisdictions | [legarithm.io](https://legarithm.io/game-provider-certifications/) — could not independently verify exact GLI-33 scope from a primary GLI source in this session |
| **ISO/IEC 27001** | Information security management system (ISMS) — organizational, not game-math — covering player data, payment data, and business information security | Increasingly a baseline expectation for B2B studios seeking operator procurement approval; 3-year certification cycle with periodic surveillance/recertification audits ([isms.online](https://www.isms.online/sectors/iso-27001-for-the-gaming-industry/), [gamingassociates.com](https://gamingassociates.com/iso-iec-27001-certification/)) |
| **eCOGRA** | Independent testing lab (UK-headquartered, est. 2003) — RNG, RTP, game-logic testing; also player-protection/fairness seal | [ecogra.org](https://ecogra.org/) |
| **iTech Labs** | Independent testing lab (Australia, est. 2004) — RNG, RTP compliance, software security; recognized in Canada, UK, Australia among others | search-summarized |
| **BMM Testlabs** | Independent testing lab (est. 1981), operating across 14 countries, licensed/recognized in 400+ gaming jurisdictions claimed | search-summarized |

Testing labs test **both** the studio's declared math (does the game actually pay out at the claimed RTP, is the RNG statistically sound) **and**, separately, the operator/vendor's deployment (has the operator tampered with anything, is the deployed build the certified build) — i.e., certification isn't a one-time studio-only event, it also touches how the operator serves the game.

### 5.2 Jurisdiction-specific licensing

| Jurisdiction | Regulator | Notable process detail |
|---|---|---|
| Malta | Malta Gaming Authority (MGA) | 6–12 month vetting for a full license; 90-day temporary license period during which system/compliance audits must complete; independent 3rd-party systems audit required before going live; license term ~10 years; quarterly compliance reporting thereafter. All games licensed in Malta must be independently certified (RNG, RTP, functional checks) before launch. ([gamblingnerd.com](https://www.gamblingnerd.com/blog/malta-gaming-authority-explained/), [csbgroup.com](https://www.csbgroup.com/igaming-services-malta/online-gaming-licence-application/)) |
| UK | UK Gambling Commission (UKGC) | Notable for **content restrictions beyond certification**: bonus-buy/feature-buy mechanics have been effectively unavailable on UKGC-licensed sites since **2019** on responsible-gambling grounds — studios like Pragmatic Play, Nolimit City, Hacksaw Gaming and Relax Gaming, who popularized bonus-buy as a mechanic, ship **UK-specific rebuilt versions** of affected titles with the feature stripped or replaced (e.g., an "ante-bet" model instead of an instant bonus purchase) rather than simply geo-blocking the title outright. UKGC also introduced (per this research's date context, effective Dec 2025/Jan 2026) a cap on bonus wagering requirements at 10x and restrictions on cross-product promotional incentives. ([bakestreet.co.uk](https://bakestreet.co.uk/bonus-buy-slots-uk/), [primecasino.co.uk](https://www.primecasino.co.uk/blog/guides/casino-slot-studio-no-limit-city/) — these are consumer-facing/industry sources, not the UKGC's own primary regulatory text, which was not directly retrieved in this session) |
| Curaçao | Curaçao Gaming Authority (CGA, since Sept 2023 reform; previously "Curaçao eGaming" sub-license model) | Historically the fastest/cheapest route to market; post-2023 reform tightened requirements and ended sub-licensing; still positioned as faster/cheaper than Malta (~6 weeks cited for licensing timeline in one source) ([turbomates.com](https://turbomates.com/blog/gaming-licenses-comparison-of-popular-jurisdictions/)) |
| Anjouan (Comoros) | Anjouan Gaming regulator | Popular low-cost/fast alternative to Curaçao; no physical office or local staff required, remote management permitted; lower prestige/trust than Malta or UK for regulated-market operators ([mygaminglicense.com](https://www.mygaminglicense.com/blog/the-best-license-between-curacao-and-anjouan-for-your-igaming-business), [altenar.com](https://altenar.com/blog/anjouan-licence-vs-curacao-licence-what-are-the-main-differences-and-which-one-should-you-choose/)) |
| Kahnawake (Mohawk Territory, Canada) | Kahnawake Gaming Commission (KGC) | Requires a Client Provider Authorization (CPA, B2C permit) plus a Key Person License; **all gaming servers must be physically hosted within Kahnawake territory**; 6-month trial license before a 5-year full license is granted; RNG/RTP testing via approved labs (eCOGRA, GLI, iTech Labs) required, or an agreement with an already-certified developer; secondary-source estimate of **$8,000–$15,000 for initial RNG testing**, **$3,000–$5,000/year for recertification** ([tetraconsultants.com](https://www.tetraconsultants.com/jurisdictions/kahnawake-gaming-license/), [wizards.us](https://wizards.us/blog/kahnawake-gaming-license/)) |
| US states (NJ, PA, MI, WV, CT, DE, RI live as of this research; ME legalized-not-launched) | Individual state regulators (e.g., NJ Division of Gaming Enforcement, Pennsylvania Gaming Control Board) | **Fully state-siloed**: each state runs independent licensing, sets its own approved game list, and audits games/operators separately — a studio or operator licensed and certified in NJ is **not** automatically approved in PA or MI; content, RTP configuration and even betting limits can differ state-by-state, meaning the same nominal game may need separate state-level approval/certification runs ([track360.io US market map](https://track360.io/blog/us-real-money-online-casino-states-operator-market-map-2026)) |

### 5.3 Per-game, per-jurisdiction certification reality

Certification is **not** a one-time, studio-wide event — it is closer to **(game × RTP variant × jurisdiction)**:

1. **Math model finalization** — PAR sheets, RTP configuration, volatility settings; internally validated with large-scale simulation (10M–100M spins cited as typical) before ever reaching a lab.
2. **Technical documentation package** — game rules, paytables, probability models, RNG design, platform integration detail submitted to the lab.
3. **Pre-certification internal testing** — studio's own RNG/math stress tests to reduce the chance of a failed lab submission.
4. **Independent lab testing** — GLI/iTech Labs/eCOGRA/BMM audit RNG statistics, review source code, verify game logic and RTP; **typically cited as taking 4–12 weeks**.
5. **Certification report issuance**.
6. **Regulatory submission** — the lab's report goes to the specific jurisdiction's regulator (UKGC, MGA, individual US state boards, etc.) for that jurisdiction's own sign-off.
7. **Deployment** via the studio's RGS or an aggregator.

Different regulated markets legitimately require **different configurations of the same underlying game** — e.g., a max-bet cap and disabled autoplay in the UK, unlimited bet/autoplay in Ontario, bonus-buy allowed in some markets and restricted/removed in the UK — meaning a single title can require **multiple, separately-certified builds**. A "modular" studio architecture (certify the core math/RNG engine once, then layer jurisdiction-specific configuration on top) is described as reducing but **not eliminating** duplicate retesting across markets, with a typical **4–6 week** turnaround for a jurisdictional variant of an already-certified game versus 4–12 weeks for a wholly new title ([gamixlabs.com](https://gamixlabs.com/blog/slot-game-certification-workflow-for-regulated-markets/) — this is an industry consultancy blog, not a regulator's own published process document, so treat the specific week-counts as indicative rather than official SLAs).

### 5.4 Geo-restriction of content

Aggregator catalogues carry this per-game, machine-readable, as seen concretely in Hub88's `/game/list` response: `blocked_countries` and `restricted_countries` (ISO codes) and a `certifications` array (e.g. `CURACAO`, `MGA`, `IOM`) per game ([docs.hub88.io – Games API](https://docs.hub88.io/developer-docs/operator-api-reference/games-api.md)). Operators are expected to cross-reference the player's declared/detected jurisdiction against this per-game metadata before offering a title — this is the mechanism by which, for example, a Hacksaw or Nolimit City bonus-buy title is simply absent (or served in a stripped-down UK build) from a UK-licensed operator's lobby while remaining fully available, bonus-buy intact, on the same operator's brand in a market without that restriction; and by which a title not yet separately licensed/certified in, say, Michigan is unavailable to Michigan players even if it's live in New Jersey on the same platform.

---

## Sources

**Hub88 public developer documentation (primary source — the strongest source in this document):**
- [Welcome to Hub88 Documentation](https://docs.hub88.io/)
- [Fundamentals](https://docs.hub88.io/developer-docs/hub88-apis/fundamentals.md)
- [Core API flow](https://docs.hub88.io/developer-docs/hub88-apis/core-api-flow.md)
- [Private and public keys](https://docs.hub88.io/developer-docs/hub88-apis/private-and-public-keys.md)
- [Getting Started - Technical Integration (Operator API)](https://docs.hub88.io/developer-docs/operator-api-reference/getting-started.md)
- [Operator API Overview](https://docs.hub88.io/developer-docs/operator-api-reference/operator-api-overview.md)
- [Demo and Real Gameplay](https://docs.hub88.io/developer-docs/operator-api-reference/operator-api-overview/demo-and-real-gameplay.md)
- [Seamless Wallet response statuses – Operator API](https://docs.hub88.io/developer-docs/operator-api-reference/operator-api-overview/seamless-wallet-response-statuses-operator-api.md)
- [Games API (Operator)](https://docs.hub88.io/developer-docs/operator-api-reference/games-api.md)
- [Wallet API (Operator)](https://docs.hub88.io/developer-docs/operator-api-reference/wallet-api.md)
- [Transactions API](https://docs.hub88.io/developer-docs/operator-api-reference/transactions-api.md)
- [Freebets API](https://docs.hub88.io/developer-docs/operator-api-reference/freebets-api.md)
- [Supplier API Overview](https://docs.hub88.io/developer-docs/supplier-api-reference/supplier-api-overview.md)
- [API Changelog – phoenix_jackpot_support deprecation](https://docs.hub88.io/api-changelog/june-2025/deprecation-of-phoenix_jackpot_support-in-operator-api-game-list-response.md)
- [Hub88 – Casino API Integration (marketing/overview)](https://hub88.io/casino-api-integration/)

**Aggregator/platform marketing & overview pages:**
- [SoftSwiss Game Aggregator](https://www.softswiss.com/game-aggregator/)
- [Slotegrator APIgrator](https://slotegrator.pro/apigrator.html)
- [Pariplay Fusion transforms gaming integration – iGaming Business](https://igamingbusiness.com/gaming/pariplay-fusion-aggregation-platform-transforms-gaming/)
- [EveryMatrix CasinoEngine](https://everymatrix.com/casinoengine/)
- [EveryMatrix – API bet pays off](https://everymatrix.com/news/everymatrixs-api-bet-pays-off-as-casinoengine-remains-platform-of-choice-for-industry/)
- [SoftGamings Casino API](https://www.softgamings.com/casino-api/)
- [SoftGamings – Golden Race](https://www.softgamings.com/online-gambling-software-providers/golden-race/)
- [Relax Gaming – Silver Bullet](https://www.relax-gaming.com/partners/silverbullet)
- [Yggdrasil YGS Masters – CasinoBeats](https://casinobeats.com/2018/12/05/yggdrasil-ygs-masters-from-concept-to-reality/)
- [Yggdrasil adds Lakka Studios (20th YGS Masters partner)](https://www.igamingtoday.com/yggdrasil-adds-lakka-studios-as-20th-ygg-masters-partner-expanding-nordic-led-content-pipeline/)
- [YGS Masters explained – goodluckmate.com](https://goodluckmate.com/blog/all-you-need-to-know-about-the-yggdrasil-ygs-masters-program)
- [Booming Games – about](https://booming-games.com/about-us)
- [BGaming – SoftGamings profile](https://www.softgamings.com/online-gambling-software-providers/bgaming/)
- [iGaming Game Aggregators 2026 – onlyigaming.com](https://onlyigaming.com/categories/game-aggregators)

**Direct vs. aggregator / commercial analysis (secondary/industry commentary):**
- [Casino Game Providers vs Aggregators – track360.io](https://track360.io/blog/casino-game-providers-aggregators-operator-integration-2026)
- [Benefits of Game Aggregators – groovetech.com](https://www.groovetech.com/blog/benefits-of-game-aggregators-for-online-casino-operators)
- [iGaming Operator Economics: GGR/NGR – Soft2Bet](https://www.soft2bet.com/news/igaming-operator-economics-what-makes-up-ggr-ngr-and-where-margin-leaks)
- [NGR vs GGR Commission Calculation – track360.io](https://track360.io/blog/ngr-vs-ggr-commission-calculation-operator-deep-dive-2026)
- [Jackpot Slots: Progressive & Must-Drop Economics – track360.io](https://track360.io/blog/jackpot-slots-progressive-must-drop-operator-guide-2026)
- [Understanding GGR – BGaming](https://bgaming.com/articles/understanding-ggr-in-the-online-gambling-industry)

**Live casino:**
- [Evolution – Dedicated Tables](https://www.evolution.com/about/solutions/dedicated-tables)
- [Pragmatic Play – Smart Studio Technology](https://www.pragmaticplay.com/en/live-casino/smart-studio/)
- [Comparing Evolution, Pragmatic Play Live, and Gaming for Streaming and Table Depth – Newsroom Panama](https://newsroompanama.com/2026/07/21/comparing-evolution-pragmatic-play-live-and-gaming-for-streaming-and-table-depth/)

**Certification & compliance:**
- [GLI-19 v3.0 standard (PDF) – Gaming Laboratories International](https://gaminglabs.com/wp-content/uploads/2020/07/GLI-19-Interactive-Gaming-Systems-v3.0.pdf)
- [GLI-11 v2.0 standard (PDF) – Gaming Laboratories International](https://gaminglabs.com/wp-content/uploads/2018/09/GLI-11-v2-0-Standard-FINAL.pdf)
- [GLI-19 v3.0 explained – gamingcompliance.io](https://gamingcompliance.io/gli-19-v3-0-what-every-online-casino-game-must-meet-to-pass-certification/)
- [Game Provider Certifications Explained: ISO 27001, GLI-19, GLI-33 & RNG – legarithm.io](https://legarithm.io/game-provider-certifications/)
- [ISO/IEC 27001 for the Gaming Industry – isms.online](https://www.isms.online/sectors/iso-27001-for-the-gaming-industry/)
- [ISO 27001 Certification – gamingassociates.com](https://gamingassociates.com/iso-iec-27001-certification/)
- [eCOGRA](https://ecogra.org/)
- [eCOGRA, iTech Labs & GLI: Casino Game Testing Explained – safeonlinecasino-uk.com](https://safeonlinecasino-uk.com/articles/ecogra-itech-labs-gli-casino-game-testing/)
- [Slot Game Certification Workflow for Regulated Markets – gamixlabs.com](https://gamixlabs.com/blog/slot-game-certification-workflow-for-regulated-markets/)
- [Remote Gaming Server (RGS) Certification – gamingassociates.com](https://gamingassociates.com/blog/remote-gaming-server-rgs-certification/)
- [How a Remote Gaming Server Works – capermint.com](https://www.capermint.com/how-a-remote-gaming-server-works/)
- [Malta Online Casino License – MGA explained – gamblingnerd.com](https://www.gamblingnerd.com/blog/malta-gaming-authority-explained/)
- [Malta Online Gaming Licence Application – csbgroup.com](https://www.csbgroup.com/igaming-services-malta/online-gaming-licence-application/)
- [2025/2026 iGaming Licenses Comparison – turbomates.com](https://turbomates.com/blog/gaming-licenses-comparison-of-popular-jurisdictions/)
- [Curaçao vs Anjouan – mygaminglicense.com](https://www.mygaminglicense.com/blog/the-best-license-between-curacao-and-anjouan-for-your-igaming-business)
- [Anjouan vs Curacao – altenar.com](https://altenar.com/blog/anjouan-licence-vs-curacao-licence-what-are-the-main-differences-and-which-one-should-you-choose/)
- [Kahnawake Gaming License – tetraconsultants.com](https://www.tetraconsultants.com/jurisdictions/kahnawake-gaming-license/)
- [Kahnawake Gaming License 2026 – wizards.us](https://wizards.us/blog/kahnawake-gaming-license/)
- [US iGaming state-by-state market map 2026 – track360.io](https://track360.io/blog/us-real-money-online-casino-states-operator-market-map-2026)
- [Bonus Buy Slots UK – bakestreet.co.uk](https://bakestreet.co.uk/bonus-buy-slots-uk/)
- [No Limit City Slots – UKGC-Compliant – primecasino.co.uk](https://www.primecasino.co.uk/blog/guides/casino-slot-studio-no-limit-city/)

**Not independently accessed (gated/private) — noted explicitly where relevant above:**
- SoftSwiss, Slotegrator, Pariplay, EveryMatrix, Pragmatic Play, Evolution, Relax Gaming and Yggdrasil's actual technical/API reference documentation are behind partner/operator login and were not reachable via public web search/fetch in this session. All statements about their technical field-level API contracts are inference from the Hub88 pattern plus vendor marketing copy, and are labeled as such in the text above.
- henkwolff.com's aggregator-selection article returned HTTP 403 on direct fetch; only search-engine-summarized fragments of it were used, and are labeled as low-confidence, single-source.
- bubblemarble.pro's SoftSwiss integration guide returned HTTP 404 on direct fetch and was not used as a source.
