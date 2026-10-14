---
name: code-cleanup-review
description: Fresh-context cleanup review of AI-written code before it ships. Use after a change's tests pass (SUBTASK scope - only that change's diff; STANDALONE STORY scope - a change that is a whole story, so its diff plus duplication and pattern checks against the repo) or when a feature/epic branch is complete (EPIC scope - the whole branch diff against its base, read against the surrounding codebase). Finds dead-end leftovers, duplication, verbosity and pattern drift; returns evidence-backed findings. Review-only, never edits.
license: MIT
---

# Code cleanup review

## Purpose and when to run

AI-written code that passes its tests still tends to carry leftovers: dead code from abandoned approaches, restating comments, needless wrappers, copies of logic that already exists elsewhere. This review is the Refactor step, done by someone other than the author.

Three scopes:

- **SUBTASK**: run after one change's tests pass, before it is submitted for merge or review.
- **STANDALONE STORY**: a change that is a complete story on its own, not one part of a larger feature. Use it when the change is the whole story, including a part of a larger feature that will get no feature-level pass; use SUBTASK for a part of a larger feature, which stays narrow because the feature-level EPIC pass covers the codebase checks later.
- **EPIC**: run once a feature or epic branch is complete, before it closes.

Rules for the reviewer:

- The writer of the code must not be the reviewer.
- Run with fresh context: a fresh reviewer with no access to the author's conversation. Inspect the files, the diff and the project's own instructions directly.
- Review only. Do not edit files. Return findings in the format below.
- Treat linter, scanner and tool output as leads, not verdicts. Report only concrete issues you can point to.

## Inputs

- **SUBTASK** and **STANDALONE STORY**: the change's diff (STANDALONE STORY also needs read access to the rest of the repository), for example `git diff <base>..HEAD -- <paths>` or `git show <sha>`.
- **EPIC**: `git diff <base>...<branch>` (three dots: changes since the merge-base), plus read access to the rest of the repository for comparison.

If the base or branch is not stated, ask for it; do not guess.

If the reviewer cannot run git (read-only reviewers often have no shell), the caller must paste the diff into the request: `git show <sha>` for SUBTASK or STANDALONE STORY, `git diff <base>...<branch>` for EPIC, or `--stat` plus per-file diffs when it is large. Without the diff, the reviewer must say which findings it could not confirm as added-vs-pre-existing, and must not guess.

## Scope rules

- **SUBTASK** judges only lines the change added or modified.
- **STANDALONE STORY** judges the change's own lines like SUBTASK, but may report duplication of, or inconsistency with, existing code elsewhere. Fixes to pre-existing code follow the EPIC rule: propose them and list them under **Decisions for the human**; never fix untouched code silently.
- **EPIC** judges the whole branch diff. It may propose changes to pre-existing code when the branch duplicates or conflicts with it, because that cleanup is its own reviewable change.
- Pre-existing code that the change itself edited (touched in the diff) is in-scope for the touched lines only. Untouched pre-existing copies of the same problem are out-of-scope at SUBTASK (file a new issue); at STANDALONE STORY or EPIC, report them as `out-of-scope: file a new issue` and also list them under **Decisions for the human** as a scope call (fold into this cleanup, or a separate issue).
- In every scope, anything outside scope is reported as `out-of-scope: file a new issue`, never fixed in place.
- Bug-fix changes stay minimal: no adjacent refactors, even if the neighbouring code is untidy.

## SUBTASK checklist

Look for these in the added or modified lines:

- Leftovers from abandoned approaches: dead code, unused parameters, imports and helpers, commented-out code, TODOs left over from the session.
- Debug output: prints, console logs, temporary assertions.
- Comments that restate the code, carry stale rationale, or are placeholder text.
- Defensive checks that hide useful errors, return vague defaults, or re-validate trusted internal data after a real boundary was already crossed.
- Type escapes: broad casts, `any`-style typing, duplicated type definitions where a local source-of-truth type exists.
- Duplication inside the change, including repeated test setup that an existing fixture covers.
- Pass-through wrappers and single-use helpers that only paraphrase an expression or enforce no invariant.
- Needless temporaries and nesting: variables that only name an obvious expression, nested returns that can be direct.
- Verbose or generic test names, docs, error messages, status text and changelog wording; prose that says the same thing twice.
- Tests that restate behavior already covered at a cheaper layer.
- Style drift from nearby code and from the project's written conventions.

Caution: shorter is better only when it is clearer and preserves behavior, error signals, cleanup semantics, useful invariants and local style. Do not recommend a cut that loses any of these.

## EPIC checklist

Apply the SUBTASK checklist over the whole branch, then add:

- Duplication across the branch's commits or parts (the same logic written twice in different places).
- Logic that duplicates code already in the repository. Search for it by name, signature and behavior; propose reusing it or extracting a shared helper.
- Inconsistency between parts of the branch: two ways of doing the same thing.
- New code that does something differently from how the existing codebase does the same thing (see pattern triage).

## STANDALONE STORY checklist

Apply the SUBTASK checklist, then add two of the EPIC checks: logic that duplicates code already in the repository, and new code that does something differently from the existing codebase (see pattern triage). Skip the cross-part checks (duplication or inconsistency between parts); there is only one part.

## Pattern triage

Give every pattern mismatch exactly one label.

- **CONFORM**: the existing pattern is fine. The fix is to change the new code to match it. Cite the existing example as `file:line`.
- **BETTER-PATTERN**: the new code's approach is better. Say concretely why (fewer failure modes, less code, clearer invariant, and so on). Do not conform the new code to the old pattern, and do not spread the new pattern to existing code on your own; that happens only if the human picks option 1 below. Raise it as a decision for the human with these options:
  1. Update the existing usages in this change or story, if that is small and in scope.
  2. Leave the existing code and file a separate ticket to migrate it.

  List the existing usages (`file:line`) so the human can size the work.

A BETTER-PATTERN with only one existing usage is still BETTER-PATTERN; the decision states how many usages exist, so small migrations are easy to approve.

Extracting a shared helper to remove duplication is a duplication finding, not a pattern finding. Label it CONFORM only when an existing helper or convention already does the job, citing it as `file:line`; otherwise leave the pattern label off.

When unsure which label applies, use BETTER-PATTERN, so a person decides.

## Return contract

Return a findings list. Each item has:

- `id`: F1, F2, ...
- `severity`: **P1** fix before merge (dead code, debug leftovers, real duplication); **P2** worth fixing, low risk; **P3** optional polish.
- `file:line`
- `evidence`: quote or describe what is there.
- `fix`: the smallest safe fix.
- `scope`: `in-scope` or `out-of-scope: file a new issue`.
- `pattern`: for pattern items only, CONFORM or BETTER-PATTERN.

Then a **Decisions for the human** list: every BETTER-PATTERN item plus any scope calls (for example an EPIC proposal that touches pre-existing code).

If nothing is found, return exactly: `No findings.`

Example finding:

```text
F2  P1  src/export/csv.py:41
evidence: `def _join_cells(cells): return ",".join(cells)` is called once, from
  write_row() at line 58, and adds no behavior beyond the expression.
fix: inline the join in write_row() and delete _join_cells.
scope: in-scope
```

Example pattern finding:

```text
F5  P2  src/api/handlers/report.py:12
evidence: opens its own DB session with try/finally; every other handler uses
  the `with_session` decorator (src/api/handlers/user.py:9).
fix: wrap the handler with `with_session` and remove the manual session code.
scope: in-scope
pattern: CONFORM
```

## After the review

For whoever applies the findings (usually the author):

1. Apply the accepted in-scope findings. Do not apply P3 items or out-of-scope items unless asked.
2. Re-run the tests. They must still pass; if a fix breaks one, revert that fix and decline it.
3. Record the outcome: which findings were applied, which were declined and why, which decisions went to the human, or that the review was clean (`No findings.`).
4. File separate issues for every `out-of-scope` finding and for any BETTER-PATTERN decision the human sends to option 2.

The checklists adapt the cleanup prompts from pi-subagents (MIT, Copyright (c) 2026 Nico Bailon); see NOTICE.
