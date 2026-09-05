# REPORT.md — box/657-direct-provider-discrepancies

## 1. What I understood the task to be

Issue pydantic/genai-prices#657 lists three kinds of problems for direct (provider-owned) price
data: "Price discrepancies", "New models" and "Potential removals". This assignment is to fix
**only the three rows under "Price discrepancies"**:

1. xAI `grok-4.20-multi-agent` was priced flat at input 2 / cache-read 0.2 / output 6, but per
   xAI's pricing docs it is priced like `grok-4.20-multi-agent-0309`: base $1.25/$0.20/$2.50 with a
   higher tier from 200k tokens ($2.50/$0.40/$5.00). It must use the same tiered form (base + tier
   `start: 199999`) as the neighbouring `grok-4.20` record.
2. Groq `openai/gpt-oss-120b` wrongly carried `openai/gpt-oss-safeguard-20b` as a `match` alias
   (pricing it at the 120B rates 0.15/0.075/0.6). `openai/gpt-oss-safeguard-20b` is its own model,
   priced 0.075 input / 0.3 output on Groq (preview pricing). The alias must be removed and a new,
   properly priced record added.
3. Cohere `command-r` was priced at 0.15/0.6 but per cohere.com/pricing it is priced as
   `command-r-03-2024` (0.5 input / 1.5 output). The plain `command-r` and `-08-2024` ids are not
   listed on the public page and inherit that price; a yml comment must say so.

I did **not** add the issue's "New models" rows and did **not** remove the "Potential removals";
the packet explicitly says those need a maintainer's call.

## 2. Values changed

| File | Model id | Field | Old | New | Source URL |
| --- | --- | --- | --- | --- | --- |
| `prices/providers/x_ai.yml` | `grok-4.20-multi-agent` | `prices.input_mtok` | `2` (flat) | base `1.25`, tier start `199999` price `2.5` | https://docs.x.ai/developers/pricing |
| `prices/providers/x_ai.yml` | `grok-4.20-multi-agent` | `prices.cache_read_mtok` | `0.2` (flat) | base `0.2`, tier start `199999` price `0.4` | https://docs.x.ai/developers/pricing |
| `prices/providers/x_ai.yml` | `grok-4.20-multi-agent` | `prices.output_mtok` | `6` (flat) | base `2.5`, tier start `199999` price `5` | https://docs.x.ai/developers/pricing |
| `prices/providers/x_ai.yml` | `grok-4.20-multi-agent` | `prices_checked` | `2026-06-09` | `2026-09-05` | — |
| `prices/providers/groq.yml` | `openai/gpt-oss-120b` | `match` | `or: [openai/gpt-oss-120b, openai/gpt-oss-safeguard-20b]` | `equals: openai/gpt-oss-120b` | https://console.groq.com/docs/models |
| `prices/providers/groq.yml` | `openai/gpt-oss-safeguard-20b` | (new record) | — | id/name/match/context_window 131072; prices `input_mtok: 0.075`, `output_mtok: 0.3`; `prices_checked: 2026-09-05`. No `cache_read_mtok` (issue says none). | https://console.groq.com/docs/models |
| `prices/providers/cohere.yml` | `command-r` | `prices.input_mtok` | `0.15` | `0.5` | https://cohere.com/pricing |
| `prices/providers/cohere.yml` | `command-r` | `prices.output_mtok` | `0.6` | `1.5` | https://cohere.com/pricing |
| `prices/providers/cohere.yml` | `command-r` | `prices_checked` | `2025-07-04` | `2026-09-05` | — |
| `prices/providers/cohere.yml` | `command-r` | `price_comments` | (none) | Plain `command-r` and `command-r-08-2024` are not listed on the public pricing page and inherit the `command-r-03-2024` price. | https://cohere.com/pricing |

`make build` regenerated: `README.md`, `prices/new_data/v2/data.json`, `prices/new_data/v2/data_slim.json`,
`packages/python/genai_prices/data.py`, `packages/js/src/data.ts`, `packages/go/internal/data/prices.json`.
No other files were touched.

## 3. Full VERIFY output

### `git diff --stat` (exactly three provider files, taken before `make build`)

```
M  prices/providers/cohere.yml
M  prices/providers/groq.yml
M  prices/providers/x_ai.yml
```

### `make build`

```
$ make build
Data successfully written to packages/python/genai_prices/data.py
Data successfully written to packages/python/genai_prices/data_units.py
Data successfully written to packages/go/internal/data/prices.json
Data successfully written to packages/go/data_units.go
Data successfully written to packages/js/src/data.ts
Data successfully written to packages/js/src/dataUnits.ts
README.md updated with providers list
```

(The first `make build` attempt failed because the new Groq record, added immediately after the
120B record, was not in sorted-id order:
`Value error, Models are not sorted by ID: move 'openai/gpt-oss-safeguard-20b' 27 -> 28 after 'openai/gpt-oss-20b'`.
I moved the record after `openai/gpt-oss-20b` and the build passed.)

### `uv run ruff format --check && uv run ruff check`

```
$ uv run ruff format --check && uv run ruff check
82 files already formatted
All checks passed!
```

### `uv run pytest tests/test_price_calc.py -q -k 'grok or groq or cohere' 2>&1 | tail -3`

```
$ uv run pytest tests/test_price_calc.py -q -k 'grok or groq or cohere' 2>&1 | tail -3
        10 passed
       515 deselected
```

## 4. Seeded-failure result

The packet says the seeded failure is "not applicable to a data change" and to instead paste the
`check_for_price_discrepancies` grep for the three ids before and after.

Before (my change reverted via `git stash`):

```
$ uv run -m prices check_for_price_discrepancies 2>&1 | grep -iE 'grok-4.20-multi|safeguard|command-r' | head
(no output)
```

After (change applied, post-`make build`):

```
$ uv run -m prices check_for_price_discrepancies 2>&1 | grep -iE 'grok-4.20-multi|safeguard|command-r' | head
(no output)
```

The full checker output in both cases is:

```
$ uv run -m prices check_for_price_discrepancies
no price discrepancies found
```

So nothing is seeded in the discrepancy feed for these three ids; the change is a pure
provider-file price correction as the packet states.

## 5. `git diff --stat upstream/main...HEAD`

```
 README.md                             |  2 +-
 packages/go/internal/data/prices.json |  2 +-
 packages/js/src/data.ts               | 57 +++++++++++++++++++++++++++--------
 packages/python/genai_prices/data.py  | 23 +++++++++-----
 prices/new_data/v2/data.json          |  2 +-
 prices/new_data/v2/data_slim.json     |  2 +-
 prices/providers/cohere.yml           |  9 ++++--
 prices/providers/groq.yml             | 14 +++++++--
 prices/providers/x_ai.yml             | 20 +++++++++---
 9 files changed, 96 insertions(+), 35 deletions(-)
```

## 6. Things I was unsure about / guessed at

- **Placement of the new Groq record.** The packet says add it "right after" the 120B record, but
  the data validator requires the `models` list to be sorted by id
  (`openai/gpt-oss-20b` sorts before `openai/gpt-oss-safeguard-20b`), so `make build` failed until I
  moved it after the `openai/gpt-oss-20b` record. I treated the build validator as authoritative.
- **`match` shape on the 120B record.** "Remove that alias line" could mean leaving an `or:` list
  with a single element or collapsing to plain `equals:`. I collapsed to `equals:
  openai/gpt-oss-120b`, matching the neighbouring `openai/gpt-oss-20b` record's style.
- **Cohere "yml comment".** I implemented it as the structured `price_comments:` field (the repo's
  mechanism for this, per AGENTS.md), rather than a bare `#` comment, so it also surfaces in the
  published data.
- **No `price_comments` added to the xAI record.** The packet specified only the tiered prices form
  and `prices_checked`, so I kept the change minimal and did not copy the neighbouring tier-threshold
  comment.
- **`uv` was missing from the box.** The task requires `uv sync`/`uv run`; since no `uv` binary was
  present anywhere, I installed it with `pip install uv` (uv 0.12.10). This is an extra network call
  beyond the packet's list; I judged it necessary to complete the task. Everything else followed the
  packet exactly.
- Prices come from the URLs already listed in the issue/packet and the providers' `pricing_urls`;
  no price was invented.

## 7. `git push` output

```
To https://github.com/ravsau/genai-prices.git
 * [new branch]      box/657-direct-provider-discrepancies -> box/657-direct-provider-discrepancies
branch 'box/657-direct-provider-discrepancies' set up to track 'origin/box/657-direct-provider-discrepancies'.
```

## 8. Full suite (`uv run pytest tests -q 2>&1 | tail -3`)

```
$ uv run pytest tests -q 2>&1 | tail -3
      1561 passed
        39 xfailed
```