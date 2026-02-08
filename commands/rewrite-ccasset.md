---
description: Rewrite any asset (rule, agent, skill, command) to meet local standards
allowed-tools: [Task, Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion]
---

# Rewrite CC Asset

Take any Claude Code asset and rewrite it to meet local standards. Detects complexity, determines asset type and weight class, then dispatches to the correct authoring agent(s).

## When to Use

- Converting content from external source into a proper asset
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

The decomposer will:
1. Create multiple assets by dispatching to authoring agents
2. Run model selection for each agent created
3. Create an orchestrator if needed
4. Return a complete report

Present the decomposer's report to the user. **STOP HERE** — decomposition flow is complete.

---

**If simple, continue with single-asset flow:**

### Step 3: Determine Asset Type

| Content Characteristic | Target Type |
|---|---|
| Language/framework-specific | skill |
| Workflow sequence (do X, then Y) | command |
| Focused expertise for dispatch | agent |
| Truly universal behavior | rule |

### Step 4: Detect Weight Class

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

### Step 5: Dispatch to Authoring Agent

Launch the appropriate agent via Task with minimal prompt:

```
Rewrite to standards:
<content>
[the input content]
</content>
```

The authoring agent has its own template and requirements. Note the file path it creates.

### Step 6: Model Selection (Agents Only)

Skip this step if asset type is rule, skill, or command.

For agents (lightweight or heavyweight), dispatch to `model-selector`:

```
Analyze this agent file and recommend the appropriate model:
<file_path>[path from Step 5]</file_path>
```

### Step 7: Confirm Model with User

Present the recommendation:

```
Model recommendation: [model]
Rationale: [rationale from model-selector]

Accept or adjust?
```

Use AskUserQuestion with options:
- The recommended model (marked recommended)
- The two alternatives

### Step 8: Apply Model Selection

Edit the agent file's frontmatter to set the confirmed model:

```yaml
model: [confirmed model]
```

### Step 9: Report

```markdown
## Asset Rewritten

**Source:** [file path or "inline content"]
**Created:** ~/.claude/<path>
**Type:** [rule|agent|skill|command]
**Weight:** [lightweight|heavyweight]
**Model:** [haiku|sonnet|opus] (if agent)

**Type transformation:** [if applicable]

**Notes from authoring agent:**
[any concerns or recommendations]
```

## Examples

**Simple rewrite:**
```
/rewrite-ccasset ~/reference/everything-claude-code/rules/security.md
→ Complexity check: simple (focused scope, <150 lines)
→ Determines: universal behavior → rule
→ Dispatches to author-rule
→ Creates ~/.claude/rules/security.md
```

**Complex decomposition:**
```
/rewrite-ccasset [400-line "super agent" doing security + performance + reporting]
→ Complexity check: complex (3 domains, sequential phases, >200 lines)
→ Plan: 3 agents + 1 orchestrator
→ Dispatches to asset-decomposer
→ Decomposer creates:
   - security-reviewer.md (agent, opus)
   - performance-analyzer.md (agent, sonnet)
   - report-generator.md (agent, haiku)
   - code-auditor.md (orchestrator)
→ Reports all 4 assets created
```

**Heavyweight agent (simple, not decomposed):**
```
/rewrite-ccasset [agent content with Task dispatches]
→ Complexity check: simple (single coordination responsibility)
→ Determines: agent + dispatches → heavyweight
→ Dispatches to author-heavyweight-agent
→ Creates ~/.claude/agents/coordinator.md
→ Model selection: opus (Task dispatch, coordination)
→ User confirms
→ Edits frontmatter: model: opus
```
