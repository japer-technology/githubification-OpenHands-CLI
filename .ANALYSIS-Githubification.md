# Githubification Analysis — OpenHands CLI

### How this repo could become a GitHub Action based mechanism

---

## What Is Githubification?

[Githubification](https://github.com/japer-technology/githubification) is the act of converting a repository into **GitHub-as-infrastructure**. Instead of cloning a repo and running the software elsewhere, the repo becomes something that runs on GitHub itself via GitHub Actions. There's no separate local runtime to install — GitHub is the runtime.

The concept rests on four GitHub primitives used in new roles:

| GitHub Primitive | Githubification Role |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent |
| **Git** | Storage and memory — sessions, conversations, and state are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread with the agent |
| **GitHub Secrets** | Credential store — LLM API keys and tokens |

---

## Why OpenHands CLI Is a Strong Candidate

OpenHands CLI is ranked **#11** in the [Githubification winners list](https://github.com/japer-technology/githubification/blob/main/.githubification/winners.md) with the note: *"CLI-based agents with stateless execution and structured output are naturally Githubifiable."*

The [lesson-from-OpenHands-CLI](https://github.com/japer-technology/githubification/blob/main/.githubification/lesson-from-OpenHands-CLI.md) analysis identifies this repo as a **Type 1 — AI Agent Repo** and makes a decisive observation:

> **When an AI agent already runs in headless mode on GitHub Actions — maintaining its own repository through PR reviews, code quality fixes, and issue triage — the infrastructure for Githubification already exists. The remaining step is redirecting that execution path from developer-facing automation to user-facing conversation.**

Every Githubification primitive is already actively in use in this repo — just not yet pointed at users:

| Primitive | Current Use (Developer-Facing) | Githubification Use (User-Facing) |
|---|---|---|
| **GitHub Actions** | Runs OpenHands in headless mode for PR reviews, code quality analysis, issue labeling | Would run OpenHands in headless mode for user conversations via Issues |
| **Git** | The codebase the agent reads and modifies through PRs | Would additionally store conversation sessions and agent memory |
| **GitHub Issues** | Traditional bug/feature tracking, auto-labeled by the agent | Would become the conversational surface for interacting with the agent |
| **GitHub Secrets** | `LLM_API_KEY`, `LLM_BASE_URL`, `ALLHANDS_BOT_GITHUB_PAT` | Same secrets, same purpose, now serving user-facing conversations |

---

## Current State — What Already Exists

### Agent Running on Actions

Three workflows already invoke the OpenHands agent on GitHub Actions:

1. **PR Review** (`pr-review-by-openhands.yml`) — On every non-draft PR, OpenHands reviews the diff using Claude and posts review comments
2. **Good First Issue Labeler** (`good-first-issue-labeler.yml`) — Weekly cron that runs OpenHands in headless mode with a prompt file to label issues
3. **Code Quality Analysis** (via `.github/prompts/`) — Prompt files that instruct the agent to analyze and auto-fix code quality issues

The labeler workflow demonstrates the complete execution model:

```yaml
- name: Run OpenHands labeler
  env:
    LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
    LLM_BASE_URL: https://llm-proxy.app.all-hands.dev
    LLM_MODEL: litellm_proxy/gpt-5.2
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    uv run openhands --headless --always-approve --override-with-envs \
      --file .github/prompts/good-first-issue-labeler.md
```

### Headless Mode

The `--headless` flag is the bridge between "local AI agent" and "GitHub-native AI agent":

```bash
openhands --headless --always-approve --override-with-envs -t "task description"
openhands --headless --always-approve --override-with-envs --file instructions.md
openhands --headless --json -t "task"  # Structured JSON output
```

### Prompt Files as Task Specifications

The `.github/prompts/` directory contains reusable Markdown task specifications:

| Prompt File | Purpose |
|---|---|
| `auto-fix-low-hanging.md` | Fix trivial code quality issues, create branches, open PRs |
| `code-quality-analysis.md` | Comprehensive code analysis: types, state, concerns, duplication |
| `good-first-issue-labeler.md` | Identify and label good-first-issue candidates |

### Agent Quality Gates

The `.openhands/hooks.json` enforces quality gates — the agent cannot complete a task until pre-commit checks pass:

```json
{
  "stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": ".openhands/hooks/on_stop.sh",
      "timeout": 300
    }]
  }]
}
```

### Comprehensive AGENTS.md

The 13KB `AGENTS.md` provides structured onboarding context covering project structure, build commands, coding style, testing guidelines, and TUI architecture — ready-made system prompt context for an issue-driven agent.

---

## The Githubification Strategy

### Recommended: Wrapping (Strategy 2)

Based on the [strategy selection guide](https://github.com/japer-technology/githubification/blob/main/.githubification/lesson-consolidation.md), this repo fits **Strategy 2 — Wrapping**:

- The agent **already exists** — it's the OpenHands CLI
- It **can run on GitHub Actions** — proven by existing workflows
- It does **not** have a multi-channel adapter architecture
- It is a **single agent** — not a multi-agent system

Wrapping means: a self-contained Githubification folder sits alongside the existing source and orchestrates how the agent runs on GitHub Actions for user-facing conversations. No modifications to the existing OpenHands CLI code.

### Alternative: Native via GMI

A complementary approach uses [GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence) (GMI) — a repository-local AI agent framework that uses Issues for conversation, Git for memory, and Actions for execution. GMI could be installed alongside OpenHands CLI to provide the issue-driven conversational layer, with OpenHands CLI available as a tool the GMI agent can invoke in headless mode for code tasks.

---

## How It Would Work

### Architecture

```
User opens/comments on Issue
    → GitHub Actions workflow triggers
    → Authorization check (repo collaborator?)
    → 🚀 reaction on issue (processing indicator)
    → Load conversation history from git (if continuing)
    → Run OpenHands CLI in headless mode with issue context
    → Post agent response as issue comment
    → Commit conversation state to git
    → 👍 reaction on issue (success indicator)
```

### The Githubification Workflow

A new workflow (`.github/workflows/githubification-agent.yml`) would trigger on issue events:

```yaml
name: OpenHands Issue Agent

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]

permissions:
  contents: write
  issues: write

concurrency:
  group: agent-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false

jobs:
  respond:
    if: |
      !github.event.issue.pull_request &&
      github.event.sender.type == 'User'
    runs-on: ubuntu-latest
    steps:
      - name: Check authorization
        # Only respond to repo collaborators
        run: |
          PERMISSION=$(gh api repos/${{ github.repository }}/collaborators/${{ github.actor }}/permission \
            --jq '.permission' 2>/dev/null || echo "none")
          if [[ "$PERMISSION" != "admin" && "$PERMISSION" != "write" && "$PERMISSION" != "maintain" ]]; then
            echo "Unauthorized user: ${{ github.actor }}"
            exit 0
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Add processing indicator
        run: |
          gh api repos/${{ github.repository }}/issues/${{ github.event.issue.number }}/reactions \
            -f content='rocket'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install uv and dependencies
        uses: astral-sh/setup-uv@v7
        with:
          enable-cache: false
      - run: uv sync

      - name: Extract issue content to file
        # Write issue/comment body to a file to avoid shell injection
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = context.payload.comment
              ? context.payload.comment.body
              : context.payload.issue.body;
            fs.writeFileSync('/tmp/issue-body.txt', body || '');

      - name: Run OpenHands agent
        env:
          LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
          LLM_MODEL: ${{ secrets.LLM_MODEL || 'gpt-5.4' }}
          LLM_BASE_URL: ${{ secrets.LLM_BASE_URL }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if ! uv run openhands --headless --always-approve --override-with-envs \
            --json --file /tmp/issue-body.txt > /tmp/agent-response.txt 2>/tmp/agent-error.txt; then
            echo "The agent encountered an error processing this request." > /tmp/agent-response.txt
            echo "" >> /tmp/agent-response.txt
            echo "<details><summary>Error details</summary>" >> /tmp/agent-response.txt
            echo "" >> /tmp/agent-response.txt
            echo '```' >> /tmp/agent-response.txt
            tail -20 /tmp/agent-error.txt >> /tmp/agent-response.txt
            echo '```' >> /tmp/agent-response.txt
            echo "</details>" >> /tmp/agent-response.txt
          fi

      - name: Post response to issue
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          gh issue comment "$ISSUE_NUMBER" --body-file /tmp/agent-response.txt

      - name: Commit conversation state
        env:
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add -A
          git diff --cached --quiet || git commit -m "state: update conversation for issue #${ISSUE_NUMBER}"
          for i in $(seq 1 10); do
            git push && break
            git pull --rebase -X theirs
            sleep $((i * 2))
          done
```

### The Prompt File

A new prompt file (`.github/prompts/respond-to-issue.md`) would define the agent's conversational behavior:

```markdown
You are an AI assistant for the OpenHands CLI project. A user has opened
or commented on a GitHub Issue. Your job is to:

1. Read the issue content and understand the user's question or request
2. Consult the codebase to provide accurate, context-aware responses
3. For bug reports: investigate the code, identify root causes, suggest fixes
4. For feature requests: assess feasibility, outline implementation approaches
5. For questions: explain relevant code, architecture, or usage patterns

Use the AGENTS.md file for project context. Follow the project's coding
conventions when suggesting code changes. Be helpful, accurate, and concise.
```

### Conversation State Management

Following the [universal pattern](https://github.com/japer-technology/githubification/blob/main/.githubification/lesson-consolidation.md) from consolidated Githubification lessons:

```
.githubification-state/
  issues/
    1.json          # Maps issue #1 → its session
    42.json         # Maps issue #42 → its session
  sessions/
    2026-03-18T..._abc123.jsonl    # Full conversation transcript
```

Each issue number is a stable conversation key. When a user comments on an issue weeks later, the agent loads the linked session and continues with full context — no database, no session cookies, just git.

---

## Key Design Decisions

### 1. Fail-Closed Security

The workflow only responds to repository **owners, members, and collaborators**. Unauthorized users cannot trigger the agent. This matches the GMI authorization model.

### 2. Concurrency Resilience

Per-issue concurrency groups prevent race conditions:

```yaml
concurrency:
  group: agent-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false
```

Combined with a git push retry loop (10 attempts with escalating backoff) to handle parallel state commits.

### 3. Label-Based Routing

Different issue labels could trigger different prompt files, extending the existing `.github/prompts/` pattern:

| Label | Prompt File | Agent Behavior |
|---|---|---|
| `question` | `respond-to-question.md` | Explain code, architecture, usage |
| `bug` | `investigate-bug.md` | Trace code paths, identify root cause |
| `feature-request` | `assess-feature.md` | Evaluate feasibility, outline approach |
| (default) | `respond-to-issue.md` | General-purpose conversation |

### 4. Self-Contained Folder

Following the Githubification containment principle, all agent state and configuration would live in one folder (`.githubification-state/`), with only the workflow file and issue templates in `.github/`.

### 5. Quality Gates Carry Over

The existing `.openhands/hooks.json` quality gate applies to all headless agent runs — the agent cannot complete tasks without passing pre-commit checks. This guarantee extends naturally to issue-driven conversations where the agent modifies code.

---

## What Makes This Different From Other Githubifications

### The Agent Is Already Running

Unlike most Githubification candidates where deploying the agent to Actions is the main deliverable, OpenHands CLI already has the agent executing on Actions, secrets configured, prompt files defining tasks, and quality gates enforcing standards. The remaining work is:

1. **A workflow trigger** on `issues` / `issue_comment` events (instead of `pull_request` or `schedule`)
2. **A prompt file** for conversational interaction (extending the existing `.github/prompts/` pattern)
3. **A state management folder** for multi-turn conversation (new, but following established patterns)

### Headless Mode Is Purpose-Built

The `--headless --always-approve --override-with-envs --json` invocation pattern is already proven in production on GitHub Actions. No adaptation layer is needed — the CLI was designed for this execution model.

### Self-Referential Development

OpenHands CLI is the only Githubification candidate where the agent in the repository is the same agent that develops the repository. Improvements to the CLI immediately improve the CLI's ability to serve users through Issues.

### MCP Extensibility

The CLI's MCP support allows the issue-driven agent to be extended with project-specific tools (web search, database access, API integration) via configuration — no code changes required.

---

## Comparison With GitHub Minimum Intelligence

| Dimension | GMI Approach | OpenHands CLI Native Approach |
|---|---|---|
| **Runtime** | TypeScript (`pi-coding-agent`) | Python (OpenHands SDK) |
| **Dependency** | 1 (`@anthropic/tokenizer`) | Multiple (openhands-sdk, textual, etc.) |
| **Installation** | Copy 1 workflow file, run once | Add workflow + prompt files |
| **Agent capabilities** | File read/write, grep, search, shell | Full coding agent: code, shell, browser, MCP |
| **State format** | JSONL sessions in git | JSONL sessions in git (to be implemented) |
| **Personality** | Hatching system (name, vibe, emoji) | AGENTS.md context (already exists) |
| **Self-install** | Yes (workflow_dispatch downloads agent folder) | No (agent is the existing CLI) |
| **Quality gates** | Lifecycle pipeline | `.openhands/hooks.json` pre-commit gate |
| **Multi-provider LLM** | 8 providers built-in | Via litellm (any provider) |
| **Maturity** | Production (fully Githubified) | Preparation phase |

Both approaches are valid and complementary. GMI provides a lightweight, single-dependency conversational agent. OpenHands CLI provides a full-featured coding agent with existing Actions infrastructure.

---

## Implementation Phases

### Phase 1 — Issue-Triggered Workflow

- Add `.github/workflows/githubification-agent.yml` with `issues` and `issue_comment` triggers
- Add `.github/prompts/respond-to-issue.md` prompt file
- Configure authorization checks (collaborator-only)
- Add reaction indicators (🚀 processing, 👍 success)

### Phase 2 — Conversation State

- Create `.githubification-state/` folder structure
- Implement issue-to-session mapping (`issues/N.json`)
- Implement session persistence (`sessions/<timestamp>.jsonl`)
- Add git commit/push with retry loop for state management

### Phase 3 — Label-Based Routing

- Create issue templates (`.github/ISSUE_TEMPLATE/`) for different interaction types
- Add label-to-prompt routing in the workflow
- Create specialized prompt files for bug investigation, feature assessment, and code questions

### Phase 4 — Enhanced Features

- MCP server configuration for project-specific tools
- Conversation condensation for long threads
- Integration with existing PR review workflow (agent can reference issue discussions in reviews)
- Public-facing documentation of the Githubified agent's capabilities

---

## Summary

This repository is closer to Githubification than any other candidate without an existing Githubification folder. The agent runs on Actions, the secrets are configured, the prompt files define reusable tasks, the quality gates enforce standards, and the headless mode produces clean stateless execution on CI runners. The gap is directional, not structural — redirecting an agent that already maintains its own repository toward serving users through Issues.

The four GitHub primitives (Actions as compute, Git as memory, Issues as UI, Secrets as credential store) are already mapped and operational. Githubification does not require building new infrastructure — it requires repurposing existing infrastructure for a new audience.
