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

3. **Independent read.** You wrote this code, so you share its blind spots. Dispatch a fresh subagent that has never seen your reasoning. Give it only the full branch diff, the PRD, and this brief: "Find bugs, security holes, business rules the server does not enforce, places where the code departs from how the rest of the module is built, and anything that changes for users who never use this feature. For each finding give the file and line, a concrete failure scenario, and a severity." Do not give it your conclusions.

   Verify every finding against the code before acting on it:
   - Confirmed findings join this review.
   - A false positive is recorded with the `file:line` that disproves it.
   - A finding is never dropped because it's inconvenient.

4. **Authorization matrix.** Before any other judgment on access, build a table with one row per route, endpoint, job, or handler the branch adds or changes that reads private data or writes anything. Include follow-up routes (finish, cancel, undo, retry, callbacks) and routes whose code did not change but whose behavior did.

   | Route | Who may call it | Tenant scope | Ownership (caller owns the object) | Parent ids (object belongs to every id in the route/body) | Business-state rules (deadline, status, already done, delivered, feature flag) | Checked before the write? | `file:line` |
   |---|---|---|---|---|---|---|---|

   - **Filling a cell.** A cell is filled only with the `file:line` of the check that enforces it, or "N/A" with a reason. "Unguessable id" or "the UI never sends it" is not an answer.
   - **Parity.** Find the existing screen or handler that already guards the same resource, for example the page where the user edits that record or the endpoint that saves it. Every rule it enforces must appear in the new route's row, or the row must say why it doesn't apply. A rule the sibling enforces and the new route skips is a finding.
   - **Severity.** An empty cell, or a check that runs after the write, is at least High. It blocks the PR until it is fixed or the user explicitly accepts it.
   - **Output.** The matrix goes into the review output as is.

5. **Conventions and reuse pass.** A reviewer on the team judges the diff against how the module is already built, not against what works. Build the module's rules first:
   - Read the 3–5 most recently changed files of the same kind in the same module (`git log --format= --name-only -- {module path} | head`), plus any architecture doc, ADR, or CLAUDE.md section. Older code in the module is not the reference when newer code does it differently.
   - Write down, each with `file:line`: where database access lives; where business rules live (UseCase, domain, controller); how dependencies arrive (container, constructor injection, `new`); how errors travel to the HTTP layer; how routes are documented; file size limits.

   Then walk every new class, function, query, and file in the diff against those rules:
   - **Layer.** Database access outside the layer the module uses for it, business rules in a controller or a service the module reserves for IO, or a class in a folder whose siblings do something else, is a finding.
   - **Wiring.** Collaborators created with `new` or nullable constructor defaults, where the module injects them, are a finding. So is production code that branches only for tests, such as an override or a stub switch.
   - **Reuse.** For every new query, helper, HTTP call, toast, or dialog, search the codebase for an existing repository method or helper that already does it. Writing a second one is a finding that names the one to use.
   - **Duplication and size.** Two new classes that differ only in a constant, or a file over the project's size limit, are findings.
   - **Query keys.** For every query against an existing table, find how existing code reads that same table and use the same keys. Filtering by a coarser key than the rest of the codebase, for example a parent id where the code uses a configuration id, is a correctness bug, not style.

   Severity: Medium by default, High when it is the query-key case or when the module's rule is written down and the diff breaks it. These are not style comments. They are what a team reviewer rejects the PR for.

6. **Blast radius pass.** List what changes for users who never touch the feature:
   - **Shared code and templates.** Every changed line in a file the feature does not own, such as a shared template, base controller, or global script. For each one, state what users with the feature flag off now see or get. Any difference is a finding unless the PRD asked for it.
   - **Moved code.** Code moved out of an existing handler into a new class must behave the same. Check type coercion (`strict_types`, typed DTOs fed by untyped model attributes), redirects, messages, and order of checks. Each branch of the moved logic needs a test that pins the old behavior.
   - **Dependency bumps.** For every upgraded package, read its changelog or diff between the two versions and list the components it changes. Search the codebase for every consumer of those components. A default behavior change in a shared component is High until each consumer is checked or the change is made opt-in.
   - **Lockfiles.** The lockfile diff must contain only the packages the change intended. A changed source protocol or URL, such as `git+ssh` becoming `git+https`, or noise from a different package manager version, is a finding. Regenerate the lockfile with the version the project pins.
   - **Deploy order.** For code that depends on a new migration, row, flag, or env var, say what happens if the code ships first. A failure in a screen unrelated to the feature is High.
   - **Migrations.** Migrations that ship together end in the final schema in as few steps as possible. Create-then-alter in the same release, a status modeled as NULL where the codebase uses explicit values, or a WHERE clause the new indexes do not cover, is a finding.

7. **Behavior pass.** Read the changed logic for what it lets happen, not what it was meant to do:
   - **Server-side state.** Every value the client sends that decides state must be checked against what the server stored. That covers an outcome, a status, an id, and any text forwarded to another system. If an "it worked" decision rests on a counter of attempts, it must instead rest on a recorded success.
   - **Authorization on every mutating route.** This includes follow-up routes (finish, cancel, undo, retry), not only the one that creates the object. The object must belong to the caller and to every parent id the route or body names. An unguessable id is not authorization.
   - **Concurrency.**
     - Check-then-act sequences without a lock or unique constraint: read-then-write, delete-then-insert, abandon-then-create. A transaction alone does not stop two requests from both passing the check.
     - Two tabs or a double click running the same request.
     - Client code that fires a dependent request without waiting for the one it depends on.
   - **Edge states.**
     - What happens after the object reaches a final state.
     - A late response after the user left.
     - A retry after a partial failure.
     - Undo, restore, or rollback actions that overwrite what the user changed after the action they undo.
     - Text from an AI or another system shown or stored as is. Limits it was asked to respect, such as length and format, are enforced on the server, not only reported.

8. **Performance pass.** Read the changed logic for work that doesn't need doing:
   - Queries inside loops, or a loop that could be one query — N+1 is the most common one and the most expensive
   - Repeated computation that could be hoisted, memoized, or done once
   - Nested iteration over the same data where a map or set gives you a lookup
   - Loading whole collections to count, filter, or take the first item
   - Sync I/O, or sequential awaits over independent work
   - Unbounded growth: no pagination, no limit, no cleanup

   Judge against expected data size, not aesthetics. A nested loop over 5 config entries is fine; the same shape over user records is not.

9. **Standards pass.** Resolve every standard in scope — `standards` from CLAUDE.md § Pipeline settings, each `--standard`, and whatever the PRD header names — per § Standards. OWASP Top 10:2025 is below; any other becomes a checklist of the same shape before you start. Then walk every row of every standard against the diff — one at a time, no skipping.

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

10. **Translation pass.** If the project ships translation catalogs, every new user-visible string must exist in every locale the project ships, not only the one it was written in:
   - Search the catalogs for each new key.
   - Find new literals that bypass the translation helper in templates and in error messages returned to the UI.
   - A string present in one locale only is a finding.

11. **Fix and re-run.** Fix everything Critical and High, plus anything that fails a static check. After *any* code change, re-run lint, types, and tests — a performance rewrite is exactly the kind of change that breaks an edge case silently.

   Loop until clean. Leave Medium and Low as findings if the fix is a design decision rather than a correction. A Medium finding from the behavior pass that lets the client change server state (status, authorization, prompt content) is fixed, not left open.

12. **Open the PR** — only if everything above passes. See below. Clear the stage marker first: `rm -f .claude/.stage`. Under `autonomy: suggest` (CLAUDE.md § Execution boundaries) stop here and hand back the diff and the review instead.

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
| Authorization matrix | ✅/❌ | {n routes, k rows complete, m gaps} |
| Conventions and reuse | ✅/⚠️ | {n deviations, k fixed} |
| Blast radius | ✅/⚠️ | {n changes outside the feature, k justified} |
| Behavior | ✅/⚠️ | {n issues, k fixed} |
| Performance | ✅/⚠️ | {n issues, k fixed} |
| {standard} | ✅/⚠️/❌ | {critical/high/medium/low counts — one row per standard in scope} |
| Translations | ✅/❌/N/A | {missing keys per locale} |
| PRD DoD | ✅/❌ | {n of m met} |

### Authorization matrix
{the table from step 4}

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
- Every route that reads private data or writes gets a complete row in the authorization matrix, with parity against the existing screen that guards the same resource. A gap is High.
- The diff follows the module's newest code: same layers, same wiring, existing helpers reused, same keys on existing tables. A deviation is a finding, not a style note.
- Anything that changes for users without the feature, including dependency bumps and lockfile noise, is listed and justified.
- Every row of every standard in scope gets answered. "No" is an answer; silence isn't. An unresolved standard is a FAIL, not an omission.
- Any code change means re-running lint, types, and tests. No exceptions.
- Critical or High open means FAIL and no PR.
- Don't flag style. Flag correctness, security, and work that measurably doesn't need doing.
- Don't create branches or remotes on your own — that's a conversation with the user.