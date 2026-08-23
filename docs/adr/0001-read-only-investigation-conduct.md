# Read-only conduct against the vendor sandbox

**Status:** accepted; extended 2026-08-23 to a second vendor (MBO, `mbo.kit.casino`)

This investigation runs against a live sandbox belonging to a third party (GammaStack), reached with credentials
they issued for evaluation. We decided the investigation is **read-only**: pages are loaded as a browser would
load them and published static assets are downloaded over HTTPS, but nothing is submitted, no deposit or bet is
placed, no record is modified, and game launch is exercised in DEMO mode only.

We also decided **not to call the admin REST API directly with the logged-in bearer token**, even though doing so
would pin down request and response shapes precisely and make the reconstructed data model authoritative rather
than inferred. That is the trade-off we accepted: a less precise model in exchange for staying inside the
conduct a vendor would recognise as evaluating their product.

## Consequences

- `DATA-MODEL.md` carries `[observed]` / `[inferred]` markers throughout, and field-level response shapes are
  unknown. This is a direct cost of the decision, not an oversight.
- Security observations in these documents are **incidental findings, not a security audit**. No authorization
  testing, input fuzzing, or cross-tenant access was attempted. Turning any of it into an actual audit requires
  the vendor's written permission first.
- The decision is revisitable, and `NEXT-STEPS.md` §4 records it as an open question for the user. If direct
  probing is authorized later, it should be restricted to GET/read endpoints, and must never touch
  `internal/create-credentials` or `internal/update-credentials`, which hold third-party integration secrets.

## Extension to MBO (2026-08-23)

The same rule was applied to the second vendor's back office at `mbo.kit.casino`. Two clarifications the ARC work
did not need:

- **Observing beats calling.** MBO is a micro-frontend app whose endpoints are assembled at runtime, so static
  string extraction recovers service prefixes but not routes. Instead the page's own `fetch`/`XMLHttpRequest`
  were instrumented and the application was navigated normally, recording only the calls the app itself made. No
  request was issued that the app would not have issued. This stays inside the rule while recovering far more
  than static analysis alone, and it is the technique to reuse.
- **The published asset manifest is fair game.** `GET /api/services/import-map` and the module bundles it names
  are the deployment's published static assets, and were fetched directly. Nothing else under `/api/` was.
- **Personal data was left masked.** MBO masks player PII by default; it was not unmasked, and only field
  structure was recorded.

The vendor is unidentified, so there is no one to seek written permission from for anything beyond this. The
incidental security observations in `src-analysis/MBO-BACKOFFICE-ANALYSIS.md` §8 are subject to the same
constraint as ARC's: they are findings from ordinary use, not an audit, and must not be turned into one without
permission.
