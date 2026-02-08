---
description: Rewrite any asset (rule, agent, skill, command) to meet local standards
allowed-tools: [Task, Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion]
---

# Rewrite CC Asset

Take any Claude Code asset and rewrite it to meet local standards. Detects complexity, classifies scope (personal vs project), determines asset type and weight class, then dispatches to the correct authoring agent(s).

## When to Use

- Converting content from external source (Cursor rules, shared configs)
- Rewriting an existing asset that doesn't meet standards
- Transforming content from one type to another (e.g., rule → skill)
- Decomposing complex input into multiple coordinated assets

## Input Formats

1. **File path:** `/rewrite-ccasset ~/.claude/rules/old-rule.md`
2. **Inline content:** `/rewrite-ccasset` then provide content

## Procedure

### Step 1: Get Input

If file path provided: read the file.
If inline content: use that directly.
If neither: ask "What would you like to rewrite? Provide a file path or paste the content."

### Step 2: Complexity Detection

Dispatch to `complexity-detector`:

```
Analyze this content for complexity:
<content>
[the input content]
</content>
```

The detector returns either:
- `COMPLEXITY: simple` → continue to Step 3
- `COMPLEXITY: complex` with decomposition plan → go to Step 2a

### Step 2a: Decomposition Flow (Complex Only)

If complex, dispatch to `asset-decomposer`:

```
Decompose this content into multiple assets:

<content>
[the original input]
</content>

<plan>
[the decomposition plan from complexity-detector]
</plan>
```

The decomposer handles scope detection per-component, user prompting for project content, and model selection internally.

Present the decomposer's report to the user. **STOP HERE** — decomposition flow is complete.

---

**If simple, continue with single-asset flow:**

### Step 3: Scope Detection

Dispatch to `scope-detector`:

```
Classify the scope of this content:
<content>
[the input content]
</content>
```

#### If SCOPE: personal
Continue to Step 4. Destination = `~/.claude/` (default).

#### If SCOPE: project
Prompt with AskUserQuestion:
- Header: "Scope"
- Question: "This content appears project-specific ([rationale]). How should it be handled?"
- Options:
  1. "Include in project at [cwd]" (Recommended)
  2. "Make it personal anyway"
  3. "Skip this content"

If "Include in project":
- Determine asset type (Step 4)
- If rule or skill → format as CLAUDE.md section (Step 5a)
- If agent or command → dispatch to authoring agent with project destination

If "Make personal": continue to Step 4 with default destination.
If "Skip": report "Content skipped." and stop.

#### If SCOPE: ambiguous
Prompt with AskUserQuestion:
- Header: "Scope"
- Question: "This could be personal or project-specific ([rationale]). Which is it?"
- Options:
  1. "Personal (save to ~/.claude/)"
  2. "Project (save to [cwd])"
  3. "Skip"

Route based on user choice.

#### Edge Case: Not in a Project Directory
If scope is project but cwd is home directory, ask:
"This looks project-specific, but you're not in a project directory. Make personal, or specify a project path?"

### Step 4: Determine Asset Type

| Content Characteristic | Target Type |
|---|---|
| Language/framework-specific | skill |
| Workflow sequence (do X, then Y) | command |
| Focused expertise for dispatch | agent |
| Truly universal behavior | rule |

### Step 5: Detect Weight Class

**Heavyweight indicators:**
- Contains "Task tool" or "dispatch" or "subagent"
- Orchestrates multiple phases
- Description includes "coordinates" or "orchestrates"

**Routing:**
- Command + heavyweight → `author-orchestrator`
- Agent + heavyweight → `author-heavyweight-agent`
- Command + lightweight → `author-command`
- Agent + lightweight → `author-agent`
- Skill → `author-skill`
- Rule → `author-rule`

### Step 5a: Project CLAUDE.md Formatting (Project-Scoped Rules/Skills Only)

If scope is project AND type is rule or skill, skip authoring agents. Format directly:

```markdown
## [Topic Name]

[Requirements as concise bullet points]
```

Check if `<project>/CLAUDE.md` exists:
- If not: create with `# Project Configuration` header
- If section with same name exists: ask "Replace, append below, or skip?"

Append the section to CLAUDE.md. Skip to Step 10 (Report).

### Step 6: Dispatch to Authoring Agent

Launch the appropriate agent via Task with minimal prompt:

```
Rewrite to standards:
<content>
[the input content]
</content>
<destination>[path, if project-scoped]</destination>
```

The authoring agent has its own template and requirements. Note the file path it creates.

### Step 7: Model Selection (Agents Only)

Skip if asset type is rule, skill, or command.

For agents, dispatch to `model-selector`:

```
Analyze this agent file and recommend the appropriate model:
<file_path>[path from Step 6]</file_path>
```

### Step 8: Confirm Model with User

Use AskUserQuestion with the recommended model and alternatives.

### Step 9: Apply Model Selection

Edit the agent file's frontmatter to set the confirmed model.

### Step 10: Report

```markdown
## Asset Rewritten

**Source:** [file path or "inline content"]
**Created:** [~/.claude/<path> or <project>/CLAUDE.md]
**Scope:** personal|project
**Type:** [rule|agent|skill|command]
**Weight:** [lightweight|heavyweight]
**Model:** [haiku|sonnet|opus] (if agent)

**Scope rationale:** [why personal or project]
**Type transformation:** [if applicable]
```

## Examples

**Personal skill:**
```
/rewrite-ccasset "Prefer functional style, avoid classes, use descriptive names"
→ Complexity: simple
→ Scope: personal (coding style preferences)
→ Type: skill
→ Dispatches to author-skill
→ Creates ~/.claude/skills/functional-style/SKILL.md
```

**Project config:**
```
/rewrite-ccasset "Use Alembic for migrations, frontend at /web/src/"
→ Complexity: simple
→ Scope: project (named tools, specific directories)
→ Prompt: "Project-specific. Include in project at /Users/me/myapp?"
→ User: "Include in project"
→ Type: rule → formats as CLAUDE.md section
→ Appends to /Users/me/myapp/CLAUDE.md
```

**Mixed (Cursor rules):**
```
/rewrite-ccasset [Cursor rules with style prefs + project config]
→ Complexity: complex (multiple domains)
→ Decomposer: 6 components, mixed scopes
→ Personal (3): written to ~/.claude/skills/
→ Project (3): user prompted → appended to CLAUDE.md
→ Ambiguous (1): user decides
```
