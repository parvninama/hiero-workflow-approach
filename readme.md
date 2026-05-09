# Hiero Workflow Automation – Design Approach

> This document was prepared as part of my application for the [LFX Mentorship Program](https://lfx.linuxfoundation.org/tools/mentorship/) under the [Hiero](https://github.com/hiero-ledger) project (LFDT). It outlines my proposed architecture and intended approach for the mentorship [project](https://github.com/LF-Decentralized-Trust-Mentorships/mentorship-program/issues/73).

## 1. Overview

This document proposes a scalable, secure, and reusable architecture for maintainer workflow automation across Hiero repositories.

The current system demonstrates strong intent but suffers from fragmentation and duplication across repositories. The goal is to evolve this into a consistent, configurable, and extensible system that improves contributor onboarding, progression, and overall maintainer efficiency.

---

## 2. Current System Analysis

### V0 – Python SDK Workflows
- Event-specific workflows (PR, issues, comments)
- Logic duplicated across multiple workflow files
- No shared abstraction layer
- Difficult to maintain and extend

### V1 – C++ SDK Workflows
- Improved modular structure and separation of concerns
- Uses `actions/github-script` to run JS files inline via `require()`
- However:
  - Scripts live inside the same repository under `.github/scripts/`
  - Tightly coupled to repository-specific constants and config
  - Inconsistent label formats (`skill: beginner` vs `beginner`)
  - Limited portability — every new repo must copy the entire scripts folder

---

### Key Problems Identified

| Problem | Impact |
|---|---|
| Logic duplication across workflows | High maintenance cost, inconsistent fixes |
| Inconsistent label formats | Progression logic breaks silently |
| Tight coupling to repo config | Cannot reuse across repositories |
| No centralized decision-making | Behavior diverges across repos over time |
| Poor portability | Each new repo requires manual adaptation |
| Inline JS via `github-script` | Cannot fully unit test GitHub environment behavior |

---

## 3. V0 → V1 → V2 Comparison

> **What changed, why it changed, and what V2 fixes.**

| Aspect | V0 (Python SDK) | V1 (C++ SDK) | V2 (Proposed) |
|---|---|---|---|
| Structure | Event-specific workflows | Modular structure | Layered architecture (trigger + engine) |
| Logic reuse | ❌ Duplicated across files | ⚠️ Improved but still scattered | ✅ Centralized rule engine |
| Config handling | ❌ Hardcoded / implicit | ⚠️ Repo-specific configs | ✅ Standardized config system |
| Portability | ❌ Not reusable | ⚠️ Limited portability | ✅ Cross-repo compatible |
| Label handling | ❌ Inconsistent | ❌ Still inconsistent | ✅ Normalized labels |
| Scalability | ❌ Poor | ⚠️ Moderate | ✅ Designed for multi-repo scale |
| Decision system | ❌ None | ❌ None | ✅ Central orchestration layer |
| Extensibility | ❌ Hard | ⚠️ Moderate | ✅ Feature flags + versioning |

---

## 4. Proposed Architecture

### Key Design Principle

> Separate **"when something runs"** from **"what should happen"**

| Layer | Role |
|---|---|
| GitHub Actions | Trigger events only — no business logic |
| JavaScript Actions | Execute operations — reusable, versioned |
| GitHub App | Decide behavior — centralized logic and config |

---

### 4.1 Trigger Layer (GitHub Actions)

**Responsibility: Event Handling Only**

- Listens to: `pull_request`, `issues`, `issue_comment`
- Passes structured event data to the execution layer

✅ No business logic in workflows  
✅ Lightweight and consistent across repositories

---

### 4.2 Execution Layer (Standalone JavaScript Actions)

**Responsibility: Standardized, Reusable Execution Units**

This is a deliberate architectural choice over the V1 approach of inline JS via `actions/github-script`.

**Why standalone JS Actions over composite actions with inline JS:**

The V1 C++ setup uses `github-script` to `require()` scripts from `.github/scripts/`. This works for a single repo but has a fundamental limitation — the script receives `github` and `context` objects injected by the GitHub Actions runner at runtime. You cannot fully replicate that environment locally. Finding a bug means pushing to a branch, triggering a real event, waiting for the runner, reading logs, and repeating.

With standalone JS Actions, inputs are explicitly defined in `action.yml` as typed, named parameters:

```yaml
inputs:
  repo:
    description: 'Repository name'
    required: true
  owner:
    description: 'Repository owner'
    required: true
  pr-number:
    description: 'Pull request number'
    required: true
  username:
    description: 'PR author login'
    required: true
```

The calling workflow passes them explicitly:

```yaml
- uses: hiero-org/hiero-automation-actions/actions/on-pr-close@v1
  with:
    repo:      ${{ github.event.repository.name }}
    owner:     ${{ github.repository_owner }}
    pr-number: ${{ github.event.pull_request.number }}
    username:  ${{ github.event.pull_request.user.login }}
```

And inside the action, inputs are read cleanly via `core.getInput()`:

```js
const repo     = core.getInput('repo');
const owner    = core.getInput('owner');
const prNumber = core.getInput('pr-number');
const username = core.getInput('username');
```

There is no ambient `context` object with runner-injected fields. The mock surface in tests shrinks to exactly what you declared — nothing more:

```js
jest.mock('@actions/core', () => ({
  getInput: (name) => ({
    'repo':      'hiero-sdk-python',
    'owner':     'hiero-ledger',
    'pr-number': '42',
    'username':  'parv'
  }[name])
}));
```

This is especially critical for workflows running on `pull_request_target`, which carries elevated permissions. Being able to fully unit test logic before it reaches a real repository isn't just convenient — it's a safety requirement.

Additional benefits:
- One fix in the action repo propagates to all repositories automatically
- Repositories can pin to `@v1` and upgrade to `@v2` deliberately — no surprise breaking changes
- Logic complexity (loops, error handling, conditional branching, API calls) belongs in JS, not YAML

✅ Fully unit testable without a live GitHub environment  
✅ Reusable across repositories via versioned references  
✅ Safe to iterate — breaking changes don't silently affect all repos

**Action Repository Structure (Monorepo)**

All actions are bundled into a single repository — `hiero-automation-actions`. This avoids duplicating shared logic and tests across multiple repos, keeps versioning consistent, and means one PR covers changes to any action:

```
hiero-automation-actions/
├── package.json                  # Shared dev dependencies
│
├── actions/                      # One folder per action
│   ├── on-pr-close/
│   │   ├── action.yml            # Input declarations
│   │   ├── src/
│   │   │   └── index.js          # Entry point — reads inputs, calls logic
│   │   └── dist/
│   │       └── index.js          # Bundled output (via ncc) — what GitHub runs
│   │
│   ├── on-pr-update/
│   │   ├── action.yml
│   │   ├── src/index.js
│   │   └── dist/index.js
│   │
│   └── on-issue-comment/
│       ├── action.yml
│       ├── src/index.js
│       └── dist/index.js
│
├── src/                          # Shared logic — used by all actions
│   ├── logic/
│   │   ├── recommend-issues.js
│   │   ├── label-normalizer.js
│   │   └── comment-builder.js
│   └── helpers/
│       ├── constants.js
│       └── github-client.js
│
└── tests/                        # Shared test suite
    ├── recommend-issues.test.js
    ├── label-normalizer.test.js
    └── helpers/
        └── mock-factory.js
```

Each action is referenced by its subfolder path — GitHub supports this natively:

```yaml
- uses: hiero-org/hiero-automation-actions/actions/on-pr-close@v1
- uses: hiero-org/hiero-automation-actions/actions/on-pr-update@v1
```

Since all actions share a version tag, they evolve together — which is appropriate here given they share the same logic layer and config system.

---

### 4.3 Orchestration Layer (GitHub App)

**Responsibility: Central Decision Making**

- Receives structured inputs from actions
- Loads repository-specific configuration
- Determines which rules apply and what outcome to produce

This replaces scattered and duplicated logic currently embedded in per-repository scripts.

---

### 4.4 Rule Engine

**Responsibility: Core System Logic**

- Contributor progression logic
- Issue recommendation logic
- Assignment eligibility checks
- Label normalization

Key principle: decisions are based on contributor history, not just the last action. Shared logic is applied consistently across all repositories.

---

## 5. Migration Strategy

### Why Migration Cost Is Low

The V1 C++ setup already made the right call — writing logic in modular, pure JavaScript functions rather than embedding it in YAML. This means the migration to standalone JS Actions is a **restructuring exercise, not a rewrite**.

### What Moves Without Changing

All pure logic functions are completely reusable as-is. These functions don't know anything about how they're called — they take inputs and return outputs. They move directly into `src/logic/` in the monorepo with zero changes:

- Progression and eligibility logic
- Issue recommendation and grouping logic
- Label normalization
- Comment building functions
- The existing test suite — since tests already mock at the function level, they work identically in the new structure

### What Changes

Only the **entry point** changes. Instead of receiving an injected `github` and `context` object from `github-script`, each action reads clean, declared inputs via `core.getInput()` and constructs what it needs explicitly. The business logic underneath is untouched.

```
V1 Structure                        V2 Structure
─────────────────────────────────   ──────────────────────────────────────
.github/
  scripts/
    helpers/          ──────────▶   hiero-automation-actions/src/helpers/
    logic modules     ──────────▶   hiero-automation-actions/src/logic/
    tests/            ──────────▶   hiero-automation-actions/tests/
    entry-point.js    ──────────▶   hiero-automation-actions/actions/
                                      <event>/src/index.js
                                    (core.getInput() + action.yml)
```

### Migration Path Per Workflow

1. Add a new folder under `actions/` in the monorepo
2. Define inputs explicitly in `action.yml`
3. Write `index.js` — replace `context.*` reads with `core.getInput()` calls
4. Move logic modules into `src/logic/` — no changes needed
5. Move tests into `tests/` — no changes needed
6. Update the calling workflow to use the monorepo path via `with:`
7. Run existing test suite — should pass without modification

The first migration establishes the pattern. Every subsequent workflow follows the same steps with decreasing effort.

---

## 6. End-to-End Flow

### Step-by-step: PR Merged → Issue Recommendation

**1. Event happens**
```
PR merged
```

**2. Workflow triggers**
```yaml
- uses: hiero-org/hiero-automation-actions/actions/on-pr-close@v1
  with:
    owner:     ${{ github.repository_owner }}
    repo:      ${{ github.event.repository.name }}
    pr-number: ${{ github.event.pull_request.number }}
    username:  ${{ github.event.pull_request.user.login }}
```

**3. JS Action collects context**
```js
const payload = {
  user:           core.getInput('username'),
  repo:           core.getInput('repo'),
  completedIssue: "beginner"
};
```

**4. JS Action calls GitHub App**
```js
await fetch("https://your-app.com/recommend", {
  method: "POST",
  body: JSON.stringify(payload)
});
```

**5. GitHub App runs logic**
```js
function recommend(userData) {
  const progress = getUserProgress(userData.user);
  if (eligibleForIntermediate(progress)) return "intermediate";
  return "beginner";
}
```

**6. App returns result**
```json
{
  "level": "intermediate",
  "issues": [...]
}
```

**7. JS Action executes**
```js
postComment(result.issues);
```

---

## 7. Configuration System

### Proposed Config Schema

Each repository opts in by adding `.hiero/automation.yml`:

```yaml
version: 1

workflows:
  /assign:
    enabled: true

  issue-recommendation:
    enabled: true
    max_recommendations: 5

progression:
  thresholds:
    good-first-issue: 0
    beginner: 1
    intermediate: 3
    advanced: 7

labels:
  mappings:
    - from: "skill: beginner"
      to: "beginner"
    - from: "skill: intermediate"
      to: "intermediate"
    - from: "good first issue"
      to: "good-first-issue"

  status:
    ready_for_dev: "status: ready for dev"
    in_progress: "status: in progress"
    blocked: "status: blocked"
    awaiting_triage: "status: awaiting triage"

notifications:
  maintainer_team: "@hiero-ledger/maintainers"
```

### How Repository Opt-In Works

1. Repository adds `.hiero/automation.yml`
2. GitHub App detects the config on startup or webhook
3. All subsequent events are processed through the config
4. Repositories without a config file are ignored — no implicit behavior

### Label Normalization

```
"skill: beginner"  ──┐
"beginner"         ──┼──▶  normalized: "beginner"
"Beginner"         ──┘
```

Normalization happens before any logic runs, so progression rules never break due to label format differences between repositories.

### Config Versioning

- The `version` field allows safe migration between schema versions
- Breaking changes increment the version number
- The app maintains backward compatibility for at least one prior version
- Repositories are notified via issue comment when migration is required

---

## 8. Safety & Permissions

### Key Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Malicious PRs from forks | High | Never use `pull_request` for sensitive ops; use `pull_request_target` carefully |
| Workflow injection via PR body | High | Never interpolate PR body into `run:` steps; validate all inputs |
| Excessive token permissions | Medium | Request only required scopes per workflow |
| Unintended automation on wrong repos | Medium | Explicit opt-in via config; no implicit behavior |
| Bot actions not traceable | Low | Log all decisions with reasoning |

### Proposed Safeguards

- **Minimal GitHub token permissions** — each workflow requests only the scopes it needs
- **Careful use of `pull_request_target`** — sensitive operations only run after trust verification; especially important given the elevated permissions this event carries
- **Input validation** — all external inputs sanitized before use; never passed to shell commands
- **Explicit opt-in** — app never acts on repositories that haven't added a config file
- **Full unit test coverage before deployment** — the inability to predict the GitHub environment with inline JS is exactly why standalone actions with mockable inputs are the right approach here

### Audit & Transparency

- All decisions logged with timestamp, repository, event, rule applied, and outcome
- Bot comments include a brief explanation of why an action was taken
- All automated actions are attributable to a specific config version

---

## 9. Testing Strategy

### Layers of Testing

**Unit Tests — Rule Engine & Logic**

Test progression logic, label normalization, and recommendation selection in isolation. No GitHub API calls required.

With standalone JS Actions, the mock surface is exactly the declared inputs — nothing more. There is no ambient runner context to reconstruct, no event payload shape to get right. Each input is a named, typed value that tests control precisely:

```js
// V1 — manually reconstructing runner context
const context = {
  eventName: 'pull_request',
  repo: { owner: 'test-owner', repo: 'test-repo' },
  payload: {
    pull_request: {
      number: 1,
      user: { login: 'parv', type: 'User' },
      body: 'Fixes #42',
      labels: [],
      assignees: [],
    },
  },
};

// V2 — mocking declared inputs only
jest.mock('@actions/core', () => ({
  getInput: (name) => ({ 'pr-number': '1', 'username': 'parv' }[name])
}));
```

The V1 approach requires knowing the exact structure of the GitHub Actions runtime context for every event type. The V2 approach requires knowing only what you declared. This makes tests smaller, more reliable, and immune to changes in GitHub's event payload structure.

**Integration Tests — GitHub App**
- Dedicated test GitHub organization with real repositories
- Trigger test events and verify the app responds correctly
- Isolated from production repositories entirely

**Workflow Tests — GitHub Actions**
- Use `act` (local GitHub Actions runner) to test workflow triggers locally
- Validate that workflows pass correct payloads to JavaScript actions
- Catch permission and event-filter errors before deployment

**Safety Tests**
- Explicitly test fork PR scenarios to verify sensitive operations are blocked
- Test that repositories without config receive no automated actions
- Test that invalid or missing labels degrade gracefully

### CI Pipeline

All tests run on every PR before merge. Unit tests run first as a fast gate — if they fail, integration tests are skipped.

---

## 10. Scalability

### What Scalability Means for This System

Scalability here is not about handling high traffic — GitHub Actions manages that. It means: **can a new repository adopt this system in under an hour, with zero changes to shared code?**

| Concern | Approach |
|---|---|
| Adding a new repository | Add `.hiero/automation.yml` — no code changes |
| Adding a new workflow type | Add a new action folder in the monorepo + config flag |
| Different rules per repository | Handled entirely by config — same codebase |
| Rolling out changes safely | Feature flags + versioned actions (`@v1`, `@v2`) |
| Fixing a bug | Fix once in the monorepo — all repos get it automatically |

---

## 11. Versioning & Rollout Strategy

- Versioned JavaScript actions (`@v1`, `@v2`) — repositories pin to a version and upgrade deliberately
- All actions in the monorepo share a version tag — they evolve together
- Config versioning for safe schema migration
- Feature flags for gradual per-repository rollout
- Backward compatibility maintained for at least one prior version

---

## 12. Summary

This approach aims to eliminate duplicated logic, improve consistency across repositories, and enable scalable and configurable automation — while ensuring safety, auditability, and maintainability.

The V1 C++ setup demonstrated that moving logic into modular JS was the right direction. The gap it left — inline scripts tightly coupled to the runner context, which can't be fully tested, versioned, or reused without manual copying — is exactly what this architecture addresses. Because the existing logic is already written as pure JS functions, the migration path is low-cost: reuse everything, replace only the entry point, bundle all actions into a single monorepo.

The focus is not only on automation, but on designing a reliable, extensible system that supports contributor growth and reduces maintainer overhead across the Hiero ecosystem.