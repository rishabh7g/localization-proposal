# Translating and opening the PR

Picks up where [request missing translations](request-missing-translations.md)
left a message on the queue. Identical for both paths. Two steps in one Func
app: translate, then open a PR.

Translate:

- One message at a time. Delivery is at least once.
- AI call chunked at about 50 keys per request. The reply is validated:
  every returned key was requested, every requested key is present, no value
  is empty. Bad rows are dropped and logged, the rest are saved.
- The upsert writes only rows whose status is pending or machine. A reviewed
  row is never overwritten. Values are live in staging from this write.

PR step, after each batch and on a 15 minute timer:

- Selects rows with status machine and no PR reference, grouped by culture.
- Branch name is a hash of the culture plus the sorted keys, so the same rows
  always map to the same branch. If the branch exists, an earlier run
  crashed: write the PR reference and stop. Otherwise create the branch,
  commit the seed file, open the PR, write the PR reference, then request
  reviewers.
- Idempotent on both sides: the PR reference column on the DB side, the
  deterministic branch name on the GitHub side. It never re-runs AI.

```mermaid
sequenceDiagram
    participant Queue as Service bus queue
    participant Func as Func app
    participant AI
    participant DB
    participant Repo as GitHub repo
    actor Team as Localization team

    Queue->>Func: trigger (one message at a time, at-least-once)
    loop chunks of ~50 keys
        Func->>AI: translate (key, source text) for culture
        AI-->>Func: values as JSON keyed by label key
        Func->>Func: validate: keys match, no empty values. drop and log bad rows
    end
    Func->>DB: upsert values, status = machine, pr_ref = null<br/>only where status in (pending, machine), never reviewed
    Note over DB: values are live in staging from this point

    Note over Func,Repo: PR step: after each batch and every 15 minutes
    Func->>DB: select rows where status = machine and pr_ref is null, grouped by culture
    DB-->>Func: rows needing a PR
    opt rows found
        Func->>Func: branch = l10n/<culture>-hash(culture, sorted keys)
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
```
