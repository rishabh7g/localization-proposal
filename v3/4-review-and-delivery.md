# 4. Review and delivery

Picks up where [3](3-translate-and-open-pr.md) opened a PR. Nothing here
waits on anyone, and the Func app is not involved.

- Staging already shows the AI value. The PR is the review surface: the
  reviewer edits values in the seed file, approves, and merges.
- The next product deploy runs the seed script in every environment the
  release reaches. The script creates any label that is missing, then updates
  it, and sets status reviewed. It is idempotent, last write wins across
  files, and a no-op when there are no new files.
- Reviewed rows are never touched by the AI path again.

```mermaid
sequenceDiagram
    actor Team as Localization team
    participant Repo as GitHub repo
    participant Deploy
    participant DB
    participant API
    participant Browser
    actor User

    Note over Team,User: staging is already showing the AI value. later, at the team's pace
    Team->>Repo: edit values in the PR seed file, approve
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
