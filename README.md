# Casino Management White-Label Platform — Investigation

Research into building an own-brand casino management white-label platform, based on hands-on investigation of
the **GammaStack "ARC"** sandbox (back office + player site) and of a second vendor's **"MBO" Modular
Back-office**, plus market and technical research.

Started 2026-07-31.

---

## Start here

| If you want… | Read |
|---|---|
| **The answer to build-vs-buy** | **[`BUILD-ESTIMATE.md`](BUILD-ESTIMATE.md)** §7 |
| The whole picture, synthesized | **[`FINDINGS.md`](FINDINGS.md)** |
| What the words mean | [`CONTEXT.md`](CONTEXT.md) |
| What's still missing and what to do next | **[`NEXT-STEPS.md`](NEXT-STEPS.md)** |

## Reference reports

| File | Contents |
|---|---|
| [`BUILD-ESTIMATE.md`](BUILD-ESTIMATE.md) | Build-side cost and effort: three scope tiers, effort by component, team shape, non-engineering costs, break-even against a white-label revenue share, and a recommendation |
| [`DATA-MODEL.md`](DATA-MODEL.md) | The reconstructed domain model — entities, relationships, state machines, the money model — with every claim marked `[observed]` or `[inferred]` |
| [`CONTEXT.md`](CONTEXT.md) | Glossary. GGR vs NGR, aggregator vs studio vs RGS vs PAM, seamless vs transfer wallet, and the rest |
| [`research/01-white-label-landscape.md`](research/01-white-label-landscape.md) | 15 vendors, white-label vs turnkey vs ownership, published pricing, feature matrices |
| [`research/02-game-provider-integration.md`](research/02-game-provider-integration.md) | Game supply chain, seamless-wallet API contract, launch flow, certification and licensing |
| [`research/03-fundist-integration.md`](research/03-fundist-integration.md) | The aggregator this platform actually uses, reconstructed from four independent operator implementations, and diffed against Hub88 |
| [`research/04-back-office-capability-benchmark.md`](research/04-back-office-capability-benchmark.md) | Two live back offices measured against report 01's marketing-derived matrix. Is any of it new? No — with one exception, and one verified absence that matters |
| [`src-analysis/MBO-BACKOFFICE-ANALYSIS.md`](src-analysis/MBO-BACKOFFICE-ANALYSIS.md) | **The second vendor.** 30-module micro-frontend back office: five-level tenancy, 715-permission RBAC, game-session and real-vs-bonus money model, and a direct diff against ARC |
| [`src-analysis/ADMIN-BUNDLE-ANALYSIS.md`](src-analysis/ADMIN-BUNDLE-ANALYSIS.md) | Back-office SPA reverse-engineered: ~250 REST endpoints, RBAC model, domain enums, security |
| [`src-analysis/RETAIL-SOURCE-ANALYSIS.md`](src-analysis/RETAIL-SOURCE-ANALYSIS.md) | Player site: routes, Server Actions, game launch, 1,796-key i18n feature map |
| [`src-analysis/RETAIL-REGISTRATION-ANALYSIS.md`](src-analysis/RETAIL-REGISTRATION-ANALYSIS.md) | Registration, onboarding and responsible gambling — the highest-risk player flow, recovered from the bundle |
| [`src-analysis/CHAT-APP-ANALYSIS.md`](src-analysis/CHAT-APP-ANALYSIS.md) | The third app: chat SPA, socket protocol, cross-origin auth handshake, moderation |
| [`docs/adr/`](docs/adr/) | Decisions worth recording, starting with the read-only conduct rule |

`src-analysis/admin/`, `src-analysis/retail/` and `src-analysis/chat/` hold the downloaded source bundles.

## The findings that matter most

1. **The evidence says buy, not build — below roughly €1M monthly GGR it isn't close.** Break-even sits at
   €0.9M–1.9M of monthly GGR, because building saves you the *spread* between the white-label rate and your own
   aggregator rate — about 8 points, not the 20 on the invoice. At a 15% white label against a 15% aggregator
   quote, building never pays back at any volume. Those two percentages move the answer roughly six times more
   than the entire engineering estimate does, which is why the next step is three quotes rather than a build
   plan. Full reasoning and the five conditions that would flip it: `BUILD-ESTIMATE.md` §7.
2. **The unavoidable engineering is the seamless-wallet callback API — and the sandbox gets it wrong.** The
   studio's game server calls *your* backend for every bet and win. It must be idempotent on a provider-supplied
   key, use integer money, and model an explicit round lifecycle. ARC does **none of those three**: there is no
   round entity, no game-session entity, and nothing corresponding to a provider idempotency key. Fundist
   compounds it by sending decimal strings rather than scaled integers and by expressing rollbacks as a flag on
   an ordinary transaction rather than a distinct operation. This is where correctness is money.
3. **Game supply runs through one aggregator, and that aggregator is also a PAM.** The back office has a single
   aggregator row — SoftGamings (Fundist) — with ~105 studios behind it. But Fundist is not merely an
   aggregator: it keeps its own player record, so an operator integrating it **shadow-registers every player
   into the vendor's system** and maintains a credential for each. That is invisible from the wallet API and is
   the largest structural difference from the Hub88 pattern.
4. **The second vendor closes ARC's central gap, which raises the bar for "parity".** MBO models an explicit
   **game session** — UUID, start, end, status — and splits every money aggregate into real versus bonus at the
   session level, propagating that split through every report. That is precisely the entity finding #2 records as
   missing from ARC. It also ships a five-level tenancy tree (Root → Root_folder → Project → Folder → Hall) with
   every page bound to a scope level, and a 715-permission RBAC matrix across 30 independently deployed
   micro-frontends. None of this moves the break-even arithmetic — that turns on the rev-share spread — but it
   is a second data point that the buy side is buying something substantial, and it is a better reference design
   for the wallet model than ARC. Detail: `src-analysis/MBO-BACKOFFICE-ANALYSIS.md`.
5. **Nothing either platform does is new — and the incumbent gap is AI.** Benchmarked feature by feature against
   the vendor landscape, every MBO capability maps to an established category; the only arguably original thing
   is a *commercial* mechanism, serving the module set as a runtime import map so licensing is enforced at page
   load. The finding that matters is an absence: searching all 30 bundles turns up **no AI or ML anywhere** — no
   recommendation engine, no churn or propensity prediction, no anomaly-based fraud scoring (the anti-fraud
   module is rules and reports). Meanwhile AI has become a primary axis of vendor comparison in 2026. That is the
   first argument this investigation has found for building that is about **capability rather than cost**:
   competing where incumbents are weakest beats replicating fifteen years of back-office breadth. Full benchmark:
   `research/04-back-office-capability-benchmark.md`.
6. **The licensing fork decides more than the price.** White label means operating on the vendor's licence.
   Turnkey and ownership require your own — and the licensing calendar, not integration work, is the real
   bottleneck. Curaçao's regulator lost its entire supervisory board and the minister behind its reform in late
   2025, against an application backlog implying well over a year of clearance.

## Scope and conduct

Investigation was **read-only**, against both vendors. Forms were opened to read their fields; nothing was
submitted, no deposits or bets were placed, and no data was modified in either sandbox. Game launch was exercised
in DEMO mode only. No direct API calls were made with an admin session token — on MBO the app's own traffic was
observed by instrumenting the page rather than by driving the API. See `NEXT-STEPS.md` §4, which is an open
decision, and `docs/adr/0001-read-only-investigation-conduct.md`.

Security observations in these documents are **incidental findings, not a security audit**. No authorization
testing, input fuzzing, or cross-tenant access was attempted, and none should be without the vendor's written
permission.

Credentials live in `.env` (not committed).
