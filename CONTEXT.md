# Casino Management Platform

The domain language for this investigation and for any platform built out of it. Every term below appears in
`FINDINGS.md`, the research reports, or the vendor's own product surface — this file fixes what each one means
so the documents, and any eventual build, use one vocabulary.

Where a vendor's product uses a different word for a concept defined here, that is recorded under _Vendor term_
rather than adopted. Their naming is evidence, not our language. Two vendors are cited: **ARC** (GammaStack) and
**MBO** (the second back office, `src-analysis/MBO-BACKOFFICE-ANALYSIS.md`).

## Language

### Commercial model

**Operator**:
The business that runs a casino brand and owns the relationship with the player. In a white label the operator
is not the licence holder and not the platform owner.
_Avoid_: merchant, casino, client

**White label**:
An arrangement where the vendor owns the platform *and* holds the gambling licence, and the operator trades
under the vendor's sub-licence for a share of revenue.
_Avoid_: turnkey (a different model — see below)

**Turnkey**:
An arrangement where the vendor supplies the platform but the operator holds its own gambling licence.
Routinely conflated with white label; the licence is the whole distinction.

**Self-service**:
Operator owns both the platform and the licence. The build case.
_Avoid_: ownership model, in-house

**Rev share**:
The vendor's cut, quoted as a percentage of GGR or of NGR. **Which of the two is not a detail** — the same
percentage against NGR is worth materially less than against GGR, and vendors quote whichever flatters them.

**Brand**:
The customer-facing casino identity: domain, name, theme, content, enabled features.
_Avoid_: skin, site
_Vendor term_: ARC calls the content-scoping field "Portal" and the theme a numbered "layout" (`layout2`).

**Tenant**:
The isolation unit in a multi-tenant deployment — one brand's data, configuration and enabled module set.
One tenant serves one brand; the platform serves many tenants.
_Vendor term_: MBO calls the tenant a **Hall**, and places it at the bottom of a five-level tree —
`Root → Root_folder → Project → Folder → Hall`. Every back-office page is bound to one of those levels, and
configuration inherits downward. A Hall owns its currency set and language set.

### Revenue

**GGR** (Gross Gaming Revenue):
Total bets minus total wins. What the house kept before any costs.
_Avoid_: gross win, hold

**NGR** (Net Gaming Revenue):
GGR minus deductions — typically bonus cost, payment processing fees, gaming duties and provider fees. **The
deduction list is contractual, not standard**, so an NGR figure means nothing without the definition attached.
_Avoid_: net win

**FTD** (First Time Deposit):
A player's first successful deposit. The standard acquisition-funnel conversion event, and a KPI in its own
right.

### Game supply

**Game studio**:
The company that owns a game's math and art, and that runs the RGS which executes it. Pragmatic, Evolution,
BGaming.
_Avoid_: game provider, vendor, supplier
_Vendor term_: ARC's catalogue calls these "Providers".

**RGS** (Remote Gaming Server):
The studio's server that runs the game logic and the RNG and decides outcomes. It always executes on the
studio's infrastructure, never the operator's — which is why the operator certifies integration, not math.

**Aggregator**:
An intermediary that holds integrations to many studios and re-exposes them through one contract, so the
operator integrates once instead of a hundred times. Costs a slice of GGR and creates a hard dependency on
one counterparty.
_Avoid_: content provider, hub

**PAM** (Player Account Management):
The platform component owning player accounts, the wallet, bonuses and compliance state. The actual product in
a "casino platform" — the back office is only a client of it.
_Avoid_: the platform, the backend

**Catalogue**:
The set of games available to a brand, modelled as `aggregator → studio → game → category`. The operator
merchandises the catalogue (featuring, ordering, categorising) but does not choose its contents; the aggregator
does.

**Geo-restriction**:
A block preventing a specific game or studio from being played from a specific country. Sourced from the
aggregator per game, then enforced by the operator. Compliance data, not application logic.

**Demo mode**:
Play with no wallet involvement and no real stake. Expressed at the contract level by omitting the player token
and passing a null currency, so no wallet call ever happens.
_Avoid_: fun mode, practice mode

### Money

**Wallet**:
A player's balance in one currency. A player may hold several. Not a ledger — the wallet is a derived balance;
the ledger is the truth.

**Ledger entry**:
An immutable double-entry record of a single movement of value. Every balance change has one. Balances are
computed from entries, never edited in place.
_Avoid_: transaction (too overloaded — say which kind)

**Banking transaction**:
A movement of value between the player and the outside world: a deposit or a withdrawal.

**Casino transaction**:
A movement of value between the player and a game: a bet, a win, or a rollback.

**Round**:
One unit of play at the RGS, identified by the studio and explicitly closed by it. A round may contain several
bets and wins — free-spin chains and re-spins are the reason a round is not the same thing as a bet.

**Game session**:
A player's continuous span of play on one game, with an identity, a start, an end and a status. Coarser than a
round and finer than a player's history; it is the level at which per-game money aggregates are naturally held.
ARC models neither this nor the round. MBO models the session — with bet, win, GGR and balance each split into
total, real and bonus — but no round entity is evidenced.

**Bet** / **Win**:
The debit taken when a player stakes, and the credit paid when they are paid out. A win always references the
bet it settles.

**Rollback**:
The reversal of a bet or win the RGS no longer considers valid, usually after a timeout or a failed call. A
routine, frequent path — not an error case, and it must be idempotent.
_Avoid_: refund, void, cancel

**Seamless wallet**:
The integration model where the player's balance stays in the operator's wallet and the RGS calls the
operator's API in real time for every bet and win. The modern default, and the endpoint where correctness is
money.
_Avoid_: single wallet, transfer-free

**Transfer wallet**:
The legacy model where funds are pushed into a per-session game wallet and pulled back afterwards.

**Minor units**:
The integer representation all money uses. Floats are not permitted anywhere in the money path; the wallet
contract's own scaling factor defines the unit.

**PSP** (Payment Service Provider):
A third party that moves real money in or out — card acquirer, e-wallet, bank rail or crypto processor. A brand
runs many, per market.

**Gold Coins / Sweeps Coins** (GC/SC):
The US sweepstakes model's two-currency scheme: Gold Coins have no redeemable value, Sweeps Coins do. It exists
to operate outside gambling licensure, and it is a genuinely different product from real-money play sharing the
same codebase.

### Bonus

**Bonus definition**:
The operator-configured offer — type, percentage, validity window, target segment, per-currency terms. A
template, not something a player holds.
_Avoid_: bonus, promotion

**Bonus instance**:
One player's individual grant of a bonus definition, with its own state and wagering progress. The thing that
moves through `Active → Claimed → Forfeit`.
_Avoid_: bonus, user bonus

**Wagering requirement**:
The total that must be staked before bonus value becomes withdrawable, expressed as a multiple.
_Avoid_: playthrough, rollover

**Wagering contribution**:
The percentage of a stake on a given game that counts toward the wagering requirement. Per game, because
low-margin games would otherwise clear a bonus for free.
_Vendor term_: MBO calls it **bet percentage** (`betPercents`) and configures it as a
`game studio × game type → percent` matrix rather than per individual game.

**Forfeit**:
Termination of a bonus instance before its wagering requirement was met, voiding the bonus value.

**Bonus liability**:
The outstanding value of un-cleared bonus funds across all players — a real obligation, and a first-class
report rather than a nice-to-have.

### Compliance

**KYC** (Know Your Customer):
Identity verification of a player against documents, either through a vendor or a manual review queue. Gates
withdrawal, and sometimes play.

**Responsible gambling**:
The player-protection controls a regulator expects: deposit, loss and wager limits over daily/weekly/monthly
windows, plus self-exclusion.
_Avoid_: RG (spell it out in prose), player protection

**Self-exclusion**:
A player-initiated block on their own account for a fixed period. Distinct from an operator-initiated
suspension, and **must not be reversible by the player on demand** — that is the entire point of it.

**Limit**:
A player-level cap on deposit, loss or wager over a rolling window. Set by the player or imposed by the
operator; tightening applies immediately, loosening must not.

**Licence**:
The regulator's authorisation to offer gambling to a market. Held by the vendor under white label and by the
operator otherwise. Its *calendar*, not integration work, is the real constraint on any launch date.

**Certification**:
Independent technical testing of the platform against a standard — GLI-19 for internet gaming systems, GLI-11
for the RNG and game devices. Required by most regulators before go-live.

### People

**Player**:
An end customer who deposits and plays.
_Avoid_: user, customer, punter

**Staff**:
An employee with back-office access, governed by the permission matrix.
_Avoid_: admin, user, operator (an operator is a company, not a person)

**Segment**:
A rule-defined set of players used to target bonuses and campaigns. Computed from player attributes and
behaviour, not hand-maintained.

**Tag**:
A free-form label attached to a player by staff. Manual, unlike a segment.
