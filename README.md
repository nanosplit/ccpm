# CCPM — Claude Code Project Management

A spec-driven project management system for Claude Code. Turns PRDs into epics, epics into tasks, and tasks into working code — all through markdown files, YAML frontmatter, and bash scripts. No database, no external tools beyond GitHub.

Forked from [automazeio/ccpm](https://github.com/automazeio/ccpm).

## How It Works

```
PRD → Epic → Tasks → Branch → Implement → Review → Merge
```

You write a product spec through guided brainstorming. CCPM converts it into a technical epic, decomposes it into tasks, syncs to GitHub Issues, and provides commands to work through each task with research, planning, implementation, and testing.

Everything lives in `.claude/` as markdown files. GitHub Issues are the shared source of truth for team visibility.

---

## Installation

### Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed and working
- Git repository with a GitHub remote
- [GitHub CLI (`gh`)](https://cli.github.com/) — CCPM will install this for you if needed

### Step 1: Add CCPM to your project

From your project root:

```bash
# Clone CCPM into a temporary directory
git clone https://github.com/nanosplit/ccpm.git /tmp/ccpm-install

# Copy the system into your project's .claude/ directory
# If you already have a .claude/ directory, this merges into it
cp -r /tmp/ccpm-install/ccpm/* .claude/

# Make scripts executable
chmod +x .claude/scripts/pm/*.sh

# Clean up
rm -rf /tmp/ccpm-install
```

This copies the commands, agents, rules, and scripts into your project's `.claude/` directory where Claude Code can find them.

### Step 2: Configure permissions (recommended)

CCPM ships with a `settings.local.json` that pre-approves common commands so you aren't prompted on every action. It's copied automatically in Step 1 and includes:

- `Edit` and `Write` — auto-allow all file edits (no per-session prompts)
- `Bash(git:*)`, `Bash(gh:*)` — git and GitHub CLI operations
- `Bash(bash .claude/scripts/pm/*)` — CCPM's shell scripts
- Common build tools (npm, bundle, make, etc.)

If you already had a `.claude/settings.local.json`, merge the entries from `.claude/settings.json.example` into yours. The key ones to add are `"Edit"` and `"Write"` in the allow array — without these, Claude will prompt for every file change.

Review the permissions and adjust to your stack — for example, add `"Bash(rails:*)"` for Rails projects.

### Step 3: Update .gitignore

Add CCPM's working directories to your `.gitignore`:

```bash
# CCPM working data (PRDs, epics, and task files are local working state)
.claude/prds/
.claude/epics/
```

The commands, agents, rules, and scripts should be committed so the whole team has access. The PRDs and epics are your local working state — sync them to GitHub Issues with `/pm:epic-sync` when ready.

### Step 4: Initialize

Open Claude Code in your project and run:

```
/pm:init
```

This will:
- Install GitHub CLI if needed (brew on macOS, apt with proper repo setup on Linux)
- Authenticate with GitHub if not already logged in
- Install the `gh-sub-issue` extension (pinned to v0.2.0) for parent-child issue relationships
- Create the `.claude/prds/` and `.claude/epics/` directories
- Verify your git remote is configured

### Step 5: Start using it

**Option A — Full workflow from scratch:**

```
/pm:start-feature user-authentication
```

This walks you through everything: PRD brainstorming, epic creation, task decomposition, branch creation, and kicks off work on the first task.

**Option B — Start from a GitHub issue:**

```
/pm:start-from-issue 42
```

When a detailed GitHub issue already exists with feature requirements, this fetches the issue, auto-generates a PRD from it, creates an epic, decomposes into tasks, and optionally syncs tasks as sub-issues under the original issue. You can provide a custom feature name: `/pm:start-from-issue 42 my-feature-name`.

**Option C — Step by step:**

```
/pm:prd-new user-authentication       # Brainstorm and write the PRD
/pm:prd-parse user-authentication     # Convert PRD to technical epic
/pm:epic-decompose user-authentication # Break epic into tasks
/pm:epic-sync user-authentication     # Push to GitHub Issues
/pm:work-on user-authentication/001   # Start working on the first task
```

---

## Everyday Workflow

Once CCPM is set up, there are two ways to work:

### Interactive — one task at a time

```bash
/pm:next                        # See what's ready
/pm:work-on feature-name/003   # Research, plan, implement, test
/pm:review                      # Code review before merge
/pm:issue-close 1234            # Close task, optionally merge branch
/pm:status                      # Check overall progress
```

### Unattended — let it run

```bash
/pm:autopilot feature-name     # Runs all remaining tasks, pushes, creates PR
```

Autopilot works through every task in dependency order — research, implement, test, advance — without stopping to ask. When it's done, it pushes the branch and creates a PR for you to review. Use this when you want to kick off work and walk away.

### Switching between features

CCPM uses git branches for isolation. To switch context:

```bash
# Commit current work (or let Claude commit for you)
git commit -am "WIP: progress on auth"

# Switch to another feature
git checkout feature/payment-flow
/pm:work-on payment-flow/002

# Switch back
git checkout feature/user-authentication
/pm:work-on user-authentication/004
```

---

## Directory Structure

After installation, your `.claude/` directory looks like this:

```
.claude/
├── agents/                # Agent definitions (researcher, code-analyzer, test-runner, spec-reviewer)
├── commands/
│   ├── pm/                # All /pm: command definitions
│   ├── context/           # Context management commands
│   └── testing/           # Test execution commands
├── context/               # Project-wide context files
├── epics/                 # Implementation plans and task files (gitignored)
│   └── feature-name/
│       ├── epic.md        # Technical plan
│       ├── 001.md         # Task files
│       └── updates/       # Progress tracking
├── hooks/                 # Claude Code hooks
├── prds/                  # Product requirement documents (gitignored)
├── rules/                 # Operational rules (testing discipline, verification, rationalization prevention, etc.)
├── scripts/
│   └── pm/                # Shell scripts (init, help, status, etc.)
├── ccpm.config            # GitHub repo detection config
├── settings.json.example  # Example permission settings
└── settings.local.json    # Active permission settings
```

---

## Command Reference

Type `/pm:help` for a quick summary.

### Workflow

| Command | Description |
|---------|-------------|
| `/pm:start-feature <name>` | End-to-end: PRD, epic, tasks, branch, and implement |
| `/pm:start-from-issue <num> [name]` | Start a feature from an existing GitHub issue |
| `/pm:work-on <task>` | Research, plan, implement, test, dual review, and verify a single task |
| `/pm:autopilot <epic>` | Unattended: run all tasks with full discipline, push branch, create PR |
| `/pm:review` | Dual review (spec compliance + code quality) and merge readiness |
| `/pm:next` | Show next priority task |
| `/pm:status` | Project dashboard |
| `/pm:standup` | Daily standup summary |
| `/pm:blocked` | Show blocked tasks |
| `/pm:in-progress` | List work in progress |

### PRD

| Command | Description |
|---------|-------------|
| `/pm:prd-new <name>` | Guided brainstorming to create a PRD |
| `/pm:prd-parse <name>` | Convert PRD to technical epic |
| `/pm:prd-list` | List all PRDs |
| `/pm:prd-edit <name>` | Edit existing PRD |
| `/pm:prd-status` | Show PRD implementation status |

### Epic

| Command | Description |
|---------|-------------|
| `/pm:epic-decompose <name>` | Break epic into task files |
| `/pm:epic-sync <name>` | Push epic and tasks to GitHub |
| `/pm:epic-oneshot <name>` | Decompose + sync in one step |
| `/pm:epic-list` | List all epics |
| `/pm:epic-show <name>` | Display epic and tasks |
| `/pm:epic-status <name>` | Show epic progress |
| `/pm:epic-close <name>` | Mark epic complete |
| `/pm:epic-edit <name>` | Edit epic details |
| `/pm:epic-refresh <name>` | Update progress from task status |
| `/pm:epic-start <name>` | Launch parallel agent execution |

### Issue

| Command | Description |
|---------|-------------|
| `/pm:issue-show <num>` | Display issue details |
| `/pm:issue-status <num>` | Check issue status |
| `/pm:issue-start <num>` | Start work (creates branch, single-threaded by default) |
| `/pm:issue-close <num>` | Close issue, optionally merge branch |
| `/pm:issue-sync <num>` | Push updates to GitHub |
| `/pm:issue-reopen <num>` | Reopen closed issue |
| `/pm:issue-edit <num>` | Edit issue details |
| `/pm:issue-analyze <num>` | Analyze for parallel work streams |

### Other

| Command | Description |
|---------|-------------|
| `/pm:sync` | Bidirectional sync with GitHub |
| `/pm:import <issue>` | Import existing GitHub issues |
| `/pm:validate` | Check system integrity |
| `/pm:clean` | Archive completed work |
| `/pm:search <query>` | Search across all content |
| `/pm:init` | Install dependencies and configure GitHub |

---

## Branching Strategy

CCPM uses branches for feature isolation:

- `feature/<name>` — Created by `/pm:start-feature`, holds all work for a feature
- `task/<issue>-<slug>` — Created by `/pm:issue-start` or `/pm:work-on`, one branch per task
- `epic/<name>` — Created by `/pm:epic-start` for parallel agent execution

When closing an issue with `/pm:issue-close`, you're offered the option to merge the task branch back to main.

Parallel execution (`/pm:issue-start <num> --parallel`) is available for splitting a single issue across multiple agents, but the default single-threaded mode is simpler and works well for most tasks.

## Agents

Four specialized agents handle different parts of the workflow:

- **researcher** — Analyzes the codebase for a task and produces a file-by-file change plan with dependency ordering, risk assessment, and a testing tier recommendation
- **spec-reviewer** — Independently verifies that an implementation satisfies a task's acceptance criteria. Does not trust the implementer's report — reads the actual code against the spec
- **code-analyzer** — Reviews code changes for bugs, security issues, and regressions
- **test-runner** — Executes tests and provides structured analysis of results

Agents are launched automatically by workflow commands. `/pm:work-on` and `/pm:autopilot` run the researcher before implementation, then dispatch both the spec-reviewer and code-analyzer as a **dual review** after implementation. `/pm:review` runs the same dual review on any branch.

## Engineering Discipline

CCPM enforces several quality gates throughout the workflow, inspired by the [Superpowers](https://github.com/obra/superpowers) framework:

### Tiered Testing Discipline

Not every change needs the same testing rigor. The researcher agent assesses the task and recommends a tier:

- **Tier 1: Full TDD** — Write a failing test before production code. For new features, bug fixes, core logic, and security-sensitive changes.
- **Tier 2: Test-after** — Implement first, then write tests. For refactors with existing coverage, internal utilities, and exploratory work.
- **Tier 3: Verify-only** — Run the existing test suite. For config, docs, and cosmetic changes.

The existing test suite always runs regardless of tier. When in doubt, the system defaults one tier up.

### Verification Before Completion

No task can be marked complete without fresh evidence. Agents must run actual commands (tests, builds), read the real output, and confirm each acceptance criterion is satisfied. "It should work" is not verification.

### Dual Review

After implementation, two independent reviews run in parallel:

1. **Spec compliance** (spec-reviewer agent) — Checks every acceptance criterion against the actual code
2. **Code quality** (code-analyzer agent) — Hunts for bugs, security issues, and pattern violations

Critical issues must be fixed before proceeding. In autopilot mode, important/minor issues are logged for the PR.

### Rationalization Prevention

Agents will try to skip steps. The system includes explicit tables of common rationalizations and counter-arguments, embedded at the decision points where shortcuts are most likely — before research, before testing, and in unattended mode.

## Design Decisions

- **No database** — All data is markdown + YAML frontmatter in `.claude/`
- **GitHub Issues as shared state** — Local files for speed, GitHub for team visibility
- **Branches over worktrees** — Simpler mental model, no duplicate dependencies
- **Explicit sync** — Nothing pushes to GitHub without you asking
- **Spec-driven** — Every task traces back to a PRD

## Technical Notes

- Task files start as `001.md`, `002.md` during decomposition. After GitHub sync, they're renamed to `{issue-id}.md` for easy navigation.
- The `gh-sub-issue` extension is pinned to v0.2.0 for reproducibility.
- All timestamps use ISO 8601 UTC format via `date -u`.
- Commands use YAML frontmatter `allowed-tools` to declare their tool permissions.
- Feature names must be kebab-case (lowercase letters, numbers, hyphens).

## Updating

To pull in updates from this repo:

```bash
git clone https://github.com/nanosplit/ccpm.git /tmp/ccpm-update
cp -r /tmp/ccpm-update/ccpm/* .claude/
chmod +x .claude/scripts/pm/*.sh
rm -rf /tmp/ccpm-update
```

This overwrites commands, agents, rules, and scripts. The following are **not affected**:

| Safe (not overwritten) | Overwritten on update |
|------------------------|-----------------------|
| `.claude/prds/` | `.claude/commands/` |
| `.claude/epics/` | `.claude/agents/` |
| `.claude/context/` | `.claude/rules/` |
| | `.claude/scripts/` |

**After updating:** restart Claude Code to pick up new or changed commands. Settings files (`settings.local.json`) are also overwritten — if you've customized permissions, back up your file first or re-merge after.

To update a single command or agent without a full update:

```bash
# Example: update just the autopilot command
git clone https://github.com/nanosplit/ccpm.git /tmp/ccpm-update
cp /tmp/ccpm-update/ccpm/commands/pm/autopilot.md .claude/commands/pm/autopilot.md
rm -rf /tmp/ccpm-update
```

## License

MIT — see [LICENSE](LICENSE).

Based on [CCPM by Automaze](https://github.com/automazeio/ccpm).
