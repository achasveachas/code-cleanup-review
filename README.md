# code-cleanup-review

A fresh-context cleanup review for AI-written code, packaged as an [Agent Skill](https://agentskills.io).

AI coding agents are good at red-green: they write a test, make it pass, and stop. The refactor step gets skipped, and the agent that wrote the code still has its dead ends in context, so it does not see them as leftovers. This skill gives that step to a reviewer who never saw the author's conversation and reads only the diff, plus the rest of the repository where the scope calls for it.

## What it checks

- Leftovers from abandoned approaches: dead code, unused parameters and helpers, commented-out code, stale TODOs
- Debug output: prints, console logs, temporary assertions
- Comments that restate the code or carry stale rationale
- Duplication, inside the change and against code already in the repository
- Verbosity: pass-through wrappers, single-use helpers, needless temporaries, wordy names and messages
- Pattern drift: new code that does something differently from the existing codebase

Every pattern mismatch is labelled CONFORM (change the new code to match) or BETTER-PATTERN (the new approach is better). BETTER-PATTERN items are never spread silently; they are raised as a decision for a human.

The reviewer only reads and reports. It never edits files.

## Scopes

| Scope | Use it when | The reviewer reads |
|---|---|---|
| SUBTASK | One part of a larger feature has passing tests | The diff of that change, judged on added or modified lines only |
| STANDALONE STORY | The change is a whole story on its own | The change's diff, plus the rest of the repo to find duplication and pattern drift |
| EPIC | A feature branch is complete | The whole branch diff against its base, compared with the surrounding codebase |

## What you get back

A findings list, each with an id, a severity (P1 fix before merge, P2 worth fixing, P3 optional polish), a `file:line`, the evidence, the smallest safe fix, and whether it is in scope. Then a list of decisions for the human. If there is nothing to report, the answer is exactly `No findings.`

```text
F2  P1  src/export/csv.py:41
evidence: `def _join_cells(cells): return ",".join(cells)` is called once, from
  write_row() at line 58, and adds no behavior beyond the expression.
fix: inline the join in write_row() and delete _join_cells.
scope: in-scope
```

## Install

pi:

```bash
pi install git:github.com/achasveachas/code-cleanup-review
```

For Claude Code and other tools, clone or download this repo first (`git clone https://github.com/achasveachas/code-cleanup-review`), then copy `skills/code-cleanup-review` as below.

Claude Code: copy `skills/code-cleanup-review` into `~/.claude/skills/` (all projects) or `.claude/skills/` (one project).

Other tools that support Agent Skills: copy the `skills/code-cleanup-review` folder into the tool's skills directory.

## Usage

Ask your agent to run a fresh reviewer with the skill on a diff:

```text
Run a fresh reviewer with the code-cleanup-review skill on `git diff main...HEAD`, EPIC scope.
```

The reviewer must not be the agent that wrote the code, and it needs a base to diff against; if you do not name one, it will ask. In pi you can also load the skill directly with `/skill:code-cleanup-review`.

After the review, apply the accepted in-scope findings, re-run the tests, and record what was applied, what was declined and why.

## Make it stick

Optional. A review that nobody has to run gets skipped. If your workflow has a step that creates a pull request, you can make it refuse to proceed until a cleanup review has been recorded for the change, whether that is a clean result or a list of applied and declined findings. The skill does not require this; it works fine as a manual step.

## License

MIT. See [LICENSE](LICENSE).

## Credits

Created by [Yechiel Kalmenson](https://yechiel.me)

The checklists are adapted from the cleanup prompts in [pi-subagents](https://github.com/nicobailon/pi-subagents) by Nico Bailon (MIT). See [NOTICE](NOTICE).
