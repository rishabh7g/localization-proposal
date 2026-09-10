# Localization proposal

Sequence diagrams for on-demand, asynchronous translation of UI labels.
Each version is a folder with the mermaid source and a rendered PNG.

- [v1](v1/label-localization-sequence.md): lazy translation. A missing label
  falls back to default text, one batched Service Bus message triggers a Func
  app that fills the gap with AI, and slow human translation writes back via a
  separate workflow.
- [v2](v2/label-localization-sequence.md): v1 plus a review notification. The
  Func app emails the localization team with each auto-translated batch and a
  review link. AI values go live at once with status `machine`; reviewers
  correct or approve them later, setting status `reviewed`.
- [v3](v3/label-localization-sequence.md): pull request review, DB source of
  truth. Runs in staging only. The Func app upserts AI values into the DB, live
  at once. An idempotent PR step opens a PR with a seed file for rows that have
  no PR yet. A human edits, approves, and merges. The next product deploy runs
  the seed script, creating missing labels and updating them with status
  `reviewed`, in every environment. Feasibility: [v3/feasibility.md](v3/feasibility.md). No Review UI, no email service. Scope:
  product UI strings only.

Render a PNG from the source:

```bash
npx -y @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -w 1900 -b white
```
