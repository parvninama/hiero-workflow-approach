# Hiero Workflow Automation – Design Approach

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
- Better organization of helper logic
- However:
  - Tightly coupled to repository-specific configurations
  - Inconsistent label formats (`skill: beginner` vs `beginner`)
  - Limited portability across repositories

---

### Key Problems Identified

| Problem | Impact |
|---|---|
| Logic duplication across workflows | High maintenance cost, inconsistent fixes |
| Inconsistent label formats | Progression logic breaks silently |
| Tight coupling to repo config | Cannot reuse across repositories |
| No centralized decision-making | Behavior diverges across repos over time |
| Poor portability | Each new repo requires rebuilding from scratch |

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
| Safety | ❌ Limited | ⚠️ Some improvements | ✅ Explicit permission + validation model |

---

## 4. Proposed Architecture

### High-Level Design

The system is structured into four logical layers:

1. **Trigger Layer** — GitHub Actions
2. **Execution Layer** — Reusable JavaScript Actions
3. **Orchestration Layer** — GitHub App
4. **Rule Engine** — Centralized Logic

---

### 4.1 Trigger Layer (GitHub Actions)

**Responsibility: Event Handling Only**

- Listens to: `pull_request`, `issues`, `issue_comment`
- Passes structured event data to the execution layer

✅ No business logic in workflows  
✅ Lightweight and consistent across repositories

---

### 4.2 Execution Layer (Reusable JavaScript Actions)

**Responsibility: Standardized Execution Units**

- Encapsulates reusable logic as JavaScript actions
- Handles data collection, API interactions, and applying decisions

✅ Reusable across repositories  
✅ Versioned and maintainable  
✅ Removes duplication from workflows

---

### 4.3 Orchestration Layer (GitHub App)

**Responsibility: Central Decision Making**

- Receives structured inputs from actions
- Loads repository-specific configuration
- Determines which rules apply and what outcome should be produced

This replaces scattered and duplicated logic currently embedded in workflows.

---

### 4.4 Rule Engine

**Responsibility: Core System Logic**

- Contributor progression logic
- Issue recommendation logic
- Assignment eligibility checks

Key principle: decisions are based on contributor history, not just last action. Shared logic is applied consistently across repositories.

---

### Key Design Principle

> Separate **"when something runs"** from **"what should happen"**

| Layer | Role |
|---|---|
| GitHub Actions | Trigger events |
| JavaScript Actions | Execute operations |
| GitHub App | Decide behavior |

---

## 5. End-to-End Flow

### Step-by-step: PR Merged → Issue Recommendation

**1. Event happens**
```
PR merged
```

**2. Workflow triggers**
```yaml
- uses: your-org/recommend-action@v1
```

**3. JS Action collects data**
```js
const payload = {
  user: "parv",
  repo: "hiero-sdk-cpp",
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

  if (eligibleForIntermediate(progress)) {
    return "intermediate";
  }

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

### What the App Actually Does

The GitHub App is the central logic engine. Your existing logic moves here:

- Progression rules
- Eligibility checks
- Fallback logic
- Label normalization
- Repository config loading

This makes the logic reusable across every repository that opts in — no duplication, no drift.

---

## 6. Configuration System

### Goals

- Repository-specific customization
- No code changes required
- Consistent behavior across repositories

### Proposed Config Schema

Each repository opts in by adding `.hiero/automation.yml`:

```yaml
version: 1

workflows:
  issue-recommendation:
    enabled: true
    max_recommendations: 5

  pr-draft-explainer:
    enabled: true

  ready-for-review-reminder:
    enabled: true
    reminder_after_days: 3

  contributor-onboarding:
    enabled: false

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

notifications:
  maintainer_team: "@hiero-ledger/maintainers"
```

### How Repository Opt-In Works

1. Repository adds `.hiero/automation.yml`
2. GitHub App detects the config on startup or webhook
3. All subsequent events are processed through the config
4. Repositories without a config file are ignored — no implicit behavior

### Label Normalization

All incoming labels are normalized internally before any logic runs:

```
"skill: beginner"  ──┐
"beginner"         ──┼──▶  normalized: "beginner"
"Beginner"         ──┘
```

This ensures progression logic never breaks due to label format inconsistency.

### Config Versioning

- The `version` field allows safe migration between schema versions
- Breaking changes increment the version number
- The app maintains backward compatibility for at least one prior version
- Repositories are notified via issue comment when migration is required

---

## 7. Safety & Permissions

### Key Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Malicious PRs from forks | High | Never use `pull_request` for sensitive ops; use `pull_request_target` carefully |
| Workflow injection via PR body | High | Never interpolate PR body into `run:` steps; validate all inputs |
| Excessive token permissions | Medium | Request only required scopes per workflow |
| Unintended automation on wrong repos | Medium | Explicit opt-in via config file; no implicit behavior |
| Bot actions not traceable | Low | Log all decisions with reasoning |

### Proposed Safeguards

- **Minimal GitHub token permissions** — each workflow requests only the scopes it needs
- **Careful use of `pull_request_target`** — sensitive operations only run after trust verification
- **Input validation** — all external inputs sanitized before use; never passed to shell commands
- **Explicit opt-in** — app never acts on repositories that haven't opted in
- **Permission boundaries** — clearly defined what the app is and is not allowed to do

### Audit & Transparency

- All decisions logged with timestamp, repository, event, rule applied, and outcome
- Bot comments include a brief explanation of why an action was taken
- Maintainers can inspect decision logs to understand or override any automated action
- All automated actions are attributable to a specific config version

---

## 8. Testing Strategy

### Layers of Testing

**Unit Tests — Rule Engine & Logic**

Test progression logic, label normalization, and recommendation selection in isolation. No GitHub API calls required.

```
getNextLevel('beginner')          → 'intermediate'
getFallbackLevel('beginner')      → 'good-first-issue'
normalizeLabel('skill: beginner') → 'beginner'
```

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

All tests run on every PR via GitHub Actions before merge. Unit tests run first as a fast gate — if they fail, integration tests are skipped.

---

## 9. Scalability

### What Scalability Means for This System

Scalability here is not about handling high traffic — GitHub Actions already manages that. It means: **can a new repository adopt this system in under an hour, with zero changes to shared code?**

### How the Architecture Achieves This

| Concern | Approach |
|---|---|
| Adding a new repository | Add `.hiero/automation.yml` — no code changes |
| Adding a new workflow type | Add a new rule to the rule engine + config flag |
| Different rules per repository | Handled entirely by config — same codebase |
| Rolling out changes safely | Feature flags + versioned actions (`@v1`, `@v2`) |
| Onboarding new maintainers | Config file is the single source of truth |

### Adoption Path for New Repositories

```
1. Repository adds .hiero/automation.yml
2. Enables only the workflows it needs
3. Customises thresholds and label mappings
4. GitHub App begins processing events immediately
5. Maintainer monitors initial runs via decision logs
6. Gradually enables additional workflows as confidence grows
```

---

## 10. Execution Plan

### Phase 1 — Orient & Audit (Weeks 1–2)
- Study existing V0 and V1 workflow implementations in detail
- Map all current behaviors, edge cases, and known gaps
- Align architecture proposal with mentor feedback
- Identify the first workflow to implement end-to-end
- Set up local development environment and test GitHub organization

### Phase 2 — Build First Workflow End-to-End (Weeks 3–7)
- Implement the full four-layer stack for one workflow (likely issue recommendation, since it already has a working prototype)
- Establish the config schema and opt-in mechanism
- Write unit and integration tests covering core logic and safety constraints
- Deploy to one repository, monitor, and iterate based on real behavior

### Phase 3 — Harden & Extend (Weeks 8–12)
- Implement remaining high-value workflows
- Harden safety constraints and audit logging
- Write documentation for maintainers and contributors
- Test adoption across a second repository with different config needs
- Produce recommendations for future extensions

---

## 11. Versioning & Rollout Strategy

### Feature Flags
- Enable/disable features per repository
- Gradual rollout of new functionality

### Versioned Actions & Configs
- Versioned JavaScript actions (`@v1`, `@v2`)
- Config versioning for safe migration
- Backward compatibility across repositories

---

## 12. Summary

This approach aims to:

- Eliminate duplicated logic
- Improve consistency across repositories
- Enable scalable and configurable automation
- Ensure safety, auditability, and maintainability

The focus is not only on automation, but on designing a reliable, extensible system that supports contributor growth and reduces maintainer overhead across the Hiero ecosystem.