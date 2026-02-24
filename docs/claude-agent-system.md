# Claude Code Multi-Agent System

## Overview

This document describes the Claude Code multi-agent software development system deployed across the **Vaultody-com** GitHub organization.

The system uses Claude Code CLI (`claude -p`) running headless inside GitHub Actions to implement a structured, AI-driven development pipeline with strict separation of duties enforced at every stage.

All reusable workflows are centralized in the `Vaultody-com/.github` repository and called by individual repositories via `workflow_call`.

---

## Agent Roles and Trigger Labels

Each agent is triggered by applying a specific GitHub label to an issue. The **Team Leader** agent acts as the process controller, deciding when to transition between stages.

| Label | Agent | Workflow | Mode |
|---|---|---|---|
| `agent:developer` | Developer | `agent-developer-reusable.yml` | — |
| `agent:unit-test` | Unit Test | `agent-tests-reusable.yml` | `unit` |
| `agent:integration-test` | Integration Test | `agent-tests-reusable.yml` | `integration` |
| `agent:reviewer` | Reviewer | `agent-validation-reusable.yml` | `review` |
| `agent:perf` | Performance | `agent-validation-reusable.yml` | `perf` |
| `agent:qa` | QA Manual | `agent-validation-reusable.yml` | `qa` |
| `done` | — | — | Pipeline complete |

---

## Pipeline Flow

```
Issue Opened
    │
    ▼
[label: agent:developer]
    │  → agent-developer-reusable.yml
    │  → Creates feature/issue-<N> branch
    │  → Opens PR
    │
    ▼
[label: agent:unit-test]
    │  → agent-tests-reusable.yml (mode: unit)
    │  → Writes unit tests in test/unit/
    │  → Commits to feature branch
    │  → Comments coverage on PR
    │
    ▼
[label: agent:integration-test]
    │  → agent-tests-reusable.yml (mode: integration)
    │  → Writes integration tests in test/integration/
    │  → Commits to feature branch
    │  → Comments coverage on PR
    │
    ▼
[label: agent:reviewer]
    │  → agent-validation-reusable.yml (mode: review)
    │  → Posts structured review comment
    │  → Does NOT commit code
    │
    ▼
[label: agent:perf]
    │  → agent-validation-reusable.yml (mode: perf)
    │  → Runs performance checks if criteria defined in issue
    │  → Posts metrics comparison comment
    │  → Does NOT commit code
    │
    ▼
[label: agent:qa]
    │  → agent-validation-reusable.yml (mode: qa)
    │  → Generates QA checklist from Acceptance Criteria
    │  → Posts PASS/FAIL comment
    │  → Does NOT commit code
    │
    ▼
[label: done]
    │  → Pipeline complete
    │  → PR ready for human merge review
```

---

## Triggering the Developer Agent

To initiate AI-driven implementation of a GitHub issue:

1. Ensure the issue contains an **Acceptance Criteria** section in the body.
2. Apply the label `agent:developer` to the issue.
3. The calling repository must have a workflow that calls `agent-developer-reusable.yml`:

```yaml
on:
  issues:
    types: [labeled]

jobs:
  developer:
    if: github.event.label.name == 'agent:developer'
    uses: Vaultody-com/.github/.github/workflows/agent-developer-reusable.yml@main
    with:
      repo: ${{ github.repository }}
      issue_number: ${{ github.event.issue.number }}
      base_branch: main
    secrets: inherit
```

The Developer agent will:
- Read the issue title, body, and Acceptance Criteria
- Implement the feature in `src/**` only
- Create branch `feature/issue-<N>`
- Push the branch and open a PR titled `Implement issue #<N>`
- **Fail** if any file outside `src/**` is modified

---

## Triggering Unit and Integration Test Agents

After the Developer agent opens a PR, apply the appropriate label to the issue:

### Unit Tests

Apply label `agent:unit-test`. The calling workflow:

```yaml
on:
  issues:
    types: [labeled]

jobs:
  unit-tests:
    if: github.event.label.name == 'agent:unit-test'
    uses: Vaultody-com/.github/.github/workflows/agent-tests-reusable.yml@main
    with:
      repo: ${{ github.repository }}
      issue_number: ${{ github.event.issue.number }}
      pr_number: ${{ vars.CURRENT_PR_NUMBER }}
      mode: unit
    secrets: inherit
```

The Unit Test agent will:
- Write tests in `test/unit/**` only
- Map tests to each Acceptance Criterion
- Run tests and fail if any criterion is uncovered
- Commit to the feature branch
- Post a coverage summary comment on the PR

### Integration Tests

Apply label `agent:integration-test`. Same structure as above with `mode: integration`. The agent writes to `test/integration/**` only and verifies real endpoint behavior.

---

## Triggering Review, Performance, and QA Agents

### Code Review

Apply label `agent:reviewer`. The Reviewer agent:
- Posts a structured code review comment on the PR
- Flags test weakening, missing edge cases, and violations of Acceptance Criteria
- **Does NOT commit any code**

### Performance Validation

Apply label `agent:perf`. The Performance agent:
- Runs only if the issue defines performance criteria
- Posts a metrics comparison comment (before vs. after)
- **Does NOT modify production code**

### Manual QA

Apply label `agent:qa`. The QA agent:
- Generates a checklist from the issue's Acceptance Criteria
- Posts a PASS/FAIL determination with per-criterion results
- **Does NOT commit any code**

---

## Separation of Duties Policy

This is a **strictly enforced** policy. Each agent is restricted to its assigned scope. Any violation causes the workflow to fail immediately.

| Agent | Allowed Paths | Can Commit | Can Post Comments |
|---|---|---|---|
| Team Leader | None (label management only) | No | Yes |
| Developer | `src/**` | Yes | Yes |
| Unit Test | `test/unit/**` | Yes | Yes |
| Integration Test | `test/integration/**` | Yes | Yes |
| Reviewer | None | **No** | Yes |
| Performance | `test/performance/**` | No | Yes |
| QA Manual | None | **No** | Yes |

**Rules:**
- No agent may modify files outside its allowed paths.
- Reviewer and QA agents are strictly read-only. Any attempt to commit will cause the workflow to fail.
- Developer must never modify test directories.
- Test agents must never modify production code.
- No agent may override the separation of duties policy — this is enforced by the workflow, not the prompt alone.

---

## Required Branch Protection Rules

Each repository using this pipeline **must** configure branch protection on `main` with the following settings:

```
Branch: main

✅ Require a pull request before merging
  ✅ Require approvals: 1 (minimum)
  ✅ Dismiss stale pull request approvals when new commits are pushed

✅ Require status checks to pass before merging
  Required checks:
    - ci / build
    - ci / lint
    - ci / test

✅ Require branches to be up to date before merging

✅ Restrict who can push to matching branches
  (Only allow CI bot or designated human reviewers)

✅ Do not allow bypassing the above settings
```

To configure via GitHub UI:
1. Go to **Settings → Branches → Add rule**
2. Branch name pattern: `main`
3. Enable the settings above

---

## Required CODEOWNERS Setup

Each repository must have a `CODEOWNERS` file to ensure human review is required for sensitive paths:

```
# CODEOWNERS
# All production source code requires human review
/src/                    @Vaultody-com/engineering

# All test files require engineering team review
/test/                   @Vaultody-com/engineering

# Workflow files require platform team review
/.github/workflows/      @Vaultody-com/platform

# Documentation requires any team member review
/docs/                   @Vaultody-com/engineering
```

Place this file at:
- `.github/CODEOWNERS` (recommended), or
- `CODEOWNERS` in the repository root

This ensures that even if an agent opens a PR, a human reviewer from the appropriate team must approve it before it can be merged.

---

## Agent Prompt Files

All agent prompts are versioned in this repository under `agent-prompts/`:

| File | Role |
|---|---|
| `agent-prompts/team-leader.md` | Process controller — manages labels and state transitions |
| `agent-prompts/developer.md` | Implements features in `src/**` |
| `agent-prompts/unit-tests.md` | Writes unit tests in `test/unit/**` |
| `agent-prompts/integration-tests.md` | Writes integration tests in `test/integration/**` |
| `agent-prompts/performance.md` | Validates performance criteria |
| `agent-prompts/reviewer.md` | Performs structured code review (read-only) |
| `agent-prompts/qa-manual.md` | Generates QA checklist and PASS/FAIL verdict (read-only) |

---

## Reusable Workflow Files

All reusable workflows live in `.github/workflows/` in this repository:

| File | Purpose |
|---|---|
| `agent-developer-reusable.yml` | Developer agent — implements issue, opens PR |
| `agent-tests-reusable.yml` | Test agent — writes unit or integration tests |
| `agent-validation-reusable.yml` | Validation agent — review, performance, or QA |

---

## Security Notes

- All agent workflows require `ORG_ADMIN_TOKEN` (organization secret) to operate across repositories.
- Agent workflows run with the minimum permissions required for their role.
- No agent may read or write secrets or environment variables outside its workflow scope.
- The `agent-validation-reusable.yml` workflows run in read-only mode for reviewer and QA — they cannot push commits even if instructed to by the Claude model.

---

## Glossary

| Term | Definition |
|---|---|
| **Headless CLI** | `claude -p <prompt-file>` — runs Claude Code without interactive UI |
| **Workflow Call** | `on: workflow_call` — GitHub Actions reusable workflow trigger |
| **Acceptance Criteria** | Section in issue body defining what "done" means |
| **PASS/FAIL** | Required structured output from all agent prompts |
| **Separation of Duties** | Policy restricting each agent to its defined scope only |
