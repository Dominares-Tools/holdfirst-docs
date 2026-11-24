# holdfirst — Stop AI jobs before the prepaid credit does

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/holdfirst**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

holdfirst sets aside the worst-case cost of each AI call before your code sends it, and refuses the call that could pass your ceiling, a per-job cap or the credit's expiry date. The ceiling follows what your billing page says is left. It calls no model, holds no key and needs nothing beyond the standard library.

## Get it

Buy holdfirst at https://dominares.org/tools/holdfirst: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the holdfirst command, an example price table and caps file, a runnable example and the test suite.

Requirements: Python 3.10 or newer, standard library only, on macOS or Linux. Your own vendor SDK makes the calls; the demo needs none.

This page documents version 1.0.0. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

You have a prepaid credit with an AI vendor: a grant, a trial balance, a monthly allowance. It runs out, or it
expires, and scripts that run by themselves (nightly jobs, evals, backfills, agents) are what spend it. Counting
the cost *after* each call finds out too late: forty workers can each see "there is money left" and together
spend it three times.

holdfirst turns that around. Before a call is sent, its worst case (every input token at the dearest rate it
could be charged, and the whole output allowance) is **held**. If spent + held + this call would pass the ceiling,
a per-job cap, or the credit's expiry, the call is refused before it leaves your machine. After the vendor answers,
the real cost goes into an append-only ledger and the hold is released.

It is a library and a small command. It never talks to a vendor and holds no key: your own code makes the call.
The runtime is the Python standard library only.

### Set it up

**1. Write your price table once**, from your vendor's price page, as `prices.json` in the budget folder
(dollars per million tokens; there is a longer example in `examples/prices.example.json`):

```json
{"models": {"big-model":   {"in": 5.0,  "out": 25.0, "cache_read": 0.5, "cache_write": 6.25},
            "small-model": {"in": 0.25, "out": 1.25, "batch_factor": 0.5}}}
```

holdfirst ships no prices of its own. They change, and a stale built-in table would under-count without telling
you. A model with no row is priced at the highest rate of every column in your table, so a surprise can only
over-count.

**2. Tell it what the billing page says**: the credit left right now, and when it expires.

```bash
holdfirst --dir spend grant --remaining 25.00 --expires 2030-03-11T00:00:00Z --buffer 2.50 --name "prepaid credit"
```

**3. Wrap each call.**

<!-- quickstart:start -->
```python
from holdfirst import Budget, Refused

budget = Budget("spend")

def summarise(prompt):
    try:
        with budget.spend("nightly:digest", model="small-model", input_text=prompt, max_output_tokens=800) as call:
            reply = send(prompt, max_tokens=800)      # your own SDK call
            call.record(reply.usage)                  # the usage object the SDK returned
            return reply.text
    except Refused as why:
        print(f"not sent: {why}")
        return None
```
<!-- quickstart:end -->

The name `"nightly:digest"` is the job. Names are yours; caps match them by prefix.

### How the ceiling is worked out

The ceiling is the most the ledger may total right now.

| You have | The ceiling is |
|---|---|
| a `grant.json` | the ledger total at your reading + the credit left at your reading - your buffer |
| a `grant.json` that has expired | the ledger total at your reading: no new spend at all |
| no `grant.json`, and `Budget(folder, max_usd=50)` | 50 |
| neither | nothing is allowed (`NoCeiling`): holdfirst will not guess a limit |

A reading is a moment you looked at the billing page. Take a new one whenever you like (`holdfirst grant ...`
again): it re-bases the sum on what the vendor says, so drift between your ledger and their meter never builds
up. The buffer is how far before empty you want to stop.

`HOLDFIRST_MAX_USD` in the environment can lower the ceiling for one process or one CI job. It can never raise it,
and by itself it is not a ceiling.

### Per-job caps

`caps.json` in the budget folder, edited by hand, read again whenever it changes:

```json
{"nightly":    {"month_usd": 20},
 "evals:":     {"total_usd": 50, "since": "2030-03-01"},
 "evals:tank": {"month_usd": 5, "note": "the slow suite"}}
```

A cap covers every job whose name starts with its prefix at a `:` or `-` boundary: `evals:tank` covers
`evals:tank`, `evals:tank:judge` and `evals:tank-2:judge`, and never `evals:tanker`. A job is checked against
every prefix it falls under. `month_usd` is the current UTC month; `total_usd` is everything since `since` (or
since the start). Money held for calls in flight counts against the caps it falls under.

### Batches and other calls that outlive the process

A hold normally ends with the process that took it, so a crashed script does not lock money away. A batch you
will collect tomorrow needs a hold that survives:

```python
token = budget.hold(worst_case_usd, "evals:batch", keep=True, note="400 requests")
# ... later, in any process, when the results are in:
for result in results:
    budget.record("evals:batch", model=result.model, usage=result.usage, batch=True)
budget.release(token)
```

`budget.estimate(model, input_text=..., max_output_tokens=..., batch=True)` gives the worst case of one request.

### When the vendor says the credit is gone

Your ledger is an estimate of their meter. When the vendor itself refuses for lack of credit, stop everything:

```python
budget.halt("vendor says the credit balance is too low")
```

Every process using that folder is refused from then on (`Halted`) until someone runs `holdfirst resume`.

### The command

```
holdfirst status [--json]          ceiling, spent, held, left, and spend against every cap
holdfirst grant --remaining 25 --expires 2030-03-11T00:00:00Z [--buffer 2.5] [--name ...]
holdfirst check --job nightly:digest --usd 0.40     exit 0 if it would be allowed now, 1 if refused
holdfirst halt [reason]            refuse everything
holdfirst resume                   lift a halt
holdfirst demo                     every refusal in a throwaway folder; sends nothing
```

The folder is `--dir`, else `HOLDFIRST_DIR`, else `./.holdfirst`. `--max-usd` sets a fixed ceiling for folders
without a grant. Exit codes: 0 fine, 1 refused, 2 bad input. `holdfirst check` is for shell scripts and CI:
`holdfirst check --job backfill --usd 12 && python backfill.py`.

### What the ledger holds

One JSON line per billed call: when, the job, the model, token counts, dollars, and a request id if you pass
one (`ref=`). It never holds a prompt, a reply or a key. A call that was made and never recorded is written at
its full held amount and marked `"assumed": true`, because a call must never look free.

The sum is kept as the file grows: each process reads only the lines appended since it last looked, and reads
the whole file again if it was replaced, shortened or edited in place.

### It fails closed

| What is wrong | What happens |
|---|---|
| `grant.json` unreadable or malformed | every call refused (`GrantInvalid`), even with a `max_usd` |
| the credit has expired | every call refused (`GrantExpired`) |
| `caps.json` unreadable or malformed | every call refused (`CapsInvalid`), for every job |
| a ledger line or `holds.json` cannot be read | every call refused (`LedgerInvalid`) |
| no price table, or a malformed one | a call priced from the table is refused (`PricesInvalid`); a call given in dollars still works |
| `HOLDFIRST_MAX_USD` is not a number | every call refused |
| the `HALT` file is present | every call refused (`Halted`) |

Every one of these is a `holdfirst.Refused`, so one `except` catches them all.

### Reading usage

`call.record(reply.usage)` reads the usage shapes of the common SDK families by their field names: chat-completions
style (`prompt_tokens`, `completion_tokens`), responses style (`input_tokens` with `input_tokens_details`),
messages style (`input_tokens` with `cache_read_input_tokens` and `cache_creation_input_tokens`) and
generate-content style (`prompt_token_count`, `candidates_token_count`, `thoughts_token_count`,
`tool_use_prompt_token_count`). It knows which of them count cached tokens and cache writes inside the input and
which beside it. Any other shape raises `UsageUnknown`: pass the
dollars yourself with `call.record(usd=...)` rather than have holdfirst guess. Fees that are not tokens (a
per-search charge, an image) go in `extra_usd=`, on the hold and on the record.

### Limits

- **It does not read your billing page.** You tell it the credit left; it paces the work from there. Between
  readings its number is an estimate of the vendor's.
- **The hold is an upper estimate, not a quote.** Input is sized from characters (three per token by default,
  which runs high for English) unless you pass a token count. It is built to over-count.
- **It does not meter your customers** or bill anyone. It guards one bill: yours.
- **It does not retry, route or call a model.** It answers one question, "may this call be sent?", and keeps the
  ledger.
- **macOS and Linux only.** It uses POSIX file locks. If the budget folder is on a network mount, pass
  `Budget(folder, lock_dir="/some/local/folder")`: file locks are unreliable on many network file systems.

### Tests

```bash
pip install pytest && python -m pytest
```

---

Docs mirror of the private repository Dominares-Tools/holdfirst; updated with each release.
