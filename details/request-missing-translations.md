# Inside "request missing translations"

One function, two callers: [path A](../path-user-landed-on-a-page.md), a
user hitting a missing label, and [path B](../backlog/path-reconcile-timer.md), the
hourly reconcile timer, which is in the backlog. Input: a set of keys and one culture. It never calls
AI and never waits.

Rules:

- Skip any key that already has a row for that culture, whatever its status.
  Pending, machine, and reviewed rows are all "already handled".
- Upsert pending rows first, publish second. A crash in between leaves a
  pending row that recovery picks up, never a message with no row.
- Chunk at a couple hundred keys per message. Message ID is a hash of the
  chunk's sorted keys plus the culture, so Service Bus duplicate detection
  drops a repeat inside the window.
- Recovery: an hourly timer republishes rows pending longer than an hour
  and refreshes their timestamp, so a dead-lettered batch does not leave
  keys stuck. Part of path A. Path B adds its query to this same timer.
- Runs only in staging, behind an explicit flag.

```mermaid
sequenceDiagram
    participant Caller
    participant RMT as request missing translations
    participant DB
    participant Queue as Service bus queue
    participant Sweep as Recovery timer (hourly)

    Caller->>RMT: (keys, culture)
    RMT->>DB: select existing rows for (keys, culture)
    DB-->>RMT: keys that already have a row
    RMT->>RMT: drop them. nothing left? return

    RMT->>DB: upsert pending rows with timestamp for remaining keys
    loop chunks of ~200 keys
        RMT->>Queue: publish (chunk keys, culture)<br/>messageId = hash(sorted keys, culture)
    end
    RMT-->>Caller: return immediately

    Note over DB,Sweep: recovery, hourly. part of path A
    Sweep->>DB: select rows pending longer than 1 hour
    DB-->>Sweep: stale pending rows, grouped by culture
    Sweep->>Queue: republish each group
    Sweep->>DB: refresh pending timestamp
```
