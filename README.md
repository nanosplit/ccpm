# CCPM — Claude Code Project Management

A spec-driven project management system for Claude Code. Turns PRDs into epics, epics into tasks, and tasks into working code — all through markdown files, YAML frontmatter, and bash scripts. No database, no external tools beyond GitHub.

Forked from [automazeio/ccpm](https://github.com/automazeio/ccpm).

## How It Works

```
PRD → Epic → Tasks → Branch → Implement → Review → Merge
```

You write a product spec through guided brainstorming. CCPM converts it into a technical epic, decomposes it into tasks, syncs to GitHub Issues, and provides commands to work through each task with research, planning, implementation, and testing.

Everything lives in `.claude/` as markdown files. GitHub Issues are the shared source of truth for team visibility.

## Quick Start

1. **Copy the `ccpm/` directory into your project's `.claude/` folder** (or clone and copy manually).

2. **Initialize:**
   ```
   /pm:init
   ```
   Installs GitHub CLI if needed, authenticates, sets up the directory structure.

3. **Start a feature:**
   ```
   /pm:start-feature your-feature-name
   ```
   This walks you through the full flow: PRD brainstorming, epic creation, task decomposition, branch creation, and kicks off implementation of the first task.

Or do it step by step:

```bash
/pm:prd-new your-feature       # Brainstorm and write the PRD
/pm:prd-parse your-feature     # Convert PRD to technical epic
/pm:epic-decompose your-feature # Break epic into tasks
/pm:epic-sync your-feature     # Push to GitHub Issues
/pm:work-on your-feature/001   # Start working on the first task
```

## Directory Structure

```
.claude/
├── prds/                  # Product requirement documents
├── epics/                 # Implementation plans and tasks
│   └── feature-name/
│       ├── epic.md        # Technical plan
│       ├── 001.md         # Task files
│       ├── 002.md
│       └── updates/       # Progress tracking
├── agents/                # Agent definitions
├── commands/pm/           # All /pm: command definitions
├── rules/                 # Reusable rules (datetime, git, frontmatter)
└── scripts/pm/            # Shell scripts (init, help)
```

## Command Reference

Type `/pm:help` for a quick summary.

### Workflow

| Command | Description |
|---------|-------------|
| `/pm:start-feature <name>` | End-to-end: PRD, epic, tasks, branch, and implement |
| `/pm:work-on <task>` | Research, plan, implement, and test a single task |
| `/pm:review` | Review current branch for bugs and merge readiness |
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

## Branching Strategy

CCPM uses branches for feature isolation:

- `feature/<name>` — Created by `/pm:start-feature`, holds all work for a feature
- `task/<issue>-<slug>` — Created by `/pm:issue-start` or `/pm:work-on`, one branch per task
- `epic/<name>` — Created by `/pm:epic-start` for parallel agent execution

When closing an issue with `/pm:issue-close`, you're offered the option to merge the task branch back to main.

Parallel execution (`/pm:issue-start <num> --parallel`) is available for splitting a single issue across multiple agents, but the default single-threaded mode is simpler and works well for most tasks.

## Agents

Three specialized agents handle different parts of the workflow:

- **researcher** — Analyzes the codebase for a task and produces a file-by-file change plan with dependency ordering and risk assessment
- **code-analyzer** — Reviews code changes for bugs, security issues, and regressions
- **test-runner** — Executes tests and provides structured analysis of results

Agents are launched automatically by workflow commands (e.g., `/pm:work-on` runs the researcher, `/pm:review` runs the code-analyzer).

## Design Decisions

- **No database** — All data is markdown + YAML frontmatter in `.claude/`
- **GitHub Issues as shared state** — Local files for speed, GitHub for team visibility
- **Branches over worktrees** — Simpler mental model, no duplicate dependencies
- **Explicit sync** — Nothing pushes to GitHub without you asking
- **Spec-driven** — Every task traces back to a PRD. No vibe coding.

## Technical Notes

- Task files start as `001.md`, `002.md` during decomposition. After GitHub sync, they're renamed to `{issue-id}.md` for easy navigation.
- The `gh-sub-issue` extension is pinned to v0.2.0 for reproducibility.
- All timestamps use ISO 8601 UTC format via `date -u`.
- Commands use YAML frontmatter `allowed-tools` to declare their tool permissions.

## License

MIT — see [LICENSE](LICENSE).

Based on [CCPM by Automaze](https://github.com/automazeio/ccpm).
