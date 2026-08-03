# Read-only conduct against the vendor sandbox

**Status:** accepted

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
