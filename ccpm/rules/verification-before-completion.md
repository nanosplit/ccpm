# Verification Before Completion

No task, issue, or implementation step may be marked as complete without fresh verification evidence. Confidence is not evidence. "It should work" is not verification.

## The Rule

**Before claiming any work is done, you MUST:**
1. Run actual verification commands (tests, builds, linters)
2. Read the real output — not a cached or remembered result
3. Confirm acceptance criteria are met against the actual state of the code

If you cannot produce fresh evidence that the work is done, the work is not done.

## Verification Gate

Every completion claim must pass through this gate:

### Step 1: Identify What to Verify
- Read the task's acceptance criteria
- List the specific, observable outcomes that prove completion
- If no acceptance criteria exist, verify that the described change is present and functional

### Step 2: Run Fresh Commands
- Run the project's test suite (or relevant subset)
- Run a build if applicable
- Run any task-specific verification commands
- **Do not skip this step.** Even if tests were run earlier, run them again — code may have changed since.

### Step 3: Read the Output
- Actually read the command output — do not assume success from exit codes alone
- Check for warnings, partial failures, or skipped tests
- Look for regressions in unrelated areas

### Step 4: Check Acceptance Criteria
- For each acceptance criterion, confirm it is satisfied with evidence:
  - "Tests pass" → cite the actual test output showing pass
  - "New endpoint exists" → show it responds correctly
  - "File is created" → confirm it exists with expected content
- If any criterion is not met, the task is NOT complete

### Step 5: Declare Result
- **Pass:** All criteria met with fresh evidence → mark complete
- **Partial:** Some criteria met → list what's done and what remains, do NOT mark complete
- **Fail:** Verification commands fail → fix the issues, re-verify

## False Confidence Patterns

These are NOT verification. If you catch yourself thinking any of these, stop and run actual commands:

| What you're tempted to say | Why it's not enough |
|---|---|
| "The code looks correct" | Reading code is not running code |
| "I just ran the tests" | If you made changes after, results are stale |
| "This is a simple change" | Simple changes break things too |
| "The tests passed earlier" | Earlier is not now |
| "I'm confident this works" | Confidence is not evidence |
| "It should work based on the logic" | Computers don't care about your logic |
| "I fixed the failing test" | Did you run it again to confirm? |
| "No tests exist, so nothing to verify" | Verify the change is present and the build succeeds |

## When This Rule Applies

- `/pm:work-on` — before summarizing completion
- `/pm:autopilot` — before marking each task closed
- `/pm:issue-close` — before closing on GitHub
- Any time you are about to say "done", "complete", or "finished"

## Failure Handling

If verification fails:
1. Do NOT mark the task as complete
2. Report what failed clearly: "❌ Verification failed: {what} — {output}"
3. Attempt to fix the issue
4. Re-run verification from Step 1
5. After 3 failed verification attempts, stop and report the situation to the user
