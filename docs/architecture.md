# Claude Asset Writer Architecture

## Overview

The Claude Asset Writer is a system for authoring Claude Code assets (rules, agents, skills, commands) according to established standards. It handles complexity detection, scope classification (personal vs project), decomposition of large inputs, type and weight classification, model selection, and user confirmation flows.

## System Components

```mermaid
flowchart TB
    subgraph Orchestrator
        rewrite["rewrite-ccasset\n(commands/rewrite-ccasset.md)"]
    end

    subgraph Detection["Detection Agents"]
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

    rewrite --> complexity
    rewrite --> scope
    rewrite --> model
    rewrite --> Lightweight
    rewrite --> Heavyweight
    rewrite --> decomp
```

---

## Data Flow

### Full Pipeline

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
        p3["4. Registration Prompts\nRegister in orchestrators?\nReference in CLAUDE.md?"]
    end
```

**Note:** Personal content flows through without prompting. Only project-scoped and ambiguous content triggers scope confirmation.

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
