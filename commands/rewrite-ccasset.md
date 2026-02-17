---
description: Rewrite any asset (rule, agent, skill, command) to meet local standards
allowed-tools: [Task, Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion]
---

# Rewrite CC Asset

Take any Claude Code asset and rewrite it to meet local standards. Detects complexity, classifies scope (personal vs project), determines asset type and weight class, then dispatches to the correct authoring agent(s). Supports single input or batch processing of multiple inputs with parallel execution.

## When to Use

- Converting content from external source (Cursor rules, shared configs)
- Rewriting an existing asset that doesn't meet standards
- Transforming content from one type to another (e.g., rule → skill)
- Decomposing complex input into multiple coordinated assets
- Processing multiple related inputs as a batch

## Input Formats

1. **File path:** `/rewrite-ccasset ~/.claude/rules/old-rule.md`
2. **Multiple file paths:** `/rewrite-ccasset file1.md file2.md file3.md`
3. **Glob pattern:** `/rewrite-ccasset ~/.claude/skills/python-*.md`
4. **Inline content:** `/rewrite-ccasset` then provide content

## Procedure

### Step 1: Get Input and Detect Batch

If file path provided: read the file.
If inline content: use that directly.
If neither: ask "What would you like to rewrite? Provide a file path or paste the content."

**Batch detection — check if input is multiple items:**
- Multiple file paths (space-separated, newline-separated, or glob pattern)
- Content with clear section boundaries (`---`, `===`, `##` headings with distinct topics)
- User explicitly says "process these" or provides multiple items

If **batch** → go to Step 1a (Batch Flow).
If **single** → go to Step 2 (Single-Item Flow).

---

## Batch Flow (Steps 1a–1i)

### Step 1a: Parse Batch Inputs

Extract individual items:
- Multiple file paths → read each file
- Glob pattern → expand with Glob tool, read matches
- Sectioned content → split on boundaries

Collect all content into a numbered list. Report: "Found [N] inputs to process."

### Step 1b: Cross-Input Analysis — BARRIER B1

Dispatch to `batch-analyzer` with ALL inputs:

```
Analyze these inputs for batch processing:
<inputs>
[all numbered inputs with content]
</inputs>
```

**BARRIER B1:** Wait for batch-analyzer to return.

Apply merge actions: combine inputs flagged for merging. Report merges to user. This produces the final work items list and phased execution plan.

### Step 1c: Parallel Complexity Detection — BARRIER B2

For each work item, dispatch to `complexity-detector` in parallel (all in one message):

```
Analyze this content for complexity:
<content>[work item N]</content>
```

**BARRIER B2:** Wait for ALL complexity detections to return.

Process results:
- Simple items → stay as single work items
- Complex items → dispatch to `asset-decomposer` in parallel, wait for all, flatten results into work items list (replace complex parent with its children)

### Step 1d: Parallel Scope Detection — BARRIER B3

For each work item (post-decomposition), dispatch to `scope-detector` in parallel:

```
Classify the scope of this content:
<content>[work item N]</content>
```

**BARRIER B3:** Wait for ALL scope detections to return.

Group: personal (no prompt needed), project (batch prompt), ambiguous (batch prompt).

### Step 1e: User Scope Decisions — BARRIER B4

**Project items** — prompt with AskUserQuestion:
- Header: "Batch scope"
- Question: "These [N] items appear project-specific: [list with rationale]. How to handle?"
- Options: "Include all in project at [cwd]" (Recommended) | "Skip all" | "Decide individually"

**Ambiguous items** — prompt with AskUserQuestion:
- Header: "Batch scope"
- Question: "These [N] items could be personal or project: [list with rationale]."
- Options: "All personal" | "All project" | "Decide individually"

If "Decide individually": prompt per item with [Include | Personal | Skip].

**BARRIER B4:** All scope decisions resolved. Remove skipped items.

### Step 1f: Phased Parallel Authoring — BARRIER B5

Determine asset type and weight class for each work item (same logic as Steps 4-5 in single flow).

Follow the phase ordering from batch-analyzer's execution plan. **For each phase:**

Dispatch all items in that phase in parallel to their appropriate authoring agents:
- Personal items → standard authoring agent dispatch
- Project rules → `author-rule` with destination `<project>/.claude/rules/<name>.md`
- Project skills → `author-skill` with destination `<project>/.claude/skills/<name>/`
- Project agents/commands → authoring agent with `<destination>` parameter

**Wait for all items in current phase to complete before starting next phase.**

**BARRIER B5:** All phases complete, all authoring done.

### Step 1g: Runtime Orchestrator (If Recommended)

Check the batch-analyzer's `RUNTIME_COORDINATION` result from Step 1b.

**If `RUNTIME_COORDINATION: no`** → skip to Step 1h.

**If `RUNTIME_COORDINATION: yes`** → prompt with AskUserQuestion:
- Header: "Coordination"
- Question: "These items form a runtime workflow. Recommended orchestrator with this execution order:\n[RUNTIME_ORDER from batch-analyzer]\nConfirm, or describe the correct order."
- Options: "Create orchestrator" (Recommended) | "Skip, keep independent"

If user selects "Other": they provide the correct execution order. Use their correction.

If user confirms or corrects, dispatch to `author-orchestrator`:
```
Create an orchestrator command that coordinates these agents:

Agents: [list of created agent names and purposes]

<execution_order>
[confirmed runtime order]
</execution_order>

Orchestrator name: [SUGGESTED_ORCHESTRATOR from batch-analyzer]
```

Wait for author-orchestrator to return. The new orchestrator is now included in model selection.

### Step 1h: Batch Model Selection — BARRIER B6

Collect all agent/orchestrator files created (including runtime orchestrator from Step 1g if applicable). Dispatch `model-selector` for each in parallel.

**BARRIER B6:** Wait for all model selections to return.

Present all at once with AskUserQuestion:
- Header: "Models"
- Question: "Model recommendations: [list]. Accept all, or specify adjustments?"
- Options: "Accept all" (Recommended) | "Adjust"

Apply confirmed models to each agent file.

### Step 1i: Add CLAUDE.md References + Report — BARRIER B7

**Add references for project-scoped rules/skills to CLAUDE.md:**

For each project-scoped rule or skill created in Step 1f:
1. Read current CLAUDE.md (create if missing with `# Project Configuration`)
2. Add brief reference under appropriate section (`## Rules` or `## Skills`), create section if missing
3. If reference already exists for this asset, skip

```markdown
## Rules
- See `.claude/rules/<name>.md` — [one-line description]

## Skills
- See `.claude/skills/<name>/` — [one-line description]
```

**BARRIER B7:** All references written.

**Report:**

```markdown
## Batch Processing Complete

**Inputs:** [original count] → **Work items:** [after merges] → **Assets created:** [final count]

### Merges Applied
- [input1] + [input2] → [merged name]

### Personal Assets (written to ~/.claude/)
| File | Type | Purpose | Model |
|---|---|---|---|
| ... | ... | ... | ... |

### Project Assets (written to <project>/.claude/)
| File | Type | Purpose | Model |
|---|---|---|---|
| .claude/rules/<name>.md | rule | ... | — |
| .claude/skills/<name>/ | skill | ... | — |
| .claude/agents/<name>.md | agent | ... | sonnet |

**Note:** References added to CLAUDE.md for rules and skills.

### Skipped
| Item | Reason |
|---|---|
| ... | ... |

### Runtime Orchestrator (if created)
**File:** [path]
**Coordinates:** [agent1] → [agent2] → ...

### Errors (if any)
| Item | Error | Status |
|---|---|---|
| ... | ... | Skipped after retry |
```

**STOP HERE** — batch flow is complete.

### Error Handling (Batch)

If a parallel dispatch fails: retry once. If retry fails: mark as failed, continue with remaining items. Report all failures in the final report. One failed item never blocks the batch.

---

## Single-Item Flow (Steps 2–10)

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
- If rule or skill → create project file + CLAUDE.md reference (Step 5a)
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

### Step 5a: Project-Scoped Rules/Skills

If scope is project AND type is rule or skill:

**Create the actual file** — dispatch to the appropriate authoring agent with project destination:
- For rules: `author-rule` with destination `<project>/.claude/rules/<name>.md`
- For skills: `author-skill` with destination `<project>/.claude/skills/<name>/`

```
Rewrite to standards:
<content>
[the input content]
</content>
<destination>[project path]/.claude/rules/<name>.md</destination>
```

**Then add a reference to CLAUDE.md:**

Check if `<project>/CLAUDE.md` exists:
- If not: create with `# Project Configuration` header

Add a brief reference under the appropriate section (create section if missing):

```markdown
## Rules
- See `.claude/rules/<name>.md` — [one-line description]

## Skills
- See `.claude/skills/<name>/` — [one-line description]
```

If reference already exists for this asset, skip. Continue to Step 10 (Report).

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
**Created:** [~/.claude/<path> or <project>/.claude/<path>]
**Scope:** personal|project
**Type:** [rule|agent|skill|command]
**Weight:** [lightweight|heavyweight]
**Model:** [haiku|sonnet|opus] (if agent)

**Scope rationale:** [why personal or project]
**Type transformation:** [if applicable]
```

## Examples

**Single personal skill:**
```
/rewrite-ccasset "Prefer functional style, avoid classes, use descriptive names"
→ Single input → Steps 2-10
→ Scope: personal → Type: skill → author-skill
→ Creates ~/.claude/skills/functional-style/SKILL.md
```

**Batch of related skills (no runtime coordination):**
```
/rewrite-ccasset skill1.md skill2.md skill3.md skill4.md skill5.md
→ Batch detected (5 files) → Steps 1a-1i
→ batch-analyzer: 2 overlap → merge → 4 work items, no runtime coordination
→ Parallel complexity → all simple
→ Parallel scope → 3 personal, 1 project
→ User confirms project item
→ Parallel authoring → model selection → report
```

**Batch of agents forming a pipeline (runtime coordination):**
```
/rewrite-ccasset scanner.md reporter.md fixer.md
→ Batch detected (3 files) → Steps 1a-1i
→ batch-analyzer: pipeline detected, runtime coordination recommended
→ Parallel authoring → runtime orchestrator created → model selection → report
```

**Mixed (Cursor rules):**
```
/rewrite-ccasset [Cursor rules with style prefs + project config]
→ Single input → Step 2: complex → Step 2a: decomposer
→ 6 components, mixed scopes
→ Personal (3): written to ~/.claude/rules/ and ~/.claude/skills/
→ Project (3): user prompted → files at <project>/.claude/rules/ and .claude/skills/, references in CLAUDE.md
```
