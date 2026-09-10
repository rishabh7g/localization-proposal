# v3 technical feasibility

**Verdict: feasible with off-the-shelf parts.** Every component is a standard
Azure or GitHub primitive with a documented SDK. There is no novel technology
and no component that needs a spike before committing. The work is mostly
wiring, and the two places that need real design attention are the seed
script's contract and the AI call's output validation.

Numeric platform limits below are from memory. Confirm each against current
docs before writing an acceptance criterion on it.

## Components

| Component | What it needs | New or existing | Risk |
|---|---|---|---|
| API bundle endpoint | Bundle-per-culture read, miss detection, pending upsert, one publish | Existing API, small change | Low |
| DB schema | `status`, `pending_at`, `pr_ref`, `source_text` columns on the translation table | Migration | Low |
| Service Bus queue | Standard tier, duplicate detection on, default dead-letter | New resource | Low |
| Func app, translate | Service Bus trigger, one message at a time, AI call, guarded upsert | New | Medium, see AI section |
| Func app, PR step | Timer plus post-batch call, GitHub App auth, branch, commit, PR, reviewers | New | Low |
| Func app, sweep | Timer, republish stale pending rows | New | Low |
| Seed script | Runs in the deploy, create-then-update, sets `reviewed` | New, runs in existing deploy | Medium, see seed section |
| AI translation | Claude API via the official SDK, structured JSON output | New | Medium |

## Detail

### API and DB

The API already reads labels. Add a bundle endpoint that returns all keys for
a culture in one response.

Write the miss handling as one function, request missing translations, that
takes a set of keys and a culture. It upserts pending rows for keys that have
no row, skipping keys already pending, machine, or reviewed, then publishes
one message. The publish must come after the upsert so a crash between them
leaves a pending row that the sweep recovers, rather than a message with no
row. Chunk the publish at a couple hundred keys per message.

The function has two callers:

- The bundle endpoint, on a miss. This is the fast lane: it fires the moment
  a user hits a missing label.
- The hourly reconcile timer in the Func app. One query: every key with an
  English row, crossed with every enabled culture, minus pairs that already
  have a row. It calls the function once per culture with what is left. This
  covers a developer adding a label, a culture being enabled, and anything
  that slipped through, with no hook into the insert path and no backfill
  command. Latency is up to an hour, which is fine because the miss path
  covers anyone who opens the page sooner.

The function runs only in staging, behind an explicit flag, since that is
the only environment with the queue and the Func app.

Schema: `status` enum of pending, machine, reviewed. `pending_at` timestamp.
`pr_ref` nullable string. `source_text` so the seed file and the AI prompt do
not need a second lookup. One migration, backward compatible, existing rows
default to reviewed.

### Service Bus

Standard tier is enough. Turn on duplicate detection at queue creation, it
cannot be enabled later. The default window is 10 minutes, configurable to 7
days. Set it long, a day is fine, since a repeat batch inside the window is
dropped silently and that is the intent. Message body is keys plus culture,
well under the 256 KB standard-tier limit even for a thousand keys. Dead-letter
is on by default after the max delivery count, leave both at defaults.

### Func app

One Function App, three functions.

Translate: Service Bus queue trigger. Set `maxConcurrentCalls` to 1 in
`host.json` so batches are serialized. Consumption plan timeout is 5 minutes
by default and 10 at most, so chunk the AI call at around 50 keys per request
and loop. A batch of a few hundred keys fits comfortably. The upsert is one
statement with a `WHERE status IN ('pending', 'machine')` clause.

PR step: timer trigger every 15 minutes, and also called at the end of
translate. Selects rows with status machine and null `pr_ref`, groups by
culture, and for each group:

1. Compute the branch name as `l10n/<culture>-<hash>` where the hash covers
   the culture and the sorted keys. The same rows always give the same name.
2. Ask GitHub whether the branch exists. If it does, look up its open PR,
   write `pr_ref` on the rows, and stop.
3. Otherwise create the branch from main, put the seed file, open the PR,
   write `pr_ref`, then request reviewers.

Idempotency is two-sided. The `pr_ref` column stops a re-run from picking the
rows up again. The deterministic branch name stops a re-run that does pick
them up, after a crash between opening the PR and writing `pr_ref`, from
opening a second PR. Writing `pr_ref` before requesting reviewers leaves only
one API call and one DB write in the crash window, and step 2 recovers that.

The hash must cover exactly the rows in the group, so the translate function
and the sweep must group the same way: by culture, keys sorted. New keys for
the same culture later are a different set, a different hash, and correctly a
different PR.

GitHub auth: a GitHub App installed on the repo with contents write and pull
requests write. Installation tokens are short-lived and scoped, unlike a
personal token. The four REST calls needed, create ref, put contents, create
pull, request reviewers, are all in the Octokit SDKs. Rate limits are
thousands per hour per installation, irrelevant at this volume.

Reconcile: timer trigger hourly, two queries. First, the missing-pairs query
above, calling request missing translations per culture. Second, rows pending
longer than an hour, grouped by culture, republished with `pending_at`
refreshed. Twenty lines.

Secrets: AI key and GitHub App private key in Key Vault, referenced from app
settings. Managed identity for Service Bus and the DB so there are no
connection strings.

### AI translation

Use the official Anthropic SDK against `claude-opus-5`, with structured output
so the response is a JSON object keyed by label key. Send the batch as one
user message containing key, source text, and target culture. Validate the
response before the upsert: every returned key must be in the request, every
requested key must be present, and no value may be empty. Drop and log any
row that fails, do not fail the batch. A short system prompt fixing the
product domain and asking for UI-length translations is enough. Keep the
prompt stable and put the batch after it so prompt caching applies.

Latency for a 50-key chunk is seconds. Cost per label is a fraction of a
cent. If a run ever needs to translate a whole culture at once, the Batch API
runs asynchronously at half price and fits the "nothing waits on this"
design, but it is not needed for the miss-driven path.

Put the call behind a small interface with one method, translate batch, so
the provider can change without touching the function.

### Seed script

This is the piece with the most implicit contract. It runs in the product
deploy against every environment the release reaches, so it must be:

- Idempotent. Re-running applies the same result.
- Create-then-update. Insert the label row when the key is absent for that
  culture, otherwise update value and status.
- Order-independent across files. Two seed files for the same key must
  converge on the later one. Simplest rule: apply files in commit order,
  last write wins.
- Fast. A few thousand rows per deploy in one transaction.
- Silent on empty input, so a deploy with no new seed files is a no-op.

Seed file format: one file per batch under a `translations/` folder, columns
culture, key, source text, value. CSV or JSON, whichever the team is more
comfortable editing in the GitHub web editor. Source text is for reviewer
context and is not applied.

The seed script needs to run in the deploy workflow, after the app migration
step and before traffic switches. That is one added step in the existing
deploy job.

## Risks and open questions

- **Reconcile query cost.** The missing-pairs query is a cross join of keys
  and cultures with an anti-join. A few thousand keys times a handful of
  cultures is trivial hourly. If the label table grows past a hundred
  thousand rows, index (culture, key) and re-check the plan.
- **Source text changes.** If the English label changes after translation,
  the reviewed value is stale and nothing flags it. Out of scope for now.
  Storing `source_text` on the row makes a later staleness check possible.
- **Seed file conflicts.** Two open PRs never touch the same file, so merges
  do not conflict. Two PRs can carry the same key only if a reviewer edits
  values across batches by hand, and last-write-wins covers that.
- **Reviewer GitHub access.** Reviewers need write access to approve and
  merge. Confirm the localization team has it.
- **AI output quality.** Untested on this product's strings. A twenty-label
  sample run before building anything else answers whether prompt work is
  needed. Cheapest de-risking step available.

## Proposed issue breakdown

In build order. Each is one branch, one merge, one verification on staging.

0. AI sample run: twenty real labels through the translate call, output read
   by a human. No code merged. Decides whether prompt work is needed.
1. DB migration: `status`, `pending_at`, `pr_ref`, `source_text`.
2. Service Bus queue with duplicate detection, plus managed identity.
3. API bundle endpoint with miss detection, pending upsert, and publish.
4. Func app skeleton with managed identity, Key Vault, `maxConcurrentCalls` 1.
5. Translate function: AI client behind an interface, validation, guarded upsert.
6. Reconcile timer: missing-pairs query plus stale-pending republish.
7. GitHub App registration and PR step function.
8. Seed script and deploy step.
9. Staging-only flag on request missing translations, and the enabled
   cultures list the reconcile query reads.
10. End-to-end verification on staging: force a miss, watch the PR appear,
    merge, deploy, confirm the reviewed value.

Issue 5 can start as soon as issue 1 lands. Issues 2 and 7 have no code
dependencies and can go first if someone else is available.
