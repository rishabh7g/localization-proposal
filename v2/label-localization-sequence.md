# Label localization sequence, v2: review notification

Same flow as v1, plus a notification step. When the Func app fills a missing
label with an AI translation, it emails the localization team (a group address
or distribution list) with the batch: key, culture, source text, AI value, and
a review link. The AI value goes live immediately with status `machine`. A
reviewer later corrects or approves it through the review UI, which sets
status `reviewed`.

What changed from v1:

- The human path is no longer a request from the Func app. It is a notification
  plus a self-service review. The team reviews when they have time; nothing
  waits on them.
- One email per batch, not per label. If batches are frequent, switch to a
  daily digest built from rows with status `machine`.
- Email failure is logged and swallowed. It must never fail the queue message,
  or at-least-once redelivery would re-run the AI translation and re-charge you.
- Still a single queue and a single consumer. The email is sent by the Func app
  after the upsert, so it needs nothing new on the bus.
- The AI upsert is guarded: it only writes rows whose status is `pending` or
  `machine`. A `reviewed` row is never overwritten by a later AI run, even if a
  duplicate batch slips past duplicate detection.
- Pending rows carry a timestamp. A timer-triggered sweep republishes any row
  pending for more than an hour, so a batch that dead-letters does not leave
  its keys stuck behind the "already pending" check forever.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant API
    participant DB
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant Email as Email service
    actor Team as Localization team
    participant Review as Review UI

    User->>Browser: change culture (es)
    Browser->>API: GET labels?culture=es (whole bundle, by key)
    API->>DB: select labels where culture = es
    DB-->>API: partial result (some keys missing)

    opt keys missing
        API->>DB: upsert pending rows with timestamp (skip if already pending)
        API->>Queue: publish one batched message (keys, culture)<br/>messageId = hash(keys, culture) for duplicate detection
    end

    API-->>Browser: bundle: translated values + default text for misses
    Browser-->>User: page shown, missing labels in default text

    Queue->>Func: trigger (at-least-once delivery)
    Func->>AI: translate batch
    AI-->>Func: values
    Func->>DB: upsert values, status = machine<br/>only where status in (pending, machine), never reviewed

    Note over Queue,Func: on repeated failure the message goes to the dead-letter queue

    Note over Func,Email: one email per batch, failure is logged and never retried
    Func->>Email: send: keys, culture, source text, AI values, review link
    Email-->>Team: "N labels auto-translated to es, please review"

    Note over Team,Review: later, at the team's pace. Nothing waits on this
    Team->>Review: open review link
    Review->>DB: select rows where status = machine
    DB-->>Review: AI values
    Team->>Review: approve or correct each value
    Review->>DB: upsert values, status = reviewed

    Note over DB,Queue: timer sweep, hourly
    Func->>DB: select rows pending longer than 1 hour
    DB-->>Func: stale pending keys
    Func->>Queue: republish batch, refresh pending timestamp

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (machine or reviewed values)
    API-->>Browser: bundle with new values
    Browser-->>User: button shown in Spanish
```
