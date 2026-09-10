# Label localization sequence

On-demand, asynchronous translation of UI labels. The browser fetches the whole
label bundle for a culture through the API and caches it. Misses fall back to
the default text, get marked pending once, and are published as one batched
Service Bus message. A Func app fills fast translations with AI; slow human or
third-party translations write back through a separate workflow. The browser
picks up new values on its next bundle fetch. No polling.

Design rules baked in:

- Labels are keyed by identifier, never by English text.
- One consumer, so a queue, not a topic. The Func app is triggered by the queue directly.
- A pending row plus Service Bus duplicate detection (message ID = hash of key + culture) stops repeat requests.
- Stores are upserts, since delivery is at least once. Dead-letter queue with a retry limit for bad messages.
- Every translation carries a status: pending, machine, reviewed.
- The browser never talks to the DB.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant API
    participant DB
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant Human as Human / 3rd party

    User->>Browser: open app
    Browser-->>User: paint app

    User->>Browser: change culture (es)
    Browser->>API: GET labels?culture=es (whole bundle, by key)
    API->>DB: select labels where culture = es
    DB-->>API: partial result (some keys missing)

    opt keys missing
        API->>DB: upsert pending rows for missing keys (skip if already pending)
        API->>Queue: publish one batched message (keys, culture)<br/>messageId = hash(keys, culture) for duplicate detection
    end

    API-->>Browser: bundle: translated values + default text for misses
    Browser->>Browser: cache bundle per culture
    Browser-->>User: page shown, missing labels in default text

    Queue->>Func: trigger (at-least-once delivery)
    Func->>AI: translate batch
    AI-->>Func: values
    Func->>DB: upsert values, status = machine

    Note over Func,Human: slow path, never blocks the Func app
    Func->>Human: request review / translation
    Human-->>DB: upsert values, status = reviewed (separate workflow, hours or days later)

    Note over Queue,Func: on repeated failure the message goes to the dead-letter queue

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result
    API-->>Browser: bundle with new values
    Browser-->>User: button shown in Spanish
```
