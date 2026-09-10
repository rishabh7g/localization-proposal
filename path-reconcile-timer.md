# Path B: reconcile timer

**Backlog. Not in the current build.** The catch-up path. An hourly timer in the Func app finds every (key,
culture) pair that should exist and does not: every key with an English
row, crossed with every enabled culture, minus pairs that already have a
row. Whatever is left is requested, once per culture. This covers a
developer adding a label, a culture being enabled, and anything else no user
has hit yet. No hook into the insert path.

Latency is up to an hour, which is fine: [path A](path-user-landed-on-a-page.md)
fires immediately for anyone who opens the page sooner. From the queue
onward the two paths are identical.

Runs in staging only. Other environments receive reviewed values through the
seed script on deploy. The DB is the source of truth.

The three shared stages are drawn as boxes here and opened up in `details/`:

- [request missing translations](details/request-missing-translations.md):
  dedup, pending rows, chunked publish, stale-pending recovery. Path A's
  hourly recovery timer is the timer this query is added to.
- [translate and open PR](details/translate-and-open-pr.md): AI call with
  validation, guarded upsert, idempotent PR step.
- [review and delivery](details/review-and-delivery.md): merge, seed script
  on deploy, reviewed value in every environment.

```mermaid
sequenceDiagram
    participant Timer as Reconcile timer (hourly)
    participant DB
    participant RMT as request missing translations<br/>(details)
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant Repo as GitHub repo
    actor Team as Localization team
    participant Deploy
    actor User

    Note over Timer,Deploy: 1. reconcile. covers new labels and newly enabled cultures
    Timer->>DB: keys with an English row x enabled cultures, minus pairs that have a row
    DB-->>Timer: missing (key, culture) pairs, grouped by culture
    loop each culture with missing keys
        Timer->>RMT: (missing keys, culture)
        RMT->>DB: pending rows for keys with no row
        RMT->>Queue: batched messages of ~200 keys, messageId = hash(sorted keys, culture)
    end
    Timer->>DB: existing query from path A: republish rows pending longer than 1 hour (recovery)

    Note over Queue,Repo: 2. translate and open PR (details). minutes later
    Queue->>Func: trigger (one message at a time)
    Func->>AI: translate batch (key, source text)
    AI-->>Func: values, validated
    Func->>DB: upsert values, status = machine. never overwrites reviewed
    Func->>Repo: PR with seed file, idempotent by branch name and pr_ref
    Repo-->>Team: review requested

    Note over DB,User: 3. staging shows the AI value from here. no user had to hit the miss
    User->>DB: next page load: bundle with machine values

    Note over Team,Deploy: 4. review and delivery (details). later, at the team's pace
    Team->>Repo: edit values in the PR, approve, merge
    Repo->>Deploy: next product deploy
    Deploy->>DB: seed script: create if missing, then update. status = reviewed, every environment

    Note over DB,User: 5. reviewed value in every environment
    User->>DB: next page load: bundle with reviewed values
```
