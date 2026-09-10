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
   per culture. The same "request missing translations" function also runs
   when a developer inserts a new English label (for every enabled culture)
   and when a culture is enabled (for every existing key), so new labels are
   translated before any user opens the page.
2. The Func app asks AI to translate the batch and upserts the values into the
   DB with status `machine` and no PR reference. Staging shows them on the next
   bundle fetch.
3. A separate PR step selects rows with status `machine` and no PR reference,
   grouped by culture. The branch name is a hash of the culture plus the
   sorted keys, so the same rows always map to the same branch. If that
   branch already exists, the step writes its PR reference on the rows and
   stops. Otherwise it commits a seed file (culture, key, source text, AI
   value), opens a pull request, writes the PR reference, then requests the
   team as reviewers. It runs after each batch and on a timer, so a failed PR
   is picked up on the next run without re-running AI.
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
- The PR step is idempotent twice over. The DB side is the PR reference
  column. The GitHub side is the deterministic branch name: a re-run after a
  crash finds the existing branch instead of opening a second PR.
- Pending rows carry a timestamp. An hourly sweep republishes rows pending
  longer than an hour, so a dead-lettered batch does not leave keys stuck.
- Queue consumer handles one message at a time. A rejected push is retried
  once after a fresh read of main.
- The request-missing-translations function runs only in staging, behind an
  explicit flag. Large requests, such as enabling a culture with thousands of
  keys, are chunked at a couple hundred keys per message.

Accepted for now: one PR per batch, and reviewers editing the seed file by
hand.

```mermaid
sequenceDiagram
    actor Dev as Developer
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

    Note over Dev,Deploy: runs in staging only. Other environments get reviewed values from the seed script on deploy

    Note over Dev,Queue: request path A: a developer adds a new label
    Dev->>DB: insert new key with English value
    DB->>API: new key inserted (hook)
    API->>DB: for every enabled culture: upsert pending row (skip if exists)
    API->>Queue: publish one batched message per culture<br/>messageId = hash(keys, culture)
    Note over API,Queue: the same hook runs when a culture is enabled, for every existing key

    Note over Dev,Queue: request path B: a user hits a missing label
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
        Func->>Func: branch = l10n/es-hash(culture, sorted keys)
        Func->>Repo: does branch exist?
        alt branch exists (earlier run crashed)
            Repo-->>Func: yes, open PR found
            Func->>DB: set pr_ref on those rows
        else new batch
            Func->>Repo: create branch, commit seed file (culture, key, source, AI value)
            Func->>Repo: open PR
            Func->>DB: set pr_ref on those rows
            Func->>Repo: request review from team
            Repo-->>Team: review requested (GitHub notification email)
        end
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
