# Path A: a user landed on a page

The priority path. A user opens a page in a culture that has no value for
some label. The page renders at once with the default text, the miss is
requested in the background, AI fills it within minutes, and the localization
team reviews it later through a pull request. Nothing on this path waits for
anyone.

Runs in staging only. Other environments receive reviewed values through the
seed script on deploy. The DB is the source of truth: the moment a value
lands there, it is live in that environment.

The three shared stages are drawn as boxes here and opened up in `details/`:

- [request missing translations](details/request-missing-translations.md):
  dedup, pending rows, chunked publish, stale-pending recovery.
- [translate and open PR](details/translate-and-open-pr.md): AI call with
  validation, guarded upsert, idempotent PR step.
- [review and delivery](details/review-and-delivery.md): merge, seed script
  on deploy, reviewed value in every environment.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant API
    participant DB
    participant RMT as request missing translations<br/>(details)
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant Repo as GitHub repo
    actor Team as Localization team
    participant Deploy

    Note over User,Deploy: 1. user hits a missing label. page renders now, miss requested in the background
    User->>Browser: change culture (es)
    Browser->>API: GET labels?culture=es (whole bundle, by key)
    API->>DB: select labels where culture = es
    DB-->>API: partial result (some keys missing)
    opt keys missing
        API->>RMT: (missing keys, es). returns immediately
        RMT->>DB: pending rows for keys with no row
        RMT->>Queue: one batched message, messageId = hash(sorted keys, es)
    end
    API-->>Browser: bundle: translated values + default text for misses
    Browser-->>User: page shown, missing labels in default text

    Note over Queue,Repo: 2. translate and open PR (details). minutes later
    Queue->>Func: trigger (one message at a time)
    Func->>AI: translate batch (key, source text)
    AI-->>Func: values, validated
    Func->>DB: upsert values, status = machine. never overwrites reviewed
    Func->>Repo: PR with seed file, idempotent by branch name and pr_ref
    Repo-->>Team: review requested

    Note over User,DB: 3. user sees the AI value on the next page load
    User->>Browser: next page load
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (machine values)
    API-->>Browser: bundle
    Browser-->>User: button shown in Spanish (AI value)

    Note over Team,Deploy: 4. review and delivery (details). later, at the team's pace
    Team->>Repo: edit values in the PR, approve, merge
    Repo->>Deploy: next product deploy
    Deploy->>DB: seed script: create if missing, then update. status = reviewed, every environment

    Note over User,DB: 5. user sees the reviewed value on the next page load
    User->>Browser: next page load
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (reviewed values)
    API-->>Browser: bundle
    Browser-->>User: button shown in Spanish (reviewed value)
```
