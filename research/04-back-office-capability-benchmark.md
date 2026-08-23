# Back-Office Capability Benchmark — Is Any of It New? (2026)

`research/01-white-label-landscape.md` §4 builds its feature matrix from vendor marketing and then says plainly
that gaps "generally mean *not mentioned in the marketing materials reviewed*, not necessarily absent", and that
the matrix "should be validated against a live back-office demo before being used as a hard feature matrix."

This report is that validation. Two vendor back offices have now been used hands-on:

- **ARC** (GammaStack/GammaPlus) — `src-analysis/ADMIN-BUNDLE-ANALYSIS.md`, investigated 2026-07-31/08-01
- **MBO** (vendor unidentified, `mbo.kit.casino`) — `src-analysis/MBO-BACKOFFICE-ANALYSIS.md`, 2026-08-23

The question this report answers is the one worth asking before paying for a platform: **beyond breadth, is there
anything genuinely new here — and if not, what actually differentiates one of these from another?**

## Evidence grade — read this before using the comparisons

There is an **asymmetry in this report that cannot be removed without vendor demos**:

- Claims about **MBO and ARC** are first-hand. Screens were opened, traffic observed, bundles read.
- Claims about **every other vendor** are marketing-derived, inherited from report 01 or from the sources at the
  end of this file. They carry report 01's caveat unchanged.

So "MBO has X and SoftSwiss does not" is never a safe reading of anything below. The safe reading is "MBO has X,
verified; whether SoftSwiss does is unestablished." The one place this asymmetry does **not** bite is the AI
finding in §3, because there the direction is reversed: competitors *advertise* the capability and MBO
*verifiably lacks* it.

---

## 1. Verdict: nothing here is a new product idea

Every capability in MBO maps to an established category. Assessed feature by feature:

| Capability in MBO | Industry status | New? |
|---|---|---|
| Site/lobby builder ("Casino Constructor") | Lobby + CMS is standard above the template tier — SoftSwiss ships "game lobby" and CMS in one stack; EveryMatrix is bought specifically for "lobby management and granular content control at scale across multiple brands"; Digitain's Centrivo lists lobby and CMS as core | **No.** But see §4 — MBO's is deeper than a merchandising CMS |
| Draft → approval → publish on design assets | Ordinary CMS editorial practice, borrowed | No |
| Real-vs-bonus money separation | A regulatory requirement in licensed markets, not a feature | **No** — ARC's *failure* to do it is the anomaly |
| Supplier-cost reporting (credit consumption) | Uncommon in marketing, but any operator at scale reconciles aggregator invoices | No, though under-advertised |
| Five-level tenancy tree (Root→…→Hall) | Concept is old — inherited from land-based/street-terminal operations where a "hall" was a physical venue | No. Notable only because it is *visible* — see §4 |
| Gamification: quests, wheel, cashback, tournaments | The 2020-23 wave, now universal. Report 01 §5 already lists "spin the wheel"/raffle mechanics as commonly advertised; Pariplay productises them as add-ons | No |
| Dual-track loyalty (tiers + XP) | Standard, imported from video games | No |
| Crypto, crypto-fiat, wallet networks | Report 01 §5: "close to table-stakes among the newer/challenger vendors" | No |
| Telegram Casino as a module | Timely — Telegram mini-app casinos scaled through 2024-25 — but several CIS-facing platforms shipped integrations | **Current, not new** |
| Chat moderation pipeline | Generic trust-and-safety tooling | No |
| CRM journey campaigns (multi-step action iterations) | Ordinary marketing automation, brought in-house | No |
| 715-permission RBAC | Scale is unusual; RBAC is not | No |
| Keycloak OIDC + PKCE, federated staff SSO | Standard enterprise IAM | No |
| PII masked by default in back office | Ordinary data-protection practice, under-adopted in iGaming | No |

**The one arguably original thing is a commercial mechanism, not a product feature.**

MBO serves its module set as a **runtime import map** (`GET /api/services/import-map`), so which business domains
a deployment runs is a server-side decision, applied at page load, with no front-end release to enable or revoke
one. single-spa is 2018 technology and micro-frontends are a well-understood pattern — but using them so that the
*packaging and licensing* of the product is data is a deliberate and unusually clean piece of commercial
engineering. EveryMatrix markets the same *outcome* ("combine modules tailored to their needs"); MBO is the only
platform where the enforcement mechanism has actually been seen.

Eleven of MBO's thirty deployed modules are unreachable by the account under test, and the permission taxonomy
confirms they are real product. That is this mechanism working as designed.

---

## 2. What "new" is worth, and where it isn't the right bar

For a system that moves money, novelty is close to the wrong criterion — correctness and operational maturity
matter more, and MBO is markedly more correct than ARC (`src-analysis/MBO-BACKOFFICE-ANALYSIS.md` §9).

But novelty matters in one specific commercial way: **it determines whether a platform's advantage is defensible.**
MBO's advantage is depth of implementation. Depth erodes when a competitor with comparable breadth adds a
capability MBO structurally lacks. Which brings us to the one that matters.

---

## 3. The finding that actually differentiates: MBO has no AI, and the market has moved

**Verified absence.** All thirty MBO bundles were searched for `recommend`, `machine learning`, `openai`,
`artificial intelligence`, `personaliz`, `LLM`, `anomaly`, `propensity`, `churn`, `next best`, `autoML`,
`tensorflow`, `embedding`. Every apparent hit dissolves on inspection
(`src-analysis/MBO-BACKOFFICE-ANALYSIS.md` §8.6):

- `churn` is a churn-**rate** metric with a formula, beside retention rate and ARPU — descriptive, not predictive
- `personaliz` is Spanish/Italian UI text from the bundled Jodit rich-text editor
- `openai` is an Ant Design **icon name**, plus `OpenAIModule` in the shell registry which is a **misspelling of
  OpenAPI** (the same bundle carries `isForOpenAPI`; the domains table has a "For OpenAPI" column; the permission
  taxonomy has an "OpenAPI Content/FAQ/SEO" category)
- the **anti-fraud module contains no anomaly detection** — it is reports plus rule-based duplicate settings

So: no recommendation engine, no churn or propensity prediction, no anomaly-based risk scoring, no personalisation
engine, no LLM anything.

**Meanwhile the market treats these as baseline.** Report 01 §4 already records **GR8 Tech** advertising an
AI-driven bonus engine, AI marketing CRM, real-time anti-fraud and intelligent payment routing, and **Altenar**
advertising AI risk and fraud — the only two AI annotations in that matrix, written before this benchmark. The
2026 picture is broader still: game recommendation engines are sold as named products (iGP), Fast Track markets
churn prediction claiming up to 92% accuracy with preventive triggers up to 48 hours ahead, Smartico sells
AI-driven CRM and gamification, and trade coverage now frames 2026 as the year of "platform-wide AI
architectures" with AI-driven offers expected to exceed 20% of revenue among leaders. There is even a published
comparison of iGaming providers ranked **by AI capability** — which is itself the signal: AI has become a primary
axis of vendor comparison, not a differentiator claim.

**The irony worth recording:** ARC — weaker than MBO on essentially every other axis — at least declares an
`/ai/` namespace in its API (`src-analysis/ADMIN-BUNDLE-ANALYSIS.md` §2.1). MBO has nothing.

Treat the vendor claims above with report 01's scepticism: "AI-driven" in gambling marketing frequently means a
rules engine with a good adjective. The asymmetry still holds, because MBO's absence is verified while the
competitors' presence is merely claimed — but the *right* diligence question is not "do you have AI", it is
"show me the model, its features, its accuracy, and how it retrains."

---

## 4. Where MBO exceeds what report 01 could see

Three capabilities are materially better in MBO than the landscape report's marketing-derived matrix suggests any
vendor offers — not because MBO invented them, but because report 01 could not see inside anyone's back office.

**Multi-tenancy.** Report 01 §6 concluded that "only Playtech's IMS/PAM+ explicitly advertises multi-vertical
single-account architecture; other vendors imply per-brand isolation without detailing tenancy model." MBO's model
is now fully visible: a five-level tree with **every back-office page bound to a required node level**, refusing to
render until a node of that type is selected — 25 pages Hall-scoped, 16 Project-scoped, 8 Root-scoped, 2
Root_folder-scoped. Configuration inherits downward (`Parent Currencies`, `Parental languages`, `Manage Sub Trees
Node Structure`). The **Multi-brand column in report 01 §4 should be read as unestablished rather than absent** —
it was measuring marketing copy, and at least one vendor has far more than it advertises.

**Permission governance.** 715 permissions as resource × action, across 22 categories, with 11 default roles — and
a **"new permissions" counter** that surfaces permissions introduced since a role was last reviewed, so a platform
upgrade cannot silently widen or narrow what staff can do. That last detail is standard practice in enterprise IAM
and rare in iGaming; it is the single most mature thing found in either back office.

**Supplier-cost reporting.** The Credit Consumption report breaks spend down by game API provider × game provider
× currency into real money, manual bonuses and automatic bonuses — what the operator **owes the aggregator**. Most
back offices report revenue. Reporting supplier cost beside GGR is how an operator sees true margin, and it is the
kind of capability vendors under-advertise because it is unglamorous.

---

## 5. Implications for build-vs-buy

None of this moves the arithmetic in `BUILD-ESTIMATE.md` §7 — that turns on the spread between the white-label
rate and an own aggregator rate, not on feature counts. It changes three softer things:

1. **It raises what "parity" costs.** A five-level tenancy tree with per-page scope binding and inherited
   configuration, a 715-entry permission matrix threaded through 30 modules, a page-builder that publishes a live
   player site, and a metadata-driven settings engine are all large, unglamorous infrastructure that never appears
   on a feature-list comparison and quietly consumes quarters of engineering time. Any build plan that treats
   multi-brand tenancy as a column on a table is wrong.
2. **It tells you what to negotiate on.** If nothing is novel, then breadth is commoditised and price and terms
   are the real levers — reinforcing report 01 §7 and `BUILD-ESTIMATE.md` §7's conclusion that three quotes beat
   a build plan.
3. **It gives the build case one genuine opening.** AI-driven retention and risk are where the market is
   differentiating in 2026, and at least one mature incumbent platform has none of it. A build that treats
   recommendation, churn prediction and anomaly-based risk as first-class from day one would be competing where
   the incumbents are weakest rather than replicating fifteen years of back-office breadth. That is not by itself
   an argument to build — but it is the first argument found in this investigation that is about *capability*
   rather than *cost*.

---

## 6. Open questions this benchmark could not close

1. **What is "Predictor"?** A deployed 2.2 MB MBO module (`bet-trade`) the test account cannot open. If it is
   prediction markets, it is the one forward-looking thing in the stack and would change §1's verdict.
2. **What is `trs`?** MBO's largest unreachable module at 6.8 MB; only visible endpoint is a scheduled-job API.
3. **`RegulatorsReportsModule`** is declared in MBO's shell registry but has no deployed bundle and no permission
   category in this tenant. Regulatory reporting is a real compliance capability — is it a licensable module?
4. **Do the AI claims of GR8 Tech, Altenar, Fast Track and Smartico survive contact with a demo?** Every one is
   marketing-sourced. The diligence question is in §3.
5. **How deep is anyone else's site builder?** MBO's publishes the whole player site; EveryMatrix's lobby CMS is
   described in terms of placing featured and promo games. These may not be the same class of tool, and the
   sources are too thin to say.

---

## Sources

Vendor and market claims in §1 and §3 are marketing- and trade-press-derived and carry report 01's caveat.
First-hand findings are cited to `src-analysis/` throughout.

- [AI in iGaming: Use Cases to Watch in 2026 — GR8 Tech](https://gr8.tech/blog/ai-in-igaming-a-look-into-machine-learning-and-personalized-gaming-experiences/)
- [AI personalization in iGaming 2026: smart odds, hyper-tailored UX and player safety — Yogonet](https://www.yogonet.com/international/news/2025/09/26/115546-ai-personalization-in-igaming-2026-smart-odds-hypertailored-ux-and-player-safety)
- [Best AI Casino Prediction Software in 2026 — Smartico](https://www.smartico.ai/blog-post/best-ai-casino-prediction-software-2025)
- [Predictive Game Recommendations — Smartico](https://www.smartico.ai/blog-post/predictive-game-recommendations-ai-curating-personal-igaming-journeys)
- [AI Game Recommendation Engine — iGP](https://igpgaming.com/game-recommendation-engine/)
- [Game Recommendation Engine — iGaming Platform](https://igamingplatform.com/game-recommendation-engine)
- [Best AI Tools for iGaming in 2026 — Playa Solutions](https://www.theplaya.solutions/post/best-ai-tools-for-igaming-in-2026)
- [iGaming Software: 12 Providers Compared by AI Capability — Playa Solutions](https://www.theplaya.solutions/post/igaming-software)
- [Online Casino Platform — SOFTSWISS](https://www.softswiss.com/casino-platform/)
- [Casino Lobby & Content Management Platforms — iGaming Platform Providers](https://igamingplatformproviders.com/casino-lobby-content-management-platforms)
- [Casino Platform Providers 2026 — casino.limo](https://casino.limo/casino-platform-providers)
- [EveryMatrix platform profile — btcgosu](https://www.btcgosu.com/crypto-casino-igaming-software-platforms/everymatrix/)
