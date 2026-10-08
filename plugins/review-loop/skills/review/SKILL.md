---
name: review
description: Review loop around the built-in /code-review. Runs "review → post findings on the PR → fix and push → re-review" until a round finds nothing (max 3 rounds by default). From round 2 on it posts only defects introduced since the previous round, so rounds converge. If another session owns the PR branch, it only reviews and comments, watches the PR, and re-reviews after that session pushes. Use when asked to review a PR, run a review loop, or address review findings.
argument-hint: "[PR number] [low|medium|high|xhigh|max] [--rounds N] [--no-fix]"
---

# review — review-loop harness

One round:

1. Review with `/code-review` and post the findings as inline PR comments.
2. Fix each unresolved thread, or reply with why it stays as is.
3. Run the project's verification commands, then commit and push.
4. Re-review and go back to 1.

The review itself is `/code-review`'s job. This skill decides what to pass it, how to handle its findings, and when to stop. The reasons behind each rule are in `${CLAUDE_SKILL_DIR}/design-notes.md`; read it when a rule seems not to fit.

## 0. Arguments, project settings, preconditions

### Arguments

Read these from `$ARGUMENTS`, in any order:

- **PR**: a number, `#123`, or a PR URL. If absent, use the current branch's open PR. If there is none, create one as a draft.
- **effort**: `low|medium|high|xhigh|max`. If absent, use the project setting, else `medium`. `/code-review` reuses the last level typed when none is given, so **always pass one explicitly, every round**.
- **`--rounds N`**: round limit. If absent, use the project setting, else **3**.
- **`--no-fix`**: review and comment in round 1 only; fix nothing.

### Project settings

Read the `## Review harness` section of the repository's `CLAUDE.md` (root first, then the nearest one to the changed files). The keys and defaults are in `${CLAUDE_SKILL_DIR}/config.md`. Any key that is missing falls back to its default.

If `verify` is missing, infer the commands from the repository: `package.json` scripts (`lint`, `typecheck`, `test`, `build`), a `Makefile`, `pyproject.toml` / `tox.ini`, `Cargo.toml`, `go.mod`, and so on. Before round 1, tell the user in one line which commands you will run and that they were inferred.

### Preconditions

- The working tree is on the PR's head branch with no uncommitted changes. If not, check it out or ask the user. Never apply fixes to a different branch's working tree.
- Pull the head so it is current (someone else may have pushed).

### Is another session already fixing this PR?

A Claude session that created the PR usually subscribed to its events. If so, the moment you post comments it wakes and starts fixing the same findings. Two writers on one branch collide and reply twice.

Before §1, list your own sessions (`mcp__claude-code-remote__list_sessions`, `mine: true`). For each other non-archived session, call `get_session`. If `session_context.outcomes[].git_info.branches` or `external_metadata.current_branches` contains **this PR's head branch**, treat that session as the **fixer**.

If there is a fixer:

- Before round 1, tell the user which session it is (title, ID, status).
- After §1, skip §2 and §3 and go to §6: watch the PR and wait for the fixer's push. When it settles, start the next round automatically.
- Never push, reply, or resolve on this PR from this harness.

Where `list_sessions` is unavailable (a local CLI, for example), this check cannot run. The after-the-fact check at the start of §2 is the only guard.

### CI review

If the project setting `ci_review` is `claude-code-action`, an automatic Claude review also runs on the PR. That upstream plugin skips any PR Claude has already commented on, so once this harness posts, CI will not review the PR again; re-review is this harness's job. Threads that the CI bot left open are handled in §2 like any other unresolved thread.

If `ci_review` is `none` (the default), nothing reviews PRs automatically. A PR with no bot comments has not been reviewed.

## 1. Review and comment

In every round, remember the head SHA you reviewed. The next round uses it to compute "the delta since the previous round".

### Round 1: the whole PR

Call the `code-review` skill with:

```
<effort> <PR> --comment
```

Pass through any other flags you received (such as `--max-findings`). Tell it to write comments in the project's `language` (see `config.md`).

### Round 2 and later: the delta only

Reviewing the whole PR every round does not converge: each fix adds code, and the new code and its surroundings attract new findings. From round 2 on, handle only **the delta from the previous round's head to the current head, and problems that delta introduced**.

`/code-review` cannot be given a commit range, so review the whole PR and filter before posting:

1. Call `code-review` **without `--comment`**: `<effort> <PR>`. Take the JSON findings (`file`, `line`, `summary`, `failure_scenario`).
2. Compute the changed lines: `git diff -U0 <previous round's SHA>..<current head>`. Each `@@ … +start,count @@` marks changed lines on the new side.
3. Filter in two steps. A finding that fails either step is dropped.
   - **Location**: `file` is in the delta and `line` is within the changed lines (±3). If a finding's cause is in the delta but it surfaces elsewhere (the delta changed how existing code is called, say), re-anchor it to the causing line and keep it.
   - **Kind**: location alone is not enough; when a fix rewrites most of a file, every finding lands on a changed line.
     - **Keep — defects in the delta**: delta code that does not work, breaks existing behavior, or states something false (in a comment, doc, or commit message). Errors that exist *because* the delta was added.
     - **Drop — scope widening**: "this case should be covered too", "also add X", or asks to do now what the PR description explicitly deferred. Drop these even when they are good ideas, as long as the delta itself is not wrong.
     - When unsure, ask: "if this is left unfixed, is the PR worse than before the delta?" If not, it is scope widening.
4. Post the kept findings as one review:
   - `mcp__github__pull_request_review_write` (method `create`, `commitID` = current head) to open a pending review.
   - `mcp__github__add_comment_to_pending_review` per finding (`path`, `line`, `side: RIGHT`, `subjectType: LINE`). Write the body from `summary` and `failure_scenario` in the project's `language`, and end it with the `footer`.
   - `pull_request_review_write` (method `submit_pending`, event `COMMENT`).
   - Without the GitHub MCP tools, use `gh api repos/{owner}/{repo}/pulls/{n}/reviews` with a `comments` array.
5. Do not post dropped findings. List them in the §5 report with the reason (location / scope widening). The user decides whether they belong in this PR or another.

### Next

If the posted findings are zero (round 1: `/code-review`'s findings; later rounds: the kept ones), stop and go to §5.

With `--no-fix`, go to §5 here.

## 2. Handle unresolved threads

### First, check that nobody else is fixing

Even without a fixer from §0, a local session or a person may be fixing in parallel. Wait 2–3 minutes after posting, then check:

- `git fetch`: has the PR head moved since §1?
- Do the §1 threads have replies or resolutions from someone else?

If either is true, **leave the fixing to them**. Do not fix or push this round. Report in §5 who is fixing and stop. Re-review once their pushes settle, on the user's instruction.

### Fix or answer each thread

Fetch **all unresolved** review threads: this round's findings, leftovers from earlier rounds, CI bot findings, and human reviewers' comments.

- GitHub MCP: `mcp__github__pull_request_read` (method `get_review_comments`).
- Otherwise: `gh api graphql` with `reviewThreads { isResolved, comments { … } }`.

Decide each thread:

- **Fix**: confirm the finding against the code. Fix it if there is a real path to the failure and the fix is proportionate. Keep the change to what the finding needs.
- **Don't fix**: false positive, does not reproduce, or not worth its code. Reply in 1–2 sentences and **leave the thread unresolved**; the user decides.
- **Large asks from a human reviewer** (multi-file refactors, API or schema changes, design feedback): do not implement. Reply with a proposal and ask the user.
- **Security findings**: if not fixed, always leave the thread unresolved and tell the user.

If a thread you already declined in an earlier round comes back with the same finding, do not repeat the reason; raise it with the user.

## 3. Verify and push

If anything was fixed, run **every** command in the project's `verify` setting (or the inferred set from §0). Do not skip one; the project's `CLAUDE.md` may say why each matters.

If any fails:

1. If your fix caused it, fix that and rerun all of them.
2. If a fix cannot be made to pass, **revert that fix only** and reply on its thread as "not fixed".
3. If it was failing before your fixes (check with `git stash`), say so in the report. Do not widen the change to fix it.

Never skip, disable, or delete a test to get green. If a test listed in `protected_tests` fails, the finding is more likely wrong than the test.

When everything passes:

1. Re-read `git diff` and confirm nothing goes beyond what the findings need.
2. Commit with the project's `commit_message` (round number filled in), listing the fixed threads one per line in the body.
3. **Before pushing**, reply on each fixed thread: what changed, and that unpushed commit `<short sha>` fixes it.
   - MCP: `mcp__github__add_reply_to_pull_request_comment`
   - gh: `gh api repos/{owner}/{repo}/pulls/{n}/comments/{id}/replies -f body=...`
4. `git fetch` again right before pushing. If the head moved, do not push; someone else is fixing (as in §2). Report to the user. Never force-push.
5. Push, then resolve the fixed threads (`mcp__github__pull_request_review_write` method `resolve_thread`, or GraphQL `resolveReviewThread`).

Write replies in the project's `language` and end them with the `footer`.

If nothing was fixed this round (every thread was "don't fix"), there is nothing to push, and re-reviewing would give the same result. Stop.

## 4. Next round

If you pushed, increment the round and go back to §1.

Stop when any of these holds:

- Zero findings were posted (from round 2, counted after filtering).
- Nothing was fixed this round.
- The round limit was reached. Report what is still open.
- The same finding on the same spot came back two rounds running. The fix is missing the root cause; ask the user.

## 5. Report

Summarize briefly, in the language the user is writing in:

- Rounds run, and findings / fixes per round.
- Threads left unresolved and why (for the user to decide).
- From round 2: findings not posted as out of scope, with the reason.
- Final result of the `verify` commands.
- The final head commit.
- If you watched the PR: that the subscription was removed, and the fixer's last state.

## 6. Watch the PR for the next round (fixer is another session)

Use this only when §0 found a fixer. The fixer fixes and pushes; this harness re-reviews. This keeps that exchange going without the user re-running the skill.

### Start watching

After posting in §1:

1. Subscribe to the PR with `mcp__claude-code-remote__subscribe_pr_activity` (load it with ToolSearch if needed).
2. Remember: the round number, the head SHA reviewed in §1, the IDs of the threads posted in §1, and the fixer's session ID.
3. Schedule one check-in 30 minutes out with `mcp__claude-code-remote__send_later`, in case an event is missed or the fixer stalls. Put the remembered state in its message.
4. Tell the user in one line that you are waiting for the fixer, and end the turn.

**Ending the turn is the only way to wait.** No `sleep`, no polling. A PR event or the check-in wakes the session.

### On each wake

Check in this order:

1. **Ignore**: echoes of your own comments and replies, deployment-bot updates, CI results. End the turn.
2. **PR merged or closed**: stop watching (below).
3. **Head has not moved since §1**: if the fixer has not started or is working, keep waiting; end the turn. If the fixer is idle and the head has not moved, the round fixed nothing; stop per §4.
4. **Fixer still working** (`get_session` → `status_bucket` is `working`): more pushes may follow; end the turn. A fixer often pushes per finding, so never re-review on the first push.
5. **Fixer `blocked`** (waiting on its user, say): do not re-review. Tell the user once that it is waiting, and end the turn. Do not repeat the notice for the same state.
6. **Head moved and fixer finished**: increment the round, check §4's stop conditions, and run §1. Mention any §1 thread that got no reply.

If a check-in finds nothing changed, schedule another 30 minutes out and end the turn. After three check-ins in a row with no change, stop watching.

### Stop watching

Stop when §4 says to stop, the PR is merged or closed, the user asks, or three check-ins in a row found nothing new.

Unsubscribe with `mcp__claude-code-remote__unsubscribe_pr_activity`, delete any pending check-in with `mcp__claude-code-remote__delete_trigger`, then give the §5 report.

### Where watching is unavailable

Without `subscribe_pr_activity` (a local CLI, for example), you cannot watch. After §1, report in §5 that the fixer is handling the fixes and that the user should run this skill again once its pushes settle.
