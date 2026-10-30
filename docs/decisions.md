# Design decisions and trade-offs

### D1 — Idempotency by `event_id`
Every event carries a unique `event_id`. Before writing anything, the scenario checks whether an Activity with that ID already exists and stops if it does. A repeated delivery never creates a second record.
**Trade-off:** this is query-then-create, not an atomic uniqueness constraint. Two identical events arriving at the same instant could both pass the check. At higher volume, a dedicated idempotency store would be the next step.

### D2 — Separate Leads (people) from Activities (events)
A person can sign up, start several checkouts and purchase. Keeping them as one Lead with many Activities gives a deduplicated contact list *and* the full journey.

### D3 — Match website Leads by email
Email is the identifier the website reliably has at signup and checkout.
**Trade-off:** email can be missing, mistyped, shared or changed. The roadmap adds normalisation (trim and lowercase), explicit missing-email handling, and stitching with Instagram identities.

### D4 — Explicit routers instead of a dynamic expression
The first version mapped `event_type` to an Activity Type with a single dynamic `switch()` expression. In testing it returned an empty value. I replaced it with explicit routers and fixed select values per event: more modules, but every path is visible, testable and easy to debug.

### D5 — Record when the event happened, not when it was processed
Activities use the timestamp sent by the website, falling back to processing time only if it is missing. Funnel timing (signup → checkout → purchase) then reflects user behaviour, not automation delays.

### D6 — The same detail on every path
Both Lead branches build the Activity detail the same way, so a purchase looks identical in the CRM whether the buyer was new or already known. Reports never have to care which branch created a record.

### D7 — A frontend purchase event is not a settled payment
The website sends `purchase_completed`, but only the payment provider can confirm that money actually moved. For authoritative revenue, purchases should be confirmed server-side by the payment provider's webhook, with signature verification and reconciliation of order ID, amount, currency and status.

### D8 — Notion as a lightweight CRM
Fast to build on and easy to inspect while the model is being validated.
**Trade-off:** not designed for high event volume or heavy reporting. A dedicated database or CRM is the path if usage grows.
