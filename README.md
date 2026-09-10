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
  truth. The Func app upserts AI values into the DB, live at once, then opens a
  PR with a translation migration file. A human edits, approves, merges, and
  deploys. The deploy applies the migration with status `reviewed`. No Review
  UI, no email service. Scope: product UI strings only.

Render a PNG from the source:

```bash
npx -y @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -w 1900 -b white
```
