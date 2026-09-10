# 1. Requesting translations

Three events can ask for a translation. All three end in the same function,
"request missing translations", drawn here as a black box and explained in
[2-request-missing-translations.md](2-request-missing-translations.md).

- A developer inserts a new label with its English value. The hook asks for
  every enabled culture, so the label is translated before anyone opens the
  page.
- A culture is enabled. The hook asks for every existing key.
- A user opens a page and a label is missing for their culture. The page
  renders with default text, and the miss is requested in the background.
  This is the safety net for anything the first two paths missed.

All of this runs in staging only.

```mermaid
sequenceDiagram
    actor Dev as Developer
    actor User
    participant Browser
    participant API
    participant DB
    participant RMT as request missing translations<br/>(black box, see 2)

    Note over Dev,RMT: path A: a developer adds a label
    Dev->>DB: insert key with English value
    DB->>API: new key inserted (hook)
    API->>RMT: (new keys, every enabled culture)

    Note over Dev,RMT: path B: a culture is enabled
    Dev->>API: enable culture (fr)
    API->>RMT: (every existing key, fr)

    Note over Dev,RMT: path C: a user hits a missing label
    User->>Browser: change culture (es)
    Browser->>API: GET labels?culture=es (whole bundle, by key)
    API->>DB: select labels where culture = es
    DB-->>API: partial result (some keys missing)
    API->>RMT: (missing keys, es)
    API-->>Browser: bundle: translated values + default text for misses
    Browser-->>User: page shown, missing labels in default text
```
