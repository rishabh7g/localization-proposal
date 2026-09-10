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

Render a PNG from the source:

```bash
npx -y @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -w 1900 -b white
```
