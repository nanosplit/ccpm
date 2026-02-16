# Rationalization Prevention

Agents will try to skip steps. They will generate plausible-sounding reasons to take shortcuts. This rule exists to catch those rationalizations before they lead to sloppy work.

**The core principle:** If a step is defined in a workflow, it gets executed. The quality of the excuse for skipping it is irrelevant.

## How to Use This Rule

When you encounter a step in a workflow and feel the urge to skip it, check the tables below. If your reasoning matches any listed rationalization — or sounds like a variation — stop and do the step anyway.

## Research Phase Shortcuts

These rationalizations lead to incomplete understanding and bad implementation plans.

| Rationalization | Why it's wrong | What to do instead |
|---|---|---|
| "This task is simple enough to implement directly" | Simple-looking tasks often have hidden dependencies and edge cases | Run the researcher agent. Let the analysis confirm it's simple. |
| "I already know this codebase well enough" | Your knowledge may be stale or incomplete. New patterns may have been introduced. | Run the researcher agent. It takes minutes and catches what you miss. |
| "The task description is clear, no research needed" | Clear requirements ≠ clear implementation. The codebase may not support the obvious approach. | Research the codebase. The task says *what*, research tells you *how*. |
| "I'll just look at the relevant files as I go" | Ad-hoc exploration misses cross-file dependencies and existing patterns | A structured research pass finds things that browsing won't. |
| "The researcher will just tell me what I already know" | Then it costs you nothing. But if it finds something you didn't know, it saves hours. | Run it. The downside of skipping is worse than the cost of running it. |

## Testing Shortcuts

These rationalizations lead to broken code shipped with false confidence.

| Rationalization | Why it's wrong | What to do instead |
|---|---|---|
| "This is a config/styling change, no tests needed" | Config changes can break builds, imports, and runtime behavior | Run the test suite. Config bugs are the hardest to trace. |
| "No test infrastructure exists" | There may be tests you haven't found. Even without tests, you can verify the build. | Search for test files. Run the build. Verify *something*. |
| "The tests are unrelated to my changes" | Regressions happen in unrelated areas. That's the whole point of running the full suite. | Run them anyway. Unrelated failures are the ones that bite hardest. |
| "I'll add tests later" | Later never comes. The context you have now is the best context for writing tests. | Write or run tests now, as part of this task. |
| "Tests passed earlier, they'll pass again" | You've made changes since then. Earlier results are stale. | Run them again. Fresh results only. |
| "Running tests will take too long" | Shipping broken code takes longer to fix than running tests takes to run. | Run them. If they're slow, run the relevant subset. |

## Implementation Shortcuts

These rationalizations lead to drift from the plan and avoidable bugs.

| Rationalization | Why it's wrong | What to do instead |
|---|---|---|
| "The plan is overkill, I'll make the obvious change" | The plan exists because research found non-obvious concerns. Ignoring it ignores that work. | Follow the plan. If it's truly wrong, flag it — don't silently deviate. |
| "I don't need to follow the dependency order" | Dependency order prevents cascading errors and merge conflicts | Follow the order. It exists for a reason. |
| "This is a trivial fix, no need for atomic commits" | Atomic commits let you bisect and revert. "Trivial" fixes are the ones that aren't. | Commit at each logical unit. It costs nothing and saves everything. |
| "I'll clean this up in a follow-up" | Follow-ups get deprioritized. Ship clean or flag the tech debt explicitly. | Clean it up now, or create a tracked task for the cleanup. |
| "The existing code doesn't follow this pattern either" | Don't make entropy worse. Follow the defined pattern even if legacy code doesn't. | Follow the defined conventions. Note the inconsistency if you want. |

## Review Shortcuts

These rationalizations lead to bugs in production.

| Rationalization | Why it's wrong | What to do instead |
|---|---|---|
| "This is a small change, review isn't needed" | Small changes cause large outages. A one-character typo can take down a service. | Review it. Small changes are fast to review. |
| "I wrote it carefully, it's fine" | Authors are the worst reviewers of their own code. Familiarity breeds blindness. | Get a review. Fresh eyes catch what yours skip. |
| "The tests pass, so the code must be correct" | Tests verify behavior they test for. They don't verify behavior they don't. | Review for logic, security, and edge cases that tests don't cover. |
| "We're in a hurry" | Rushed code creates more work than it saves. A bug in production costs more than a review. | Take the time. It's faster than the rollback. |

## Unattended Mode (Autopilot) Shortcuts

These rationalizations are especially dangerous because no human is watching.

| Rationalization | Why it's wrong | What to do instead |
|---|---|---|
| "This task is too complex for unattended mode, I'll skip it" | Every task was decomposed to be achievable. If it's genuinely blocked, mark it and move on — don't skip silently. | Attempt it. If it fails, leave it as `in-progress` with a clear note. |
| "The previous task already handled this" | Tasks may overlap but each has distinct acceptance criteria. Don't assume coverage. | Check the acceptance criteria. Implement what's specified. |
| "I'll just mark this as done and move on" | This is the exact failure mode this rule exists to prevent. | Never mark done without verification evidence. Follow the verification gate. |
| "This doesn't need testing in autopilot mode" | Autopilot mode needs *more* testing discipline, not less, because no one is watching. | Run tests. Always. No exceptions. |
| "I can combine these tasks to be more efficient" | Tasks are decomposed for a reason — isolation, atomic commits, clear verification. | One task at a time. Follow the defined sequence. |

## The Meta-Rationalization

The most dangerous rationalization is: **"This rule is overkill for what I'm doing."**

If you're thinking that, you are exactly the kind of situation this rule was written for. Do the step.
