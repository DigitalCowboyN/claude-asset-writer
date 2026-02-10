# Claude Asset Writer Architecture

## Overview

The Claude Asset Writer is a system for authoring Claude Code assets (rules, agents, skills, commands) according to established standards. It handles complexity detection, scope classification (personal vs project), decomposition of large inputs, type and weight classification, model selection, batch processing with parallel execution, and user confirmation flows.

## System Components

```mermaid
flowchart TB
    subgraph Orchestrator
        rewrite["rewrite-ccasset\n(commands/rewrite-ccasset.md)"]
    end

    subgraph Analysis["Analysis Agents"]
        batch["batch-analyzer\nmodel: opus"]
        complexity["complexity-detector\nmodel: sonnet"]
        scope["scope-detector\nmodel: haiku"]
        model["model-selector\nmodel: haiku"]
    end

    subgraph Lightweight["Authoring Agents (Lightweight)"]
        arule["author-rule\nmodel: sonnet"]
        aagent["author-agent\nmodel: sonnet"]
        askill["author-skill\nmodel: sonnet"]
        acmd["author-command\nmodel: sonnet"]
    end

    subgraph Heavyweight["Authoring Agents (Heavyweight)"]
        aorch["author-orchestrator\nmodel: sonnet"]
        aheavy["author-heavyweight-agent\nmodel: sonnet"]
    end

    subgraph Decomposition
        decomp["asset-decomposer\nmodel: opus"]
    end

    rewrite --> batch
    rewrite --> complexity
    rewrite --> scope
    rewrite --> model
    rewrite --> Lightweight
    rewrite --> Heavyweight
    rewrite --> decomp
```

---

## Data Flow

### Input Detection

```mermaid
flowchart TD
    input["User Input"]
    entry["/rewrite-ccasset"]
    detect{"Single or Batch?"}

    input --> entry --> detect

    detect -->|"Single file/content"| single["Single-Item Flow\n(Steps 2-10)"]
    detect -->|"Multiple files/glob/sections"| batch["Batch Flow\n(Steps 1a-1h)"]
```

### Batch Pipeline

```mermaid
flowchart TD
    parse["Step 1a: Parse Inputs\n(read all files)"]
    ba["Step 1b: batch-analyzer\n(relationships, merges, phases)"]
    b1[/"BARRIER B1"/]

    cd["Step 1c: complexity-detector\n(x N in parallel)"]
    b2[/"BARRIER B2"/]
    decomp["asset-decomposer\n(complex items only, parallel)"]

    sd["Step 1d: scope-detector\n(x N in parallel)"]
    b3[/"BARRIER B3"/]

    scope_prompt["Step 1e: User Scope Decisions\n(batched prompts)"]
    b4[/"BARRIER B4"/]

    author["Step 1f: Phased Authoring\n(parallel per phase, sequential between phases)"]
    b5[/"BARRIER B5"/]

    runtime{"Step 1g: Runtime Orchestrator?\n(batch-analyzer recommended?)"}
    create_orch["author-orchestrator\n(if user confirms)"]

    ms["Step 1h: model-selector\n(x N in parallel)"]
    model_prompt["User Confirms Models"]
    b6[/"BARRIER B6"/]

    claude_md["Step 1i: CLAUDE.md Writes\n(serialized, one at a time)"]
    b7[/"BARRIER B7"/]
    report["Final Report"]

    parse --> ba --> b1 --> cd --> b2
    b2 -->|"complex items"| decomp --> sd
    b2 -->|"simple items"| sd
    sd --> b3 --> scope_prompt --> b4 --> author --> b5 --> runtime
    runtime -->|"yes"| create_orch --> ms
    runtime -->|"no"| ms
    ms --> model_prompt --> b6 --> claude_md --> b7 --> report
```

### Single-Item Full Pipeline

```mermaid
flowchart TD
    input["User Input\n(file path or content)"]
    entry["/rewrite-ccasset"]
    cd["complexity-detector"]

    input --> entry --> cd

    cd -->|"COMPLEXITY: simple"| simple_path
    cd -->|"COMPLEXITY: complex"| complex_path

    subgraph simple_path["Phase 2b: Single Asset Flow"]
        direction TB
        sd["scope-detector"]
        sd -->|personal| classify["Determine Type\n& Weight Class"]
        sd -->|project| scope_prompt["User Prompt:\nInclude in project?"]
        sd -->|ambiguous| scope_prompt2["User Prompt:\nPersonal or project?"]
        scope_prompt -->|include| classify
        scope_prompt -->|skip| skipped["Skipped"]
        scope_prompt -->|make personal| classify
        scope_prompt2 -->|personal| classify
        scope_prompt2 -->|project| classify
        classify -->|rule| ar["author-rule"]
        classify -->|agent + lightweight| aa["author-agent"]
        classify -->|skill| as["author-skill"]
        classify -->|command + lightweight| ac["author-command"]
        classify -->|agent + heavyweight| ahv["author-heavyweight-agent"]
        classify -->|command + heavyweight| ao["author-orchestrator"]

        aa --> ms1["model-selector"]
        ahv --> ms2["model-selector"]
        ao --> ms3["model-selector"]

        ms1 --> uc1["User Confirm Model"]
        ms2 --> uc2["User Confirm Model"]
        ms3 --> uc3["User Confirm Model"]

        uc1 --> apply1["Apply Model"]
        uc2 --> apply2["Apply Model"]
        uc3 --> apply3["Apply Model"]

        ar --> report1["Report"]
        as --> report1
        ac --> report1
        apply1 --> report1
        apply2 --> report1
        apply3 --> report1
    end

    subgraph complex_path["Phase 2a: Decomposition Flow"]
        direction TB
        decompose["asset-decomposer"]
        decompose --> scope_each["scope-detector\n(per component)"]
        scope_each -->|personal| authors["author-agent / author-skill\nauthor-rule / author-command"]
        scope_each -->|"project (user confirms)"| proj_dest["CLAUDE.md sections\nor project .claude/ files"]
        decompose -->|"if needed"| orch["author-orchestrator"]

        authors --> ms4["model-selector (x N)"]
        orch --> ms5["model-selector"]

        ms4 --> confirm["User Confirm\n(all models at once)"]
        ms5 --> confirm

        confirm --> report2["Report\n(all assets)"]
    end
```

---

## Classification Logic

### Asset Type Detection

```mermaid
flowchart LR
    content["Input Content"]
    content -->|"Language/framework-specific\n(Go, Python, React, etc.)"| skill["skill"]
    content -->|"Workflow sequence\n(do X, then Y, then Z)"| command["command"]
    content -->|"Focused expertise\nfor dispatch"| agent["agent"]
    content -->|"Universal behavior\n(every session, every project)"| rule["rule"]
```

### Weight Class Detection

```mermaid
flowchart TD
    input["Typed Asset"]

    input --> hw_check{"Heavyweight indicators?\n- Contains 'Task tool' / 'dispatch' / 'subagent'\n- Orchestrates multiple phases\n- 'coordinates' or 'orchestrates'"}

    hw_check -->|Yes| heavy["Heavyweight"]
    hw_check -->|No| light["Lightweight"]

    heavy -->|agent| aheavy["author-heavyweight-agent"]
    heavy -->|command| aorch["author-orchestrator"]
    light -->|rule| arule["author-rule"]
    light -->|skill| askill["author-skill"]
    light -->|command| acmd["author-command"]
    light -->|agent| aagent["author-agent"]
```

### Scope Detection

```mermaid
flowchart TD
    input["Input Content"]
    input --> scan{"Scan for Signals"}

    scan -->|"Coding style preferences\nNaming conventions\nLanguage idioms\nPattern preferences"| personal["SCOPE: personal\n-> ~/.claude/"]

    scan -->|"Specific directories\nNamed tools/dependencies\nArchitectural decisions\nDeployment/infra specifics"| project["SCOPE: project\n-> project CLAUDE.md\n(user confirms)"]

    scan -->|"Tech-specific principles\nCould be style OR requirement\nTeam conventions"| ambiguous["SCOPE: ambiguous\n(user decides)"]
```

#### Project Destination Routing

| Asset Type | Project Destination | Format |
|---|---|---|
| rule | `<project>/CLAUDE.md` | Inline section |
| skill | `<project>/CLAUDE.md` | Inline section |
| agent | `<project>/.claude/agents/` | Separate file |
| command | `<project>/.claude/commands/` | Separate file |

### Complexity Detection

```mermaid
flowchart TD
    input["Input Content"]
    input --> analysis{"Analyze Content"}

    analysis -->|"Line count > 200\nMultiple domains\nPhase patterns\nMultiple outputs\nConditional branches"| complex["COMPLEX\n-> asset-decomposer"]

    analysis -->|"Line count < 150\nFocused scope\nSingle domain\nLinear flow"| simple["SIMPLE\n-> single authoring agent"]
```

---

## Model Selection Logic

The `model-selector` agent analyzes written agent files and recommends a model:

```mermaid
flowchart TD
    agent_file["Agent File"]
    agent_file --> analyze{"Analyze Signals"}

    analyze -->|"Tools include Task\nMulti-phase coordination\nComplex reasoning\nSecurity/architecture review"| opus["opus"]

    analyze -->|"Tools include Write, Edit, Bash\nStandard code implementation\n5-10 step procedures"| sonnet["sonnet"]

    analyze -->|"Read-only tools (Read, Glob, Grep)\nSimple data collection/formatting\n< 5 step procedures"| haiku["haiku"]
```

---

## File Size Guidelines

| Asset Type | Target Lines |
|---|---|
| Rules | < 50 |
| Lightweight agents | ~80 |
| Heavyweight agents | 100-150 |
| Skills | ~100 (main SKILL.md) |
| Lightweight commands | ~80 |
| Orchestrator commands | 100-150 |

---

## User Interaction Points

```mermaid
flowchart LR
    subgraph Confirmation Points
        direction TB
        p0["1. Scope Decision\nProject content detected\n-> user confirms: include, skip, or make personal"]
        p1["2. Model Selection\nmodel-selector recommends\n-> user accepts or overrides"]
        p2["3. Decomposition Models\nall model selections at once\n-> user confirms batch"]
        p3["4. Execution Order\nRecommended runtime order\n-> user confirms or corrects"]
        p4["5. Runtime Orchestrator\nBatch items form a workflow\n-> user confirms or corrects order"]
        p5["6. Registration Prompts\nRegister in orchestrators?\nReference in CLAUDE.md?"]
    end
```

**Note:** Personal content flows through without prompting. Only project-scoped and ambiguous content triggers scope confirmation.

---

## Batch Processing

### Synchronization Barriers

The batch flow uses 7 explicit barriers to ensure correct execution order:

| Barrier | After | Before | Purpose |
|---|---|---|---|
| B1 | Batch analysis | Complexity detection | Need merge/phase plan and runtime coordination assessment |
| B2 | All complexity detections | Scope detection | Need to know which items expand |
| B3 | All scope detections | User scope prompts | Need all scopes to batch prompts |
| B4 | User scope decisions | Authoring | Need to know what to write and where |
| B5 | All authoring (all phases) | Runtime orchestrator | Need written files before coordinating them |
| B6 | All model selections | CLAUDE.md writes | Need user model confirmation first |
| B7 | All CLAUDE.md writes | Final report | Everything must be written |

### Parallel Execution Model

The Task tool blocks until the subagent returns. "Parallel" means dispatching multiple Task calls in a single orchestrator message — Claude Code processes these concurrently. The orchestrator waits for all to return before proceeding past a barrier.

### CLAUDE.md Write Serialization

Multiple project-scoped writes to the same CLAUDE.md file are always serialized (one at a time) to prevent race conditions. Each write reads the current file state, checks for conflicts, and appends.

### Error Handling

When a parallel dispatch fails: retry once. If retry fails: mark as failed, continue with remaining items. One failed item never blocks the batch. All failures are reported in the final report.

---

## Execution Order Analysis

### Where It Happens

Execution order analysis occurs in two places:

| Context | Agent | When | Purpose |
|---|---|---|---|
| Decomposition | `asset-decomposer` (Step 5b) | After agents created, before orchestrator authored | Determine runtime order for the orchestrator's dispatch flow |
| Batch processing | `batch-analyzer` (Step 5) | During initial batch analysis | Detect if batch items form a runtime workflow needing coordination |

### Data Flow Signals

| Signal | Meaning |
|---|---|
| B's purpose references A's output | A before B (sequential dependency) |
| A and B both analyze the same input independently | A and B can run in parallel |
| C consolidates or reports on A and B | C runs after both (fan-in) |
| D only runs if A finds something specific | D conditional on A |

### Output

Both contexts produce a phased execution order:

```
Phase 1 (parallel): [agent1], [agent2]
Phase 2: [agent3] (needs output from phase 1)
Conditional: [agent4] (only if agent1 finds issues)
```

This order is confirmed by the user before being passed to `author-orchestrator`.

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

### Adding New Scope Signals

1. Update `scope-detector.md` signal taxonomy with new personal/project/ambiguous indicators

### Adding New Project Destinations

1. Update `rewrite-ccasset.md` Step 5a for new inline formatting patterns
2. Update `asset-decomposer.md` Step 4 for new project file routing
