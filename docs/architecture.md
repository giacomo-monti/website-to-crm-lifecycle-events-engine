# Architecture, event contract and data model

## 1. Where this engine sits

```mermaid
flowchart LR
    IG[Instagram audience] --> MC[ManyChat funnel]
    MC -.->|planned trigger| M1[Make: Instagram scenario]
    M1 --> CRM[(Shared Notion CRM<br/>Leads + Activities)]
    WEB[Website built with Lovable] -->|JSON events| M2[Make: website scenario]
    M2 --> CRM
    CRM -.-> BI[Future funnel and revenue reporting]
```

## 2. Scenario flow

```mermaid
sequenceDiagram
    participant W as Website (Lovable)
    participant MK as Make
    participant NA as Notion: Activities
    participant NL as Notion: Leads
    W->>MK: POST JSON event
    MK->>NA: query by event_id
    alt event_id found
        MK-->>MK: stop (duplicate)
    else new event
        MK->>NL: query by email
        alt Lead exists
            MK->>NL: update Lead (last interaction)
        else no Lead
            MK->>NL: create Lead (Website, New Lead, Cold)
        end
        MK->>NA: create Activity for event_type, linked to the Lead
    end
```

**Routing.** After the Lead step, each branch (existing Lead / new Lead) has its own router with exact-match filters on `event_type`. That makes six Activity modules in total, three per branch, configured identically.

| `event_type` | Activity Type | Content detail stored |
|---|---|---|
| `signup` | Website Signup | — |
| `checkout_started` | Checkout Started | level, package, months, amount, currency, add-ons |
| `purchase_completed` | Purchase | level, package, months, amount, currency, add-ons, order ID |

Every Activity also stores: Activity ID = `event_id`, Channel = Website, Event ID, Timestamp = the event's own timestamp (falling back to processing time only if missing), Score Value, and the relation to the Lead.

## 3. Event contract

Synthetic examples. Fields shared by all events: `event_type`, `event_id`, `email`, `timestamp`.

**Signup**
```json
{
  "event_type": "signup",
  "event_id": "signup_demo_0001",
  "email": "person@example.com",
  "name": "Example Person",
  "language": "en",
  "country": "GB",
  "marketing_email_consent": false,
  "timestamp": "2026-01-15T12:00:00Z"
}
```

**Checkout started**
```json
{
  "event_type": "checkout_started",
  "event_id": "checkout_demo_0001",
  "email": "person@example.com",
  "level": "example_level",
  "package": "three_months",
  "months": 3,
  "amount": 149,
  "currency": "EUR",
  "selected_addons": ["addon_a", "addon_b"],
  "timestamp": "2026-01-15T12:05:00Z"
}
```

**Purchase completed**
```json
{
  "event_type": "purchase_completed",
  "event_id": "purchase_demo_0001",
  "email": "person@example.com",
  "level": "example_level",
  "package": "three_months",
  "months": 3,
  "amount": 149,
  "currency": "EUR",
  "selected_addons": ["addon_a"],
  "order_id": "ORDER-DEMO-0001",
  "timestamp": "2026-01-15T12:10:00Z"
}
```

`event_id` identifies a delivered event and is used for deduplication; `order_id` identifies the underlying order. They serve related but distinct purposes.

## 4. CRM data model (shared with the Instagram engine)

Simplified view: key fields only.

```mermaid
erDiagram
    LEADS ||--o{ ACTIVITIES : has
    LEADS {
        string lead_id
        string email
        string instagram_user_id
        string source
        string stage
        string lead_temperature
        date first_interaction
        date last_interaction
        string acquisition_keyword
        string acquisition_content_id
    }
    ACTIVITIES {
        string activity_id
        string event_id
        string channel
        string activity_type
        datetime timestamp
        string content_detail
        number score_value
    }
```

- **Leads** are people; **Activities** are lifecycle events and touchpoints.
- A website Lead starts as source Website, stage New Lead, temperature Cold. Its signup, checkout and purchase events then build its history as Activities.
- Original acquisition fields are never overwritten on update, so a person who first arrived from Instagram keeps that attribution.
