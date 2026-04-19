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

- ❌ Logic duplication across workflows and repositories  
- ❌ Inconsistent behavior (labels, progression rules)  
- ❌ Tight coupling to repository-specific configurations  
- ❌ Poor portability across repositories  
- ❌ No centralized decision-making system  
- ❌ Difficult to scale and maintain  

---

## 3. Proposed Architecture

### High-Level Design

The system is structured into four logical layers:

1. **Trigger Layer (GitHub Actions)**
2. **Execution Layer (Reusable JavaScript Actions)**
3. **Orchestration Layer (GitHub App / Service)**
4. **Rule Engine (Centralized Logic)**

---

### 3.1 Trigger Layer (GitHub Actions)

**Responsibility: Event Handling Only**

- Listens to:
  - `pull_request`
  - `issues`
  - `issue_comment`
- Passes structured event data to the execution layer

✅ No business logic in workflows  
✅ Lightweight and consistent across repositories  

---

### 3.2 Execution Layer (Reusable JavaScript Actions)

**Responsibility: Standardized Execution Units**

- Encapsulates reusable logic as JavaScript actions
- Handles:
  - data collection from GitHub context
  - API interactions
  - applying decisions returned by the orchestration layer

Example responsibilities:
- posting comments
- assigning issues
- applying labels

✅ Reusable across repositories  
✅ Versioned and maintainable  
✅ Removes duplication from workflows  

---

### 3.3 Orchestration Layer (GitHub App)

**Responsibility: Central Decision Making**

- Receives structured inputs from actions
- Loads repository-specific configuration
- Determines:
  - which rules apply
  - what outcome should be produced

This replaces scattered and duplicated logic currently embedded in workflows.

---

### 3.4 Rule Engine

**Responsibility: Core System Logic**

Handles:
- Contributor progression logic  
- Issue recommendation logic  
- Assignment eligibility checks  

Key principles:
- Decisions are based on contributor history (not just last action)
- Shared logic is applied consistently across repositories

---

### Key Design Principle

> Separate **“when something runs”** from **“what should happen”**

- Workflows → trigger events  
- Actions → execute operations  
- App → decides behavior  

---

## 4. Hybrid Model (GitHub Actions + JavaScript Actions + GitHub App)

### Problem with Current Approach

- Logic and execution are tightly coupled inside workflows  
- Hard to reuse across repositories  
- Difficult to maintain and evolve  

---

### Proposed Hybrid Model

#### GitHub Actions → Trigger Layer
- Event-driven
- Minimal logic

#### JavaScript Actions → Execution Layer
- Reusable, versioned automation units
- Handle API calls and execution

#### GitHub App → Decision Layer
- Centralized logic and configuration
- Determines system behavior

---

### Example Flow

1. PR is merged  
2. Workflow triggers  
3. JavaScript action collects context  
4. Action sends structured request → GitHub App  
5. App:
   - evaluates contributor history  
   - applies progression rules  
6. App returns decision  
7. Action executes (e.g., posts recommendations)

---

## 5. Configuration System

### Goals

- Repository-specific customization  
- No code changes required  
- Consistent behavior across repositories  

---

### Proposed Config Design

Each repository defines:

- Enabled workflows  
- Skill progression thresholds  
- Label mappings  
- Feature flags  

---

### Label Normalization

Support multiple formats:

- `skill: beginner`  
- `beginner`  

➡️ Normalize internally to a consistent representation  

---

## 6. Safety & Permissions

### Key Risks

- Malicious PRs from forks  
- Workflow injection  
- Excessive permissions  

---

### Proposed Safeguards

- Minimal GitHub token permissions  
- Careful use of `pull_request_target`  
- Input validation (never trust PR body directly)  
- Restrict sensitive operations  

---

### Audit & Transparency

- Log all decisions made by the system  
- Include reasoning in bot comments  
- Ensure actions are traceable and explainable  

---

## 7. Scalability

### Design Goals

- Reusable across multiple repositories  
- Minimal duplication  
- Easy adoption  

---

### Approach

- Centralized rule engine  
- Reusable JavaScript actions  
- Plug-and-play repository configurations  

---

## 8. Versioning & Rollout Strategy

### Feature Flags

- Enable/disable features per repository  
- Gradual rollout of new functionality  

---

### Versioned Actions & Configs

- Versioned JavaScript actions (`@v1`, `@v2`)  
- Config versioning for safe migration  
- Backward compatibility across repositories  

---

## 9. Future Improvements

- Reviewer recommendation system  
- PR health scoring  
- Contributor progression analytics  
- Maintainer dashboards and insights  

---

## 10. Summary

This approach aims to:

- Eliminate duplicated logic  
- Improve consistency across repositories  
- Enable scalable and configurable automation  
- Ensure safety, auditability, and maintainability  

The focus is not only on automation, but on designing a reliable, extensible system that supports contributor growth and reduces maintainer overhead across the Hiero ecosystem.