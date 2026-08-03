# White-Label Casino Platform — MVP Design

**Date:** 2026-08-03
**Status:** draft, awaiting review
**Supersedes nothing.** Builds on `BUILD-ESTIMATE.md`, `DATA-MODEL.md`, `research/02`, `research/03`.

---

## 1. What we are building and why

A **multi-tenant white-label casino platform**, run on sandbox integrations, that one small team can build and
operate. When a client signs, they get their own domain, their own branding and their own data — served by one
shared deployment.

This is a **platform-vendor product**, not a casino. We are not applying for a gambling licence, not taking real
money, and not operating a brand. That is a deliberate scope decision, and it removes the entire non-engineering
calendar — licence applications, PSP underwriting, mandated compliance staffing — which `BUILD-ESTIMATE.md` §6
identifies as the real bottleneck on any operator build.

### 1.1 Success criteria

The MVP is done when it demonstrates three things. These were chosen explicitly; a fourth (bonus and wagering
mechanics) was considered and excluded.

1. **The money path survives abuse.** A player bets and wins; the ledger stays correct under retry storms,
   duplicate transaction ids and 500-retry rollbacks; reconciliation proves it.
2. **A player journey end to end.** Register, verify identity against a sandbox KYC vendor, deposit, play a real
   game, watch the balance move, request a withdrawal, staff approves it.
3. **The operator experience.** Back office: player 360, transaction search, manual balance adjustment,
   withdrawal queue, game merchandising.

Criterion 1 is the one that cannot be faked and the one the whole architecture is arranged around.

### 1.2 Scope

**In:** tenancy and per-tenant theming; identity, auth and staff RBAC; wallet and ledger; the seamless-wallet
callback API; an adversarial RGS simulator; game catalogue and launch; a player front end; sandbox KYC; sandbox
payments with a withdrawal queue; reconciliation; back office; basic responsible-gambling limits; audit logging.

**Out, deliberately:** the bonus and wagering engine beyond a trivial deposit credit; tournaments; loyalty and
VIP; spin wheel; referrals; cashback; segmentation and campaign targeting; CMS beyond static legal pages; live
chat; risk and fraud tooling; sportsbook; in-house games; affiliate module; tenant billing and rev-share
accounting; provisioning self-service; real money of any kind.

**A note on the bonus engine.** It is excluded because it is the deepest rabbit hole in the product — 20–32
person-months at full scope, combinatorial rather than additive (`BUILD-ESTIMATE.md` §2.1). But **the ledger
must be designed for it now**: every money row carries a separate bonus amount, and the wallet distinguishes
cash from bonus balance from day one. Retrofitting bonus-aware accounting into a live ledger is the expensive
mistake this spec is trying to avoid.

---

## 2. Architecture

### 2.1 The shape: pooled compute, isolated data

One AWS account for production. One cluster. One deployment of the application, serving every tenant. But **a
separate Postgres database per tenant**, several of them hosted on shared RDS instances.

Database-per-tenant is not instance-per-tenant. That distinction is the whole design: physical data separation
without multiplying the infrastructure bill.

| Layer | Shared | Per tenant |
|---|---|---|
| Compute, cluster, app deployment | ✅ | |
| CDN, WAF, DDoS protection | ✅ | |
| Game catalogue cache, provider adapters | ✅ | |
| **Database, including the ledger** | | ✅ |
| KYC documents in S3 | | ✅ separate prefix, own KMS key |
| Secrets, PSP credentials | | ✅ |
| Config, theme, feature flags, domain | | ✅ |

**Why not a separate stack per client.** A tenant with fifty players costs nearly the same as one with five
thousand: Multi-AZ RDS, ElastiCache, NAT gateways at $250–450/month, and — because gambling was the
most-attacked industry for DDoS in Q1 2025 — protection that is not optional. AWS Shield Advanced is $3,000 per
month *per account*. Ten siloed clients is roughly $27–35k/month, mostly idle; the same ten pooled fit in
$3–5k/month. The operational argument is stronger still: two people cannot run ten AWS accounts.

**Why not a shared database with a `tenant_id` column.** We are holding player money. A tenant-scoping bug in a
shared table is not a data leak, it is one operator's players spending another operator's money — a financial
incident threatening both parties' licences. Separate databases make that class of bug impossible rather than
unlikely. It also buys per-tenant backup and point-in-time restore, a straight answer to the due-diligence
question every operator asks, and a clean exit: we can hand a departing client a dump. `BUILD-ESTIMATE.md` §6.5
notes vendors deliberately leave data-migration exit fees unpublished because that is the lock-in — **offering a
clean exit is a genuine differentiator for the small challenger.**

The cost is that migrations run N times. Below roughly fifty tenants that is a loop in a deploy script.

### 2.2 Control plane and data plane

The **control plane** is one small shared database: the tenant registry mapping hostname to tenant, plus config,
feature flags, theme and database connection details. Read-heavy, cacheable, small.

The **data plane** is the pooled services plus the per-tenant databases holding players, ledger, transactions and
documents.

The tenant registry is the only shared table in the system. Keeping it deliberately small is what stops pooled
multi-tenancy from decaying into shared everything.

### 2.3 Deployables

One codebase, one container image, several entrypoints.

| Deployable | Responsibility | Why separate |
|---|---|---|
| `wallet-callback` | The five callback endpoints and wallet core | Independent availability and scaling; hot path of every spin |
| `api` | Player and admin REST; modular monolith internally | |
| `worker` | Reconciliation, catalogue sync, KYC polling, scheduled jobs | Failure isolation from request path |
| `web-player` | Next.js player site, all tenants | |
| `web-admin` | Back office | |

The `wallet-callback` split is the only one justified on day one, and it is justified by **availability, not
domain boundaries**: if a bad admin deploy takes down the back office, bets must keep settling, or studios begin
pausing traffic to us.

Precedent worth noting: the ARC admin bundle presents twenty microservice-style namespaces, and `DATA-MODEL.md`
established those are routing prefixes on **two actual services**. Even the incumbent runs a near-monolith
behind a microservice-shaped URL scheme.

**Do not start on Kubernetes.** ECS Fargate gives the same container boundary at a fraction of the operational
surface. The container boundary is the portable part — if a client later requires deployment into their own AWS
account, containers plus Terraform travel fine, and EKS remains available when there is someone whose job is to
run it.

### 2.4 Tenant resolution

```
Host header → edge → resolve hostname to tenant (cached registry)
            → middleware sets tenant context (AsyncLocalStorage)
            → acquire connection from that tenant's pool
            → handle request
```

**Fail closed.** No resolved tenant means reject the request. There is no default tenant and no fallback. A
query must never execute without tenant context — enforce this in the data-access layer, not by convention.

### 2.5 Connection pooling — the known failure mode

Database-per-tenant has one genuinely nasty operational trap: **connection exhaustion**. Tenants × pods × pool
size multiplies quickly against a Postgres `max_connections` ceiling of a few hundred. Fifty tenants, six pods
and ten connections each is 3,000 connections.

Required from the start, not later:

- **RDS Proxy or PgBouncer in transaction mode**
- Connections acquired **lazily per request**, not pools held open per tenant
- **Per-tenant pool caps**, so one busy client cannot starve the others

Retrofitting this under load is miserable. Designing for it costs nothing.

---

## 3. Sites, theming and custom domains

### 3.1 Layout is code, theme is data

A **layout** is a structural template — a Next.js route group. There will be about three. Adding one is a
development task and a product decision we make.

A **theme** is per-tenant configuration: colours, logo, fonts, imagery, copy, enabled features. Editable in the
back office, live without a deploy.

"Slightly customize" therefore has a precise meaning: **pick a layout, configure everything inside it.**

This mirrors ARC, which serves `/en/casino` by internally rewriting to `/en/layout2/casino` — captured directly
as an `x-middleware-rewrite` response header during the investigation.

### 3.2 What a client changes without us

Logo, favicon and brand name; colour palette as CSS custom properties; typography from a curated set; hero and
banner imagery; **which routes exist at all**, via feature flags; navigation structure and ordering; CMS pages
per locale (terms, privacy, responsible gambling); enabled currencies, languages and payment methods; game
categories, featured games and merchandising order.

Feature flags gate **whole routes**, not menu items. ARC does this correctly — `/vip` and `/promotions` return a
genuine 404 on tenants without them rather than being hidden with CSS.

**Requires us:** structural layout changes, new page types, novel components.

**Make that boundary contractual, not merely technical.** The failure mode for a small platform vendor is
commercial rather than architectural: every client asks for one small change, we say yes because they are
paying, and eighteen months later we maintain nine bespoke forks and have become a consultancy with a hosting
bill.

Theming must stay **runtime**, injected as CSS custom properties. The moment a tenant needs its own build, one
deployment stops serving everyone and the economics collapse.

### 3.3 Custom domains

The client CNAMEs their domain to us. TLS is the awkward part — a certificate per customer domain, issued and
renewed automatically, which is real work with ACM given per-distribution alternate-name limits.

**Use Cloudflare for SaaS custom hostnames.** It is purpose-built for this, handles issuance and renewal per
client domain, and we are already behind Cloudflare. Domain onboarding becomes an API call in the provisioning
script rather than an engineering ticket.

### 3.4 The highest-severity bug in this design

**CDN cache keys must include the hostname.** Two tenants request `/casino`; if the cache key is the path alone,
tenant B is served tenant A's rendered page — one brand's players seeing another brand's site, potentially with
personalised content.

It is silent until it is catastrophic, and it is one line of cache configuration. Set it on day one and write a
test that requests the same path on two hostnames and asserts the responses differ.

The quieter cousin is **content bleed**. The investigation found the string `Tower.bet` in ARC's live
translation catalogue on an ARC-branded tenant, along with a hardcoded `Earn ₹150 each` from an India
deployment. That is shared content templates without per-tenant override discipline. We need per-tenant
overrides falling back to defaults, and **a test that fails if any tenant's rendered copy contains another
tenant's brand name.**

---

## 4. The money path

This is the part that must be right. Everything else is recoverable.

### 4.1 Ledger

- **Append-only entries.** Every balance change is an immutable row. Balances are derived, never updated in
  place.
- **Integer minor units** (`bigint`). No floats anywhere in the money path, at any layer, ever.
- **Double-entry with explicit parties.** Both banking and casino transactions carry `from` and `to` rather than
  a signed delta — matching what `DATA-MODEL.md` recovered from ARC, which is a transfer model and the right
  foundation.
- **Cash and bonus amounts are separate columns on every money row.** One transaction can touch both.
- **A continuously verified invariant** that entries sum to the balance — a job, not an assumption.

`DATA-MODEL.md` recovered ARC's transaction type enum verbatim, in two parallel tracks:
`CasinoBet / CasinoWin / CasinoRefund` and `CasinoBonusBet / CasinoBonusWin / CasinoBonusRefund`. We adopt the
same shape.

**Open design question to resolve in Phase 1:** the cash-versus-bonus debit order on a mixed-funds wager. This
directly determines bonus cost and could not be recovered from ARC. Decide it explicitly, document it, and make
it configurable per tenant.

### 4.2 Rounds and idempotency — where ARC gets it wrong

`DATA-MODEL.md` established that ARC has **no game round entity, no game-session entity, and nothing
corresponding to a provider-supplied idempotency key**. `roundId` appears zero times in the admin bundle.

That is the single highest-risk finding of the whole investigation, and **we do the opposite**:

- **A `round` entity** with explicit open and close. A round may contain several bets and wins — free-spin
  chains and cascading wins are why a round is not a bet.
- **A unique constraint on `(provider, provider_transaction_id)`** doing the idempotency work. Idempotency is a
  database constraint, not application logic.
- **Wins reference the bet they settle.**
- **Rollback is a first-class, frequent, idempotent path** — not an error case.

The design constraint is Hub88's documented retry policy: bet and win retried **3× with a 1-second timeout**,
rollbacks retried **up to 500 times with exponential backoff**. The endpoint must be safely re-callable
indefinitely and must distinguish a legitimate retry from a genuine duplicate carrying different details.

### 4.3 The aggregator adapter

`research/03` §11 concluded a clean adapter is achievable for catalogue, launch and wallet, but **not** for
player provisioning. We draw the seam accordingly.

**Inside the adapter:** catalogue normalisation, launch-parameter construction, signing and verification,
wallet-callback translation, money-format conversion.

**Outside it:** player provisioning. Fundist requires `User/Add` and a persistent per-player login and password;
Hub88 requires nothing. That is a vendor-specific lifecycle hook invoked at registration, not part of a
game-supply interface.

**Two traps to encode as tests now:**

1. **Fundist expresses rollback as an `i_rollback` flag on an ordinary debit or credit, with inverted sign** —
   not as a distinct operation. A naive adapter with a `{bet, win, rollback}` interface will silently corrupt
   balances rather than fail loudly. Write the test that feeds a `type: "credit"` payload carrying `i_rollback`
   through the adapter and **asserts the player's balance decreases.**
2. **Fundist sends decimal strings, Hub88 sends integers scaled ×100,000.** Convert at the adapter boundary.
   Never let a vendor's decimal reach the ledger.

### 4.4 The RGS simulator

Not a stopgap — **the centrepiece of Phase 1 and a permanent part of the test infrastructure.**

No aggregator will issue sandbox credentials without a contract, and there is a trap in the obvious workaround:
**demo-mode games never call the wallet.** Per `research/02` §3.4, demo mode means passing a null currency and
omitting the player token precisely so no wallet callbacks occur. Embedding public demo game URLs produces a
convincing lobby that exercises exactly zero of the money path.

So the simulator plays the counterparty, written against the Fundist shape in `research/03` and the Hub88 retry
policy in `research/02`. It must be able to attack us with:

- Bet and win retried 3× within the timeout
- Rollbacks retried up to 500× with backoff
- Duplicate transaction ids with **identical** details (legitimate retry — must be absorbed)
- Duplicate transaction ids with **differing** details (genuine conflict — must be rejected)
- Rollbacks arriving before the transaction they reverse
- Rollbacks for transactions that never existed
- Multi-leg rounds with cascading wins
- Concurrent bets on one wallet
- Connection drops mid-transaction

**Phase 1 exit criterion:** the simulator runs a sustained hostile session and the ledger reconciles to the
last minor unit.

---

## 5. Sandbox integrations

| Concern | Approach | Notes |
|---|---|---|
| **Game supply — money path** | Our RGS simulator | The only way to exercise bet/win/rollback under retry storms |
| **Game supply — lobby** | Public demo-mode titles | Real games, real launch flow, **no wallet involvement** |
| **Game supply — real** | Ask an aggregator for sandbox access in parallel | With a Curaçao licence we are a credible prospect; worst case is no |
| **KYC** | Sumsub or Veriff sandbox | Genuinely available: Sumsub 14 days / 50 checks, Veriff 15 days / 50 verifications |
| **Payments** | Stub PSP implementing our provider interface | Deposit, withdrawal, callbacks, deliberate failure injection |

The stub PSP matters more than it sounds: it forces the provider abstraction to exist from the start, so adding
a real PSP later is an implementation of an interface rather than a refactor.

---

## 6. Stack

TypeScript end to end. Fastify or NestJS for services, Next.js for both front ends, Postgres, Redis, ECS
Fargate, Terraform, Cloudflare.

Two reasons specific to this team: one language for two people, and mainstream choices measurably improve
AI-assisted output, which is a real input when tooling is doing this much of the work. It also matches the
industry — ARC's player site is Next.js.

Postgres because the ledger wants genuine transactional integrity, unique constraints doing idempotency, and
exclusion constraints available when needed.

---

## 7. Phases

Each phase ends at a success criterion, so each is a decision point rather than a status report.

### Phase 1 — foundation and money path (3–5 months)

Tenant model and registry; per-tenant database provisioning; connection routing and pooling; identity, auth,
sessions; staff RBAC core; wallet and ledger; the five callback endpoints; the aggregator adapter interface with
one implementation; the RGS simulator and chaos harness; reconciliation; roughly six admin screens (tenant list,
player list, player detail, ledger view, transaction search, manual adjustment); CI, environments, observability.

**Exit:** criterion 1. Sustained hostile simulator session, ledger reconciles exactly. **If this phase goes
badly, stop.** The important thing has been learned for under €80k.

### Phase 2 — player journey (3–4 months)

Catalogue sync and merchandising; game launch, demo and real; player front end (register, login, lobby, game
frame, balance, history, account); sandbox KYC with document state machine and manual fallback; stub PSP with
deposit; withdrawal request queue with staff approval; responsible-gambling limits.

**Exit:** criterion 2. Register through withdrawal, end to end.

**Responsible gambling detail worth specifying now:** the player site enforces a **24-hour cooling-off on
loosening a limit**, while tightening applies immediately. `RETAIL-REGISTRATION-ANALYSIS.md` §10 recovered this
from ARC's own copy, and `DATA-MODEL.md` notes the corresponding pending-change and effective-at fields appear
**nowhere** in ARC's admin model. Model it properly: a limit change is a state machine, not a mutable value.

### Phase 3 — operator experience and theming (4–6 months)

Back-office breadth across the five route shapes; RBAC UI; reporting and exports; per-tenant theming and config
UI; CMS pages per locale; custom-domain onboarding automation; provisioning script hardening.

**Exit:** criterion 3.

**RBAC detail:** `DATA-MODEL.md` recovered **15 permission verbs, not 4** — alongside CRUD there are `MM`
(manage money), `UW` (user wallet), `I` (issue bonus), `E` (export) and others. Those first four are exactly
what a fraud or regulatory review cares about and must be separable from ordinary create/read/update/delete. A
model with four CRUD bits per module cannot express this.

---

## 8. Effort and budget

**49–79 person-months raw; 22–35 compressed.** Compression applied per component rather than as a blanket
factor: ~1.5× on wallet, callback and reconciliation (design and verification bound), ~2× on tenancy, ~2.5× on
integrations and infrastructure, ~3.5× on CRUD and UI.

| | Two people | One person |
|---|---|---|
| Calendar | **11–17 months** | 22–35 months |
| Cost at €4–8k per person-month | **€88k–280k** | same total, twice the calendar |

Add roughly €500–1,500/month for infrastructure and tooling during the build.

**Reconciling this with §7.** The three phase estimates sum to 10–15 months of focused delivery. The 11–17
month figure above carries the difference as slack for the things phase plans habitually omit: environment
setup before Phase 1 can start, the sandbox KYC and aggregator negotiations running in parallel, and the
integration friction between phases. If the phases are hit exactly, the lower bound is real.

**Note what compression does to the shape of the work.** Uncompressed, ledger and callback are about 20% of the
project; compressed they approach 30%, and with the simulator and reconciliation the four hard components are
**over 40% of remaining effort**. Claude Code does not make this a UI project — it makes it *more* of a
correctness project, because everything else shrinks around the part that does not.

---

## 9. Risks

| Risk | Mitigation |
|---|---|
| **Cross-tenant data exposure** | Database per tenant; fail-closed resolution; no default tenant; enforcement in the data layer |
| **CDN serves tenant A's page to tenant B** | Hostname in cache key; automated two-host test |
| **Connection exhaustion** | RDS Proxy / PgBouncer transaction mode; lazy acquisition; per-tenant caps |
| **Ledger corruption under retry** | Unique constraint on provider transaction id; simulator as permanent CI gate; continuous invariant check |
| **Adapter inverts a rollback sign** | Explicit test asserting balance *decreases* on `i_rollback` credit |
| **Content bleed between tenants** | Per-tenant overrides; test failing on foreign brand names |
| **Scope creep into per-client forks** | Layout/theme boundary stated contractually, not just technically |
| **No aggregator will contract with us** | Simulator makes the money path demonstrable regardless; pursue sandbox access in parallel |
| **Bonus engine demanded early** | Ledger is bonus-aware from day one, so it is additive rather than a rewrite |

---

## 10. Assumptions and open questions

1. **Tenant count year one assumed 3–10.** If it is closer to fifty: migrations need orchestration rather than a
   loop, provisioning must be self-service rather than scripted, and connection pooling needs attention in
   Phase 1 rather than Phase 2. **Please confirm.**
2. **Rates assumed €4–8k per person-month fully loaded**, the Ukraine/Poland senior band from
   `BUILD-ESTIMATE.md` §4.9. The single largest input to the budget.
3. **Team assumed two people.** One person doubles the calendar and makes Phase 1 a poor risk — the ledger
   benefits from a second pair of eyes more than any other component.
4. **Cash-versus-bonus debit order on mixed-funds wagers** is undecided and could not be recovered from ARC.
   Must be resolved in Phase 1.
5. **No client has been assumed to require a dedicated stack.** If one does — a regulator demanding residency,
   or a contractual requirement — that becomes a priced tier and changes Phase 3 ordering.
6. **Layout count assumed ~3 at launch.** More is a product decision with direct cost.

---

## 11. What this deliberately does not do

It does not make us a licensed operator, hold real player funds, or process real payments. It does not include
the bonus engine, the affiliate module, tenant billing, or sportsbook. It does not attempt parity with ARC —
that is 199–311 person-months and a different project.

It builds the part that is hard to get right, on top of a tenancy model that is expensive to retrofit, and
leaves everything else additive.
