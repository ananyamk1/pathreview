# Module 3 Journal — Ananya

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/153

**Issue title:** Faithfulness checker crashes when a context chunk has `text: None`

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The RAG evaluation suite's faithfulness checker (`rag/evaluator/faithfulness_checker.py`)
is meant to score how well generated feedback is supported by the retrieved context
chunks. It builds one big context string by joining the `text` field of every chunk.
The bug is that it does this with `chunk.get("text", "")`, and `dict.get`'s default
only applies when the key is *absent* — if a chunk explicitly has `text: None`, `.get`
returns `None`, and `" ".join([... None ...])` then raises a `TypeError`. So any
retrieval result containing a null-text chunk crashes the checker instead of skipping
that chunk. A successful fix would coerce missing/`None` text to an empty string (or
filter such chunks out) so scoring proceeds normally, plus a unit test covering the
`text: None` case.

**Branch name:** fix/153-faithfulness-checker-none-text

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

### "Is this right for me?" — scope reasoning
- **Tier 1 / good first issue:** labeled `tier-1`, `good first issue`, `bug` — appropriate
  for a first contribution to a large codebase.
- **Small, well-bounded blast radius:** the defect is a single line
  (`faithfulness_checker.py:34-36`); the fix is localized and won't ripple across modules.
- **Reproducible & testable:** the failure is a deterministic `TypeError` on a known
  input shape (`{"text": None}`), so it's straightforward to write a regression test.
- **Matches the codebase area I want to learn:** touches the `rag` evaluation code and
  the pytest unit-test layout, which is a good on-ramp to the project's testing patterns.
- **No external services required to reproduce/fix:** the checker is pure Python with no
  DB or API-key dependency, so I can iterate without the full Docker stack.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/ananyamk1/pathreview/commit/7f533b777dad3626a09864477a86bef4fd9cc33a

**Reproduction summary:**
Running the existing unit test `test_none_context_chunk_text` against
`FaithfulnessChecker.check` locally raised
`TypeError: sequence item 0: expected str instance, NoneType found` at
`rag/evaluator/faithfulness_checker.py:44`, confirming that a context chunk of
`{"text": None}` crashes the checker because `dict.get("text", "")` returns
`None` (its default only applies to absent keys) and `" ".join([... None ...])`
then fails. I documented the reproduced defect with an inline comment at the
crash site.

**PLAN.md link:** https://github.com/ananyamk1/pathreview/blob/fix/153-faithfulness-checker-none-text/PLAN.md

**Walkthrough video (recommended):** N/A

**Blockers or open questions:**
Deciding whether the fix should coerce null/missing text to `""` (keeps chunk
count/order, minimal change) or filter such chunks out entirely — leaning toward
coercion plus a `structlog` warning so null chunks don't silently disappear.
Also unsure whether non-string `text` values ever occur in real retrieval
output; the plan handles them defensively regardless.
