# Why the review loop works the way it does

These notes record what went wrong in practice and which rule in `SKILL.md` each failure produced. Read them when a rule seems not to fit your situation, before bending it.

## Rounds did not converge when every round reviewed the whole PR

A PR was taken through 21 rounds of review and fixes. The findings never stopped and the diff grew to more than twenty files, so it was abandoned and redone from scratch. On a later PR, whole-PR review gave 2, then 6, then 6 findings.

The cause: each round's fixes add code (tests, guards, docs), and a whole-PR review treats that new code and its neighborhood as fresh territory. Most of the later findings were reasonable ideas that widened the PR's scope, not errors in it.

→ **Round 2 onward reviews only the delta since the previous round** (§1).

Filtering by location alone was tried first and kept every finding. The fix had rewritten most of the test file, so all six findings sat on changed lines. Adding the **kind** filter (keep defects the delta introduced, drop scope widening) brought that round to zero, and the round before it from six findings to one: a doc sentence the delta had written that was false.

## Another session fixed the same findings at the same time

The PR's creator was a Claude session subscribed to its events. Two minutes after the harness posted its comments, that session had pushed fixes for two of them, replied, and resolved the threads, while the harness was preparing its own fixes for the same findings. Pushing both would have conflicted, and the replies would have doubled.

→ **Check for a fixer session before posting** (§0). When one exists, only review and comment, and let it fix.
→ **Check again after posting and right before pushing** (§2, §3), for fixers the session list cannot see.

Detecting the fixer by the PR's head branch in `get_session` (`session_context.outcomes[].git_info.branches` or `external_metadata.current_branches`) worked on the first try.

## Waiting for the fixer: the first push is not the last

The fixer replied and pushed per finding, in several pushes over a few minutes. Re-reviewing on the first push would review a half-finished round.

→ **Re-review only once the fixer is no longer `working`** (§6), and never act on deployment-bot or CI events alone.

The watch loop ran three rounds end to end this way with no user input.

## The CI review bot skips PRs Claude has commented on

The upstream code-review plugin (used by `anthropics/claude-code-action`) stops early if "Claude has already commented on this PR". Once this harness posts, CI will never review that PR again, so a fix cannot be re-checked by CI.

→ **Re-review is the harness's job** (§0 "CI review"). Threads the CI bot left open are handled as ordinary unresolved threads in §2.

## The level must be passed every round

`/code-review` reuses the last level typed when none is given, so a forgotten argument silently runs at whatever level was used last (possibly `max`).

→ **Always pass the effort explicitly** (§0).

## Comments are posted under the user's account

Comments made through the session's GitHub credentials appear as the user, not as a bot, even with the Claude footer. A fixer session reads them as the PR owner's own review. There is currently no way to post as a bot from a session; the footer is the only marker.

## Known gaps, not yet addressed

- The last allowed round still posts comments. A fixer will fix them, but nothing re-reviews those fixes.
- `/code-review` picks its comment language per run; rounds of the same PR have come out in different languages. Passing `language` explicitly is meant to address this.
