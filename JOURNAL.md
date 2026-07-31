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

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the core fix from PLAN.md. Done so far:
- Sub-task 1 (fix the coercion in `check`): replaced
  `" ".join([chunk.get("text", "") ...])` in
  `rag/evaluator/faithfulness_checker.py` with a loop that keeps non-empty
  string texts, skips `None`/empty values (logging a `structlog` count), and
  `str()`-coerces any non-string value.
- Sub-task 2 (guard the return path): confirmed an all-`None`/empty context now
  returns a valid `0.0` instead of raising.
- Sub-task 3 (turn the None test green) and sub-task 4 (regression coverage):
  rewrote `test_none_context_chunk_text` as an explicit `#153` regression test
  and added `test_mixed_none_and_valid_chunks`, `test_all_none_chunks_returns_zero`,
  `test_empty_string_text_chunk`, and `test_non_string_text_is_coerced` in
  `tests/unit/test_faithfulness_checker.py`.

**Next steps:**
Sub-task 5 — run the full suite + linters, self-review against
`docs/CONTRIBUTING.md`, then open the PR and request feedback in the cohort
Slack channel.

**Blockers:**
The repo has ~53 pre-existing unit-test failures and ~181 pre-existing lint
errors unrelated to issue #153 (e.g. `test_skill_extractor.py`,
`test_tech_detector.py`, and 3 pre-existing faithfulness scoring-threshold
tests). I baselined these before starting so I can show my change introduces no
new failures; not a blocker for the fix itself.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/462

**Branch:** `fix/153-faithfulness-checker-none-text`

**What you built:**
`FaithfulnessChecker.check` crashed with
`TypeError: sequence item 0: expected str instance, NoneType found` whenever a
retrieved context chunk was `{"text": None}`, because `dict.get`'s default only
applies to absent keys. The fix coerces each chunk's text to a string —
skipping `None`/empty values and `str()`-coercing non-strings — so faithfulness
scoring proceeds over whatever valid text remains instead of aborting on one
null chunk.

**Tests added or updated:**
`tests/unit/test_faithfulness_checker.py` — rewrote `test_none_context_chunk_text`
as an explicit regression test for the `#153` `TypeError`, and added four tests:
`test_mixed_none_and_valid_chunks` (a `None` chunk is skipped while a valid chunk
still supports the claim), `test_all_none_chunks_returns_zero` (all-`None` context
returns `0.0`), `test_empty_string_text_chunk` (empty-string text is no-support,
not a crash), and `test_non_string_text_is_coerced` (a numeric `text` is coerced,
not crashed).

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
*(Interpreted per the assignment's pre-existing-failures rule: the repo has
~53 pre-existing unit-test failures and ~181 pre-existing lint errors unrelated
to #153. Baselined before my change and re-checked after — my change introduces
no new failures. My two touched files pass `ruff`, `black`, and `mypy`
individually. Details in the PR's "Notes for Reviewers".)*

**Draft PR feedback received from:** none
