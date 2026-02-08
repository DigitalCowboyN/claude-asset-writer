# Claude Asset Writer

A system for authoring Claude Code assets (rules, agents, skills, commands) according to established standards. Handles complexity detection, scope classification (personal vs project), automatic decomposition of large inputs, type and weight classification, and model selection.

## Features

- **Automatic Type Detection** — Determines if input should become a rule, agent, skill, or command
- **Scope Classification** — Distinguishes personal config (`~/.claude/`) from project config (`CLAUDE.md`)
- **Weight Classification** — Distinguishes lightweight (focused) from heavyweight (orchestrating) assets
- **Complexity Detection** — Identifies when input is too complex for a single asset
- **Automatic Decomposition** — Breaks complex input into multiple coordinated assets with per-component scope
- **Model Selection** — Recommends appropriate model (haiku/sonnet/opus) for agents
- **User Confirmation** — Prompts for scope decisions, model selection, and registration

## Quick Start

### Installation

Copy the contents to your Claude Code configuration:

```bash
# Copy agents
cp agents/*.md ~/.claude/agents/

# Copy commands
cp commands/*.md ~/.claude/commands/
```

### Usage

Invoke the rewrite command with content to process:

```bash
# From a file
/rewrite-ccasset ~/path/to/content.md

# From inline content
/rewrite-ccasset
[paste or describe content]
```

## Components

### Orchestrator

| File | Purpose |
|---|---|
| `commands/rewrite-ccasset.md` | Main entry point for all asset writing |

### Detection Agents

| File | Purpose |
|---|---|
| `agents/complexity-detector.md` | Determines if input needs decomposition |
| `agents/scope-detector.md` | Classifies content as personal or project-scoped |
| `agents/model-selector.md` | Recommends model for agent files |

### Authoring Agents

| File | Asset Type | Weight |
|---|---|---|
| `agents/author-rule.md` | Rules | — |
| `agents/author-agent.md` | Agents | Lightweight |
| `agents/author-heavyweight-agent.md` | Agents | Heavyweight |
| `agents/author-skill.md` | Skills | — |
| `agents/author-command.md` | Commands | Lightweight |
| `agents/author-orchestrator.md` | Commands | Heavyweight |

### Decomposition

| File | Purpose |
|---|---|
| `agents/asset-decomposer.md` | Creates multiple assets from complex input |

## How It Works

### Simple Input Flow

```
Input → Complexity (simple) → Scope Detection → Type Detection → Authoring Agent → Model Selection → Done
```

Personal content flows through without prompting. Project content prompts the user.

### Complex Input Flow

```
Input → Complexity (complex) → Decomposer → Scope (per component) → Authoring Agents → Model Selection → Done
```

Personal components go to `~/.claude/`. Project components (with user confirmation) go to project `CLAUDE.md` or project `.claude/` directory.

## Asset Types

| Type | Description | Example |
|---|---|---|
| **Rule** | Always-on behavior, every session | Security checks, commit conventions |
| **Agent** | Focused expertise, dispatched via Task | Code reviewer, test generator |
| **Skill** | Domain knowledge, auto-activates by context | Go idioms, React patterns |
| **Command** | User-invoked action via `/command` | `/review-pr`, `/run-tests` |

## Weight Classes

| Weight | Description | Indicators |
|---|---|---|
| **Lightweight** | Single focused task | No Task tool, linear procedure |
| **Heavyweight** | Coordinates subagents | Uses Task tool, multi-phase, orchestrates |

## Model Selection

| Model | Use Case |
|---|---|
| **haiku** | Simple data collection, formatting, read-only operations |
| **sonnet** | Standard coding work, implementation, review |
| **opus** | Complex reasoning, coordination, security analysis |

## File Size Guidelines

| Asset Type | Target Lines |
|---|---|
| Rules | <50 |
| Lightweight agents | ~80 |
| Heavyweight agents | 100-150 |
| Skills | ~100 |
| Commands | ~80-150 |

## Documentation

See `docs/architecture.md` for detailed system architecture and data flow diagrams.

## Scope Classification

| Scope | Destination | User Prompt? |
|---|---|---|
| **Personal** | `~/.claude/` | No |
| **Project** | `<project>/CLAUDE.md` or `<project>/.claude/` | Yes |
| **Ambiguous** | User decides | Yes |

Project-scoped rules and skills are appended as inline sections to `CLAUDE.md`. Project-scoped agents and commands are written as files to the project's `.claude/` directory.

## Example: Mixed-Scope Input (Cursor Rules)

Input: Cursor rules with coding style + project config

Output:
```
Personal Assets (written to ~/.claude/):
  ~/.claude/skills/coding-style/SKILL.md
  ~/.claude/skills/naming-conventions/SKILL.md

Project Assets (appended to CLAUDE.md):
  ## Project Structure (directory layout)
  ## Database Tooling (Alembic, SQLAlchemy)
  ## Deployment (Docker, Redis)
```

## Example: Complex Input Decomposition

Input: 400-line "super agent" handling security + performance + reporting

Output:
```
Created 4 assets:

1. ~/.claude/agents/security-reviewer.md (opus)
2. ~/.claude/agents/performance-analyzer.md (sonnet)
3. ~/.claude/agents/report-generator.md (haiku)
4. ~/.claude/commands/code-auditor.md (orchestrator)
```

The orchestrator coordinates the three agents in sequence.

## Design Principles

1. **Minimal Dispatch Prompts** — Subagents know their job; orchestrators pass content + goal only
2. **Progressive Complexity** — Simple inputs get simple handling; complexity triggers decomposition
3. **Scope Awareness** — Personal style goes to `~/.claude/`; project config goes to project directories
4. **User Control** — Scope decisions, model selection, and registration are confirmed, not automatic
5. **Focused Assets** — Each asset does one thing well; complex tasks become multiple assets
