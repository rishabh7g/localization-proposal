# Label localization sequence, v3: pull request review, DB source of truth

Scope: product UI strings only. Tenant-authored content is out of scope.

The DB stays the source of truth for culture, key, and value. The moment a
value lands in the DB it is live. The pull request replaces the Review UI and
the email: it is the notification, the review tool, and the audit log.

Flow:

1. A miss in the bundle upserts pending rows and publishes one batched message
   per culture.
2. The Func app asks AI to translate the batch and upserts the values into the
   DB with status `machine`. Users see them on the next bundle fetch.
3. The Func app then opens a pull request containing a translation migration
   file for the batch: culture, key, source text, AI value. Reviewers are
   requested on it. GitHub's review-request email is the notification.
4. A reviewer edits the values in the PR, approves, merges, and deploys. The
   deploy applies the migration as an upsert with status `reviewed`.
5. The Func app never merges and never deploys.

Rules that keep it safe:

- The AI upsert only writes rows whose status is `pending` or `machine`. A
  `reviewed` row is never overwritten by a later AI run.
- The migration upsert always wins. It sets status `reviewed`, which the AI
  path will not touch again.
- Pending rows carry a timestamp. An hourly sweep republishes rows pending
  longer than an hour, so a dead-lettered batch does not leave keys stuck.
- The PR is opened after the DB upsert. If the PR step fails, the values are
  already live and the failure is logged, never retried, so at-least-once
  redelivery cannot re-run the AI call. The sweep does not cover a missing PR,
  so log it loudly.
- Queue consumer handles one message at a time. A rejected push is retried
  once after a fresh read of main.

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
    Func->>DB: upsert values, status = machine<br/>only where status in (pending, machine), never reviewed
    Note over DB,Func: values are live from this point<br/>hourly sweep republishes rows pending over 1 hour

    Func->>Repo: commit migration file (culture, key, source, AI value) on branch l10n/es-<batch>
    Func->>Repo: open PR, request review from team
    Note over Func,Repo: rejected push: re-read main, retry once<br/>PR failure is logged, never retried, values already live
    Repo-->>Team: review requested (GitHub notification email)

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (machine values)
    API-->>Browser: bundle with new values
    Browser-->>User: button shown in Spanish (AI value)

    Note over Team,Deploy: later, at the team's pace. Nothing waits on this
    Team->>Repo: edit values in the PR, approve
    Team->>Repo: merge
    Team->>Deploy: deploy
    Deploy->>DB: apply migration: upsert values, status = reviewed

    User->>Browser: next page load or bundle refresh
    Browser->>API: GET labels?culture=es
    API->>DB: select labels where culture = es
    DB-->>API: full result (reviewed values)
    API-->>Browser: bundle with corrected values
    Browser-->>User: button shown in Spanish (reviewed value)
```
