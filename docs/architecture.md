# Claude Asset Writer Architecture

## Overview

The Claude Asset Writer is a system for authoring Claude Code assets (rules, agents, skills, commands) according to established standards. It handles complexity detection, decomposition of large inputs, type and weight classification, model selection, and user confirmation flows.

## System Components

### Orchestrator

| Component | File | Purpose |
|---|---|---|
| rewrite-ccasset | `commands/rewrite-ccasset.md` | Main entry point. Routes input through detection, authoring, and model selection phases. |

### Detection Agents

| Component | File | Model | Purpose |
|---|---|---|---|
| complexity-detector | `agents/complexity-detector.md` | sonnet | Analyzes input to determine if it needs decomposition into multiple assets. |
| model-selector | `agents/model-selector.md` | haiku | Analyzes written agent files to recommend haiku/sonnet/opus. |

### Authoring Agents (Lightweight)

| Component | File | Model | Purpose |
|---|---|---|---|
| author-rule | `agents/author-rule.md` | sonnet | Authors rules (always-on behavioral instructions). |
| author-agent | `agents/author-agent.md` | sonnet | Authors lightweight agents (focused, single-purpose). |
| author-skill | `agents/author-skill.md` | sonnet | Authors skills (contextually-activated domain knowledge). |
| author-command | `agents/author-command.md` | sonnet | Authors lightweight commands (user-invoked actions). |

### Authoring Agents (Heavyweight)

| Component | File | Model | Purpose |
|---|---|---|---|
| author-orchestrator | `agents/author-orchestrator.md` | sonnet | Authors orchestrator commands that dispatch to multiple agents. |
| author-heavyweight-agent | `agents/author-heavyweight-agent.md` | sonnet | Authors agents that coordinate subagents. |

### Decomposition Agent

| Component | File | Model | Purpose |
|---|---|---|---|
| asset-decomposer | `agents/asset-decomposer.md` | opus | Breaks complex input into multiple coordinated assets. |

---

## Data Flow

### Entry Point

All requests enter through `/rewrite-ccasset`:

```
User Input (file path or content)
         │
         ▼
  ┌─────────────┐
  │  /rewrite-  │
  │   ccasset   │
  └──────┬──────┘
         │
         ▼
```

### Phase 1: Complexity Detection

```
         │
         ▼
  ┌─────────────────┐
  │   complexity-   │
  │    detector     │
  └────────┬────────┘
           │
     ┌─────┴─────┐
     │           │
  SIMPLE      COMPLEX
     │           │
     ▼           ▼
```

### Phase 2a: Decomposition Flow (Complex)

When complexity-detector returns `COMPLEXITY: complex`:

```
  COMPLEX
     │
     ▼
  ┌─────────────────┐
  │ asset-decomposer│
  └────────┬────────┘
           │
           ├──────────────────────────────────────┐
           │                                      │
           ▼                                      ▼
  ┌─────────────────┐                   ┌─────────────────┐
  │  author-agent   │ (×N)              │author-orchestr- │
  │  author-skill   │                   │     ator        │
  │  author-rule    │                   │  (if needed)    │
  └────────┬────────┘                   └────────┬────────┘
           │                                      │
           ▼                                      │
  ┌─────────────────┐                             │
  │ model-selector  │ (×N for agents)             │
  └────────┬────────┘                             │
           │                                      │
           ├──────────────────────────────────────┘
           │
           ▼
  ┌─────────────────┐
  │  User Confirm   │
  │  (all models)   │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │     Report      │
  │  (all assets)   │
  └─────────────────┘
```

### Phase 2b: Single Asset Flow (Simple)

When complexity-detector returns `COMPLEXITY: simple`:

```
  SIMPLE
     │
     ▼
  ┌─────────────────┐
  │ Determine Type  │
  │  & Weight Class │
  └────────┬────────┘
           │
     ┌─────┴─────┬─────────┬─────────┬─────────┐
     │           │         │         │         │
   rule       agent     skill    command   heavy
     │           │         │         │      agent/cmd
     ▼           ▼         ▼         ▼         ▼
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐
  │author│  │author│  │author│  │author│  │author-   │
  │-rule │  │-agent│  │-skill│  │-cmd  │  │orchestr- │
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘  │ator/     │
     │         │         │         │      │heavyweight│
     │         ▼         │         │      └────┬─────┘
     │    ┌──────────┐   │         │           │
     │    │  model-  │   │         │           │
     │    │ selector │   │         │           ▼
     │    └────┬─────┘   │         │      ┌──────────┐
     │         │         │         │      │  model-  │
     │         ▼         │         │      │ selector │
     │    ┌──────────┐   │         │      └────┬─────┘
     │    │  User    │   │         │           │
     │    │ Confirm  │   │         │           ▼
     │    └────┬─────┘   │         │      ┌──────────┐
     │         │         │         │      │  User    │
     │         ▼         │         │      │ Confirm  │
     │    ┌──────────┐   │         │      └────┬─────┘
     │    │  Apply   │   │         │           │
     │    │  Model   │   │         │           ▼
     │    └────┬─────┘   │         │      ┌──────────┐
     │         │         │         │      │  Apply   │
     │         │         │         │      │  Model   │
     └────┬────┴────┬────┴────┬────┴──────┴────┬─────┘
          │                                    │
          ▼                                    │
     ┌──────────┐                              │
     │  Report  │◀─────────────────────────────┘
     └──────────┘
```

---

## Classification Logic

### Asset Type Detection

| Content Characteristic | Target Type |
|---|---|
| Language/framework-specific (Go, Python, React, etc.) | skill |
| Workflow sequence ("do X, then Y, then Z") | command |
| Focused expertise for dispatch | agent |
| Truly universal behavior (every session, every project) | rule |

### Weight Class Detection

**Heavyweight Indicators:**
- Contains "Task tool" or "dispatch" or "subagent"
- Orchestrates multiple phases
- Description includes "coordinates" or "orchestrates"

**Routing Matrix:**

| Type + Weight | Authoring Agent |
|---|---|
| rule | author-rule |
| skill | author-skill |
| command + lightweight | author-command |
| command + heavyweight | author-orchestrator |
| agent + lightweight | author-agent |
| agent + heavyweight | author-heavyweight-agent |

### Complexity Detection

**Complex Indicators (triggers decomposition):**
- Line count >200
- Multiple distinct domains/responsibilities
- "first X, then Y, finally Z" phase patterns
- Multiple unrelated outputs
- Conditional workflow branches

**Simple Indicators (single asset):**
- Line count <150
- Focused scope
- Single domain
- Linear, non-separable flow

---

## Model Selection Logic

The model-selector agent analyzes written agent files:

| Signal | Recommended Model |
|---|---|
| Tools include Task (dispatches subagents) | opus |
| Multi-phase coordination, complex reasoning | opus |
| Security analysis, architecture review | opus |
| Standard code implementation, review | sonnet |
| Tools include Write, Edit, Bash | sonnet |
| Procedure has 5-10 steps | sonnet |
| Simple data collection, formatting | haiku |
| Tools are read-only (Read, Glob, Grep) | haiku |
| Procedure has <5 steps | haiku |

---

## File Size Guidelines

| Asset Type | Target Lines |
|---|---|
| Rules | <50 |
| Lightweight agents | ~80 |
| Heavyweight agents | 100-150 |
| Skills | ~100 (main SKILL.md) |
| Lightweight commands | ~80 |
| Orchestrator commands | 100-150 |

---

## User Interaction Points

The system prompts users at these decision points:

1. **Model Selection Confirmation**
   - After model-selector recommends a model
   - User can accept or choose alternative (haiku/sonnet/opus)

2. **Decomposition Model Confirmation**
   - After decomposer creates multiple agents
   - User confirms all model selections at once

3. **Registration Prompts** (in authoring agents)
   - "Should this be registered in any orchestrators?"
   - "Should this be referenced in CLAUDE.md?"

---

## Extension Points

### Adding New Asset Types

1. Create `author-<type>.md` in `agents/`
2. Add type detection logic to `rewrite-ccasset.md` Step 3
3. Add routing in Step 4

### Adding New Complexity Patterns

1. Update `complexity-detector.md` with new indicators
2. Update `asset-decomposer.md` if new decomposition logic needed

### Adding New Model Selection Signals

1. Update `model-selector.md` with new criteria
