# Localization proposal

On-demand, asynchronous translation of product UI strings. AI fills a missing
label within minutes and it goes live in staging at once. The localization
team reviews it later through a pull request, and the next product deploy
seeds the reviewed value into every environment. The DB is the source of
truth. No Review UI, no email service. Scope: product UI strings only.

Two paths, each drawn end to end. Pick the one that matches how the
translation was requested.

- [Path A: a user landed on a page](path-user-landed-on-a-page.md). The
  priority path. Page renders with default text, miss requested in the
  background, AI value on the next page load, reviewed value after merge and
  deploy.
- [Path B: reconcile timer](path-reconcile-timer.md). Backlog, not in the
  current build. An hourly query finds every missing (key, culture) pair no
  user has hit yet. Identical to path A from the queue onward.

The three stages both paths share are opened up in `details/`:

- [request missing translations](details/request-missing-translations.md)
- [translate and open PR](details/translate-and-open-pr.md)
- [review and delivery](details/review-and-delivery.md)

Technical feasibility and build order: [feasibility.md](feasibility.md).

Render a PNG from any file's mermaid block:

```bash
npx -y @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -w 1600 -b white
```
