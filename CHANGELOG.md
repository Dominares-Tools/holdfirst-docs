# Changelog

## 1.0.0

First release.

- `Budget`: a ceiling that follows a prepaid credit (`grant.json`: credit left at a reading, expiry, buffer), a
  fixed `max_usd` when there is no credit, and `HOLDFIRST_MAX_USD` to lower either.
- Worst-case holds before a call (`hold`, `release`, `spend`, `check`), kept across processes for batches.
- Per-job caps by name prefix, per UTC month and in total (`caps.json`).
- An append-only ledger summed incrementally, with no prompt or reply text in it.
- A price table you write (`prices.json`): long-context cards, cache rates, batch factors; unlisted models are
  priced at the dearest rate of every column.
- Usage readers for chat-completions, responses, messages and generate-content style SDK objects.
- `halt` / `resume`, and the `holdfirst` command: `status`, `grant`, `check`, `halt`, `resume`, `demo`.
