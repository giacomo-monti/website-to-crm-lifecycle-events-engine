# Testing, limitations and roadmap

Every claim in this repository is labelled as **implemented**, **tested and observed**, or **planned**. Functional tests prove the logic; they say nothing about traffic, conversion or revenue.

## Tested and observed (test events)

- A `signup` event created a new Lead and a linked Website Signup Activity.
- A repeated email **resolved to the existing Lead** instead of creating a duplicate. The router filters were initially reversed during testing; I caught it and corrected it, then re-tested.
- `checkout_started` payloads arrived with package, months, amount, currency and add-ons.
- Explicit routing produced the correct Activity Type for each event, fixing the empty-value issue of the earlier dynamic expression.
- A `purchase_completed` test payload carried the order ID and commercial fields through to the CRM.

## Not claimed

Website traffic, signup or checkout volumes, conversion rates, revenue, ROI. Purchase events are test payloads, not settled payments.

## Known limitations

- Deduplication is query-then-create, not atomic.
- Lead matching relies on email; missing or inconsistent emails need explicit rules.
- Lead profile fields from the signup (name, phone, language, country, consents) are received but not yet written to the CRM: this part is still in development.
- Purchases are not yet confirmed by a payment provider.
- No webhook authentication, failure alerting or reconciliation yet.

## Roadmap

1. **Payment-verified purchases:** payment-provider webhook as the source of truth, signature verification, and reconciliation of order ID, amount, currency and status.
2. **Webhook hardening:** shared secret or signature on incoming events, plus schema validation.
3. **Identity:** normalise emails, define missing-email behaviour, and stitch website Leads with Instagram Leads.
4. **Complete Lead profiles:** write name, language, country and consent flags. A signup is never treated as marketing consent.
5. **Reliability:** error handlers, failure alerts, an event receipt log and periodic reconciliation.
6. **Funnel reporting:** signup → checkout → purchase conversion, drop-off by package and add-on demand.
7. **Scale path:** atomic idempotency and a dedicated store if volume or reporting needs outgrow Notion.
