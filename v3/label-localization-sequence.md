# Label localization sequence, v3: pull request review, DB source of truth

Scope: product UI strings only. Tenant-authored content is out of scope.

The DB is the source of truth for culture, key, and value. The moment a value
lands in the DB it is live in that environment. The pull request replaces the
Review UI and the email: it is the notification, the review tool, and the
audit log.

This process runs in staging only. Staging is where misses are detected, AI
values are written, and PRs are opened. Other environments receive reviewed
values through the seed script on deploy.

Flow:

1. A miss in the bundle upserts pending rows and publishes one batched message
   per culture.
2. The Func app asks AI to translate the batch and upserts the values into the
   DB with status `machine` and no PR reference. Staging shows them on the next
   bundle fetch.
3. A separate PR step selects rows with status `machine` and no PR reference,
   commits a seed file for them (culture, key, source text, AI value), opens a
   pull request with the team requested as reviewers, and writes the PR
   reference back on those rows. It runs after each batch and on a timer, so a
   failed PR is picked up on the next run without re-running AI.
4. A reviewer edits the values in the PR, approves, and merges. The next
   product deploy runs the seed script, which creates any label that is
   missing and then updates it, setting status `reviewed`. That applies in
   every environment the release reaches.
5. The Func app never merges and never deploys.

Rules that keep it safe:

- The AI upsert only writes rows whose status is `pending` or `machine`. A
  `reviewed` row is never overwritten by a later AI run.
- The seed script always wins. It creates missing labels, updates existing
  ones, and sets status `reviewed`, which the AI path will not touch again.
- The PR step is idempotent. It is keyed on rows with no PR reference, so a
  retry or a timer run cannot open a second PR for the same rows.
- Pending rows carry a timestamp. An hourly sweep republishes rows pending
  longer than an hour, so a dead-lettered batch does not leave keys stuck.
- Queue consumer handles one message at a time. A rejected push is retried
  once after a fresh read of main.

Accepted for now: one PR per batch, and reviewers editing the seed file by
hand.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant API
    participant DB
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant Repo as GitHub repo
    actor Team as Localization team
    participant Deploy

    Note over User,Deploy: runs in staging only. Other environments get reviewed values from the seed script on deploy

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

    Queue->>Func: trigger (one message at a time, at-least-once)
    Func->>AI: translate batch (key, source text)
    AI-->>Func: values
    Func->>DB: upsert values, status = machine, pr_ref = null<br/>only where status in (pending, machine), never reviewed
    Note over DB,Func: values are live in staging from this point<br/>hourly sweep republishes rows pending over 1 hour

    Note over DB,Repo: PR step: after each batch and on a timer. Idempotent, never re-runs AI
    Func->>DB: select rows where status = machine and pr_ref is null
    DB-->>Func: rows needing a PR
    opt rows found
        Func->>Repo: commit seed file (culture, key, source, AI value) on branch l10n/es-<batch>
        Func->>Repo: open PR, request review from team
        Note over Func,Repo: rejected push: re-read main, retry once
        Func->>DB: set pr_ref on those rows
        Repo-->>Team: review requested (GitHub notification email)
    end

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (machine values)
    API-->>Browser: bundle with new values
    Browser-->>User: button shown in Spanish (AI value)

    Note over Team,Deploy: later, at the team's pace. Nothing waits on this
    Team->>Repo: edit values in the PR, approve
    Team->>Repo: merge
    Repo->>Deploy: next product deploy
    Deploy->>DB: run seed script: create label if missing, then update<br/>status = reviewed, in every environment the release reaches

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (reviewed values)
    API-->>Browser: bundle with corrected values
    Browser-->>User: button shown in Spanish (reviewed value)
```
