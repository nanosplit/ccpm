# Testing Discipline

Not every change needs the same level of testing rigor. This rule defines three tiers of testing discipline and how to choose the right one.

## The Non-Negotiable Baseline

Regardless of tier, you MUST always:
- **Run the existing test suite** (or relevant subset) after implementation
- **Verify the build succeeds**
- **Report test results honestly** — never skip reporting failures

The tiers only govern whether you write **new** tests and whether you write them **before** or **after** implementation.

## The Three Tiers

### Tier 1: Full TDD (Test-First)

Write a failing test before writing production code. Red → Green → Refactor.

**When to use:**
- New features that introduce user-facing behavior
- Bug fixes (write a test that reproduces the bug, then fix it)
- Changes to core business logic, data models, or algorithms
- Changes to public APIs, endpoints, or interfaces
- Security-sensitive code (authentication, authorization, input validation)
- Acceptance criteria describe specific behavioral outcomes that are testable

**The process:**
1. For each change unit: write a test that describes the expected behavior
2. Run the test — confirm it fails (red)
3. Write the minimum production code to make it pass (green)
4. Refactor if needed while keeping tests green
5. Commit the test and production code together

**Why test-first matters here:** These changes define new behavior. Writing the test first forces you to think about the contract before the implementation, catches misunderstandings early, and guarantees test coverage for the new behavior.

### Tier 2: Test-After (Implement, Then Cover)

Implement the change first, then write tests for the new or modified behavior.

**When to use:**
- Refactors of code that already has test coverage
- Enhancements to existing features where behavior changes are incremental
- Internal utilities, helpers, or abstractions
- Changes where the implementation approach is uncertain (exploratory work)
- Code where writing the test first would require knowing implementation details

**The process:**
1. Implement the changes
2. Run existing tests to catch regressions
3. Write new tests covering the changed behavior
4. Verify all tests pass
5. Commit

**Why test-after works here:** The behavior isn't new from scratch — it's an evolution. Existing tests provide a safety net, and writing tests after lets you test what you actually built rather than what you guessed you'd build.

### Tier 3: Verify-Only (Run Existing Suite)

Implement the change, then run existing tests and verify the build. No new tests needed.

**When to use:**
- Configuration file changes (build config, CI, linting rules)
- Documentation-only changes
- Dependency version updates without behavior changes
- Cosmetic changes (formatting, renaming, comment edits)
- Removing dead code
- Changes to files that have no testable behavior (templates, static assets)
- No test infrastructure exists in the project

**The process:**
1. Implement the changes
2. Run the existing test suite
3. Verify the build succeeds
4. Commit

**Why this is enough:** These changes don't introduce or modify behavior. The existing test suite catches regressions. Writing new tests for a config change or a comment edit adds overhead without value.

## How to Choose the Tier

Assess these signals after the researcher agent completes its analysis:

| Signal | Points toward |
|---|---|
| Acceptance criteria describe specific behaviors | Tier 1 |
| Task is a bug fix | Tier 1 |
| Changes touch public APIs or user-facing code | Tier 1 |
| Changes touch auth, security, or data integrity | Tier 1 |
| Existing test coverage for affected code is strong | Tier 2 |
| Task is a refactor or internal enhancement | Tier 2 |
| Implementation approach is exploratory | Tier 2 |
| Changes are config, docs, or cosmetic only | Tier 3 |
| No test infrastructure exists | Tier 3 |
| Changes remove code without adding new behavior | Tier 3 |

**When in doubt, go one tier up** — Tier 2 instead of Tier 3, or Tier 1 instead of Tier 2. The cost of slightly over-testing is lower than the cost of shipping a bug.

**Mixed changes:** If a task involves changes across multiple tiers (e.g., a new feature + config changes), use the highest applicable tier for the task as a whole.

## Declaring the Tier

After research, state the chosen tier explicitly before implementation begins:

```
Testing discipline: Tier {1/2/3} — {Full TDD / Test-after / Verify-only}
Rationale: {1-2 sentence justification based on the signals above}
```

This makes the decision visible and auditable. If the user disagrees with the tier, they can override it before implementation starts.

## Rationalization Prevention

These are NOT valid reasons to downgrade a tier:

| Rationalization | Reality |
|---|---|
| "Writing tests first is slower" | It's faster overall — you catch design issues before building the wrong thing |
| "I'll bump it up to Tier 1 if I find bugs later" | Later never comes. Assess now. |
| "This feature is too simple for TDD" | Simple features get Tier 1 because the tests are also simple and fast to write |
| "There aren't any tests in this project so Tier 3" | No existing tests is a reason to START writing them (Tier 1 or 2), not a reason to avoid it — unless the task itself is truly Tier 3 work |
| "The tests would just be trivial assertions" | Trivial tests are still tests. They catch regressions. Write them. |
