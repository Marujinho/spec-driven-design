---
name: review-work
description: Validate implemented work — the whole branch, an independent read, behavior, performance, lint, types, tests, security standards, i18n — then open a PR
argument-hint: "[prd: path | url | text] [--standard <name | url | path>]..."
---

# Review

**Input**: $ARGUMENTS (the PRD this work was built against, if available — a path, a URL, or already in the conversation; resolve it per CLAUDE.md § Inputs. `--standard` adds a security or compliance standard to the default, repeatable — CLAUDE.md § Standards)

Runs after `/implement-work`. Checks what was actually written, fixes what's wrong, then opens the PR.

Nothing here is advisory. If you find a problem, fix it and re-run the checks.

## Process

1. **Find what changed.** Mark the stage first: `printf review-work > .claude/.stage` (CLAUDE.md § Hooks). The agent may have committed, or left everything in the working tree, or both. Check all of it:

   ```bash
   git status --porcelain          # uncommitted edits + untracked files
   git branch --show-current
   git diff $(git merge-base HEAD {base_branch})...HEAD --stat   # committed on this branch
   git diff --stat                 # unstaged
   git diff --cached --stat        # staged
   ```

   Untracked files are the ones that get missed — `git diff` doesn't show them, so a new file the agent wrote goes unreviewed. Take the union of all of the above.

   The scope is the whole branch against `{base_branch}`, not the last task or the last PRD. The PR reviewer, and any review agent in CI, reads the full diff; a review that saw only the newest commits misses what the earlier ones left. List every changed file and read each one.

   Read the changed code, not the whole repo — pre-existing issues are out of scope unless they're Critical or the change made them reachable.

   Read the PRD if given. Its Definition of Done and security tables are the checklist; anything listed there and missing is a finding. Standards named in its header are in scope here too, whether or not they were passed again. If the implementation left a report or plan listing files it touched, diff that list against what git actually shows — files claimed but unchanged, or changed but unclaimed, are both worth knowing about.

2. **Static checks.** Find the real commands (`package.json` scripts, `Makefile`, CI config) — don't assume a toolchain. Run lint, type check, and tests, capturing each.

3. **Independent read.** You wrote this code, so you share its blind spots. Dispatch a fresh subagent that has never seen your reasoning. Give it only the full branch diff, the PRD, and this brief: "Find bugs, security holes, and business rules the server does not enforce. For each finding give the file and line, a concrete failure scenario, and a severity." Do not give it your conclusions.

   Verify every finding against the code before acting on it:
   - Confirmed findings join this review.
   - A false positive is recorded with the `file:line` that disproves it.
   - A finding is never dropped because it's inconvenient.

4. **Behavior pass.** Read the changed logic for what it lets happen, not what it was meant to do:
   - **Server-side state.** Every value the client sends that decides state must be checked against what the server stored. That covers an outcome, a status, an id, and any text forwarded to another system. If an "it worked" decision rests on a counter of attempts, it must instead rest on a recorded success.
   - **Authorization on every mutating route.** This includes follow-up routes (finish, cancel, undo, retry), not only the one that creates the object. The object must belong to the caller and to every parent id the route or body names. An unguessable id is not authorization.
   - **Concurrency.**
     - Check-then-act sequences without a transaction, lock, or unique constraint: read-then-write, delete-then-insert, abandon-then-create.
     - Two tabs or a double click running the same request.
     - Client code that fires a dependent request without waiting for the one it depends on.
   - **Edge states.**
     - What happens after the object reaches a final state.
     - A late response after the user left.
     - A retry after a partial failure.

5. **Performance pass.** Read the changed logic for work that doesn't need doing:
   - Queries inside loops, or a loop that could be one query — N+1 is the most common one and the most expensive
   - Repeated computation that could be hoisted, memoized, or done once
   - Nested iteration over the same data where a map or set gives you a lookup
   - Loading whole collections to count, filter, or take the first item
   - Sync I/O, or sequential awaits over independent work
   - Unbounded growth: no pagination, no limit, no cleanup

   Judge against expected data size, not aesthetics. A nested loop over 5 config entries is fine; the same shape over user records is not.

6. **Standards pass.** Resolve every standard in scope — `standards` from CLAUDE.md § Pipeline settings, each `--standard`, and whatever the PRD header names — per § Standards. OWASP Top 10:2025 is below; any other becomes a checklist of the same shape before you start. Then walk every row of every standard against the diff — one at a time, no skipping.

   | # | Category |
   |---|---|
   | A01 | Broken Access Control |
   | A02 | Security Misconfiguration |
   | A03 | Software Supply Chain Failures |
   | A04 | Cryptographic Failures |
   | A05 | Injection |
   | A06 | Insecure Design |
   | A07 | Authentication Failures |
   | A08 | Software or Data Integrity Failures |
   | A09 | Security Logging and Alerting Failures |
   | A10 | Mishandling of Exceptional Conditions |

   Two questions per category: did this code introduce a weakness, and did it weaken an existing control — by widening a query, reusing a permissive handler, loosening validation, or logging a field that wasn't logged before. Logic flaws count: an auth check that runs after the mutation, a limit enforced client-side only, an ownership check on the wrong ID.

   For a privacy or sector standard the two questions become: does this diff collect, store, log, or transmit data the standard covers, and does it do so the way the standard requires — basis, minimization, retention, access, deletion, audit. A control that has no code-level reading in this diff is N/A with a reason, not skipped.

   When the diff sends anything to an LLM or AI agent, the OWASP Top 10 for LLM Applications (2025) is in scope even if nobody named it. For LLM01 (prompt injection), trace every string that reaches the prompt back to its origin. Text the client could have forged, such as questions, instructions, or "system" fields echoed back by the browser, is a finding unless the server generated it and verifies it now.

   New dependencies get checked for known CVEs and for whether they were needed at all.

7. **Translation pass.** If the project ships translation catalogs, every new user-visible string must exist in every locale the project ships, not only the one it was written in:
   - Search the catalogs for each new key.
   - Find new literals that bypass the translation helper in templates and in error messages returned to the UI.
   - A string present in one locale only is a finding.

8. **Fix and re-run.** Fix everything Critical and High, plus anything that fails a static check. After *any* code change, re-run lint, types, and tests — a performance rewrite is exactly the kind of change that breaks an edge case silently.

   Loop until clean. Leave Medium and Low as findings if the fix is a design decision rather than a correction. A Medium finding from the behavior pass that lets the client change server state (status, authorization, prompt content) is fixed, not left open.

9. **Open the PR** — only if everything above passes. See below. Clear the stage marker first: `rm -f .claude/.stage`. Under `autonomy: suggest` (CLAUDE.md § Execution boundaries) stop here and hand back the diff and the review instead.

## PR

Target `base_branch` from CLAUDE.md § Pipeline settings. Not set: `development` if it exists, else `develop`, else the repo's default branch.

If none of those exist, or there's no remote, or the repo was just initialized: stop and set it up with the user. Confirm the target branch name, whether a remote exists and where, and whether they want the branch created now. Don't invent a branching model or push to a remote they didn't name.

```bash
gh pr create --base {base_branch} --title "{what changed}" --body "{summary}"
```

Body: what changed and why, the validation results table, any Medium/Low findings left open, anything the reviewer should look at closely.

## Output

```markdown
## Review

| Check | Result | Details |
|---|---|---|
| Lint | ✅/❌ | {n errors or "passed"} |
| Types | ✅/❌ | {n errors or "passed"} |
| Tests | ✅/❌ | {n passed, m failed} |
| Independent read | ✅/⚠️ | {n findings, k confirmed, m false positives} |
| Behavior | ✅/⚠️ | {n issues, k fixed} |
| Performance | ✅/⚠️ | {n issues, k fixed} |
| {standard} | ✅/⚠️/❌ | {critical/high/medium/low counts — one row per standard in scope} |
| Translations | ✅/❌/N/A | {missing keys per locale} |
| PRD DoD | ✅/❌ | {n of m met} |

### Fixed
- `{file:line}` — {what was wrong, what changed}

### Findings
**[SEVERITY] {title}** · `{file:line}` · {standard and row, e.g. OWASP A01 / LGPD art. 46}
{What's wrong, what it risks, and the fix.}

### Verdict
{PASS / PASS WITH NOTES / FAIL}

### PR
{link, or why one wasn't opened}
```

Severity: **Critical** exploitable now · **High** real weakness · **Medium** defense-in-depth · **Low** best practice.

## Rules

- The review covers the whole branch, and an independent subagent reads it before you trust your own read.
- Every row of every standard in scope gets answered. "No" is an answer; silence isn't. An unresolved standard is a FAIL, not an omission.
- Any code change means re-running lint, types, and tests. No exceptions.
- Critical or High open means FAIL and no PR.
- Don't flag style. Flag correctness, security, and work that measurably doesn't need doing.
- Don't create branches or remotes on your own — that's a conversation with the user.