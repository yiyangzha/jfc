# Just Furnish Context (JFC)

JFC is a proof-of-concept framework for autonomous high energy physics analysis.
It integrates autonomous analysis agents with literature-based knowledge retrieval
and multi-agent review, and is sufficient to plan, execute, and document a
credible HEP analysis from a short physics prompt.

The framework specification splits into three components:

- **Methodology** — a structured encoding of the typical particle physics analysis
  workflow, from planning and data exploration to statistical analysis and paper
  drafting, including tiered multi-agent review at every stage.
- **General agent behavior** — strict specifications for what each subagent
  receives and outputs, context management across phases, and an experiment log
  for human oversight and debugging.
- **Domain-specific conventions** — HEP-specific tool use, visualization
  standards, and analysis technique guidance that general models cannot reliably
  infer on their own.

For details see our paper:
> *AI Agents Can Already Autonomously Perform Experimental High Energy Physics*  
> E. A. Moreno, S. Bright-Thonney, A. Novak, D. Garcia, P. Harris

## Quick start

```bash
pixi run scaffold analyses/my_analysis --type measurement
cd analyses/my_analysis
# Edit .analysis_config → set data_dir=/path/to/data, add allow= lines
pixi install
claude/codex   # pass your physics prompt
```

## How it works

```
┌─────────────────────────────────────────────────────────────┐
│                     ORCHESTRATOR                             │
│  Never writes code. Holds: prompt, summaries, verdicts only  │
└─────┬───────────────────────────────────────────────────────┘
      │
      ▼
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ Phase 1  │──▶│ Phase 2  │──▶│ Phase 3  │──▶│ Phase 4a │──▶│ Phase 4b │──▶│ Phase 4c │──▶│ Phase 5  │
 │ Strategy │   │ Explore  │   │Processing│   │ Expected │   │  10% Val │   │Full Data │   │ Document │
 │ (4-bot)  │   │(self+plt)│   │ (1-bot)  │   │(4bot+bib)│   │(4bot+bib)│   │ (1-bot)  │   │ (5-bot)  │
 └──────────┘   └──────────┘   └──────────┘   └──────────┘   └────┬─────┘   └──────────┘   └──────────┘
                                                                   │
                                                             HUMAN GATE
```

Each phase runs the same loop:

```
  1. EXECUTE ── spawn executor subagent (enters plan mode first)
  2. REVIEW ─── spawn reviewer(s) per review type
  3. CHECK:
       Regression trigger? → Investigator → fix origin + downstream → resume
       A or B items?       → fix agent + fresh reviewer → re-review (loop)
       Only C items?       → PASS, executor applies Cs before commit
  4. COMMIT
  5. HUMAN GATE (after 4b for both measurements and searches)
  6. ADVANCE
```

### Phases

| Phase | Review | Key deliverable |
|-------|--------|-----------------|
| **1. Strategy** | 4-bot | Technique selection, systematic plan, reference analysis table, conventions enumeration |
| **2. Exploration** | Self + plot validator | Sample inventory, data quality, variable ranking, preselection cutflow |
| **3. Processing** | 1-bot | Event selection, correction chain or background model, closure tests |
| **4a. Expected** | 4-bot+bib | Systematic completeness table, covariance matrix, reference comparisons |
| **4b. 10% Validation** | 4-bot+bib → human gate | 10% data results, draft AN with full structure |
| **4c. Full Data** | 1-bot | Full observed results, post-fit diagnostics |
| **5. Documentation** | 5-bot | Analysis note (pandoc markdown → PDF, 50-100 pages), machine-readable results |

Both measurements and searches follow the same 4a → 4b → 4c flow. The
human gate is between 4b and 4c.

### Review classification

| Cat | Meaning | Action |
|-----|---------|--------|
| **A** | Would cause rejection | Fix + re-review + fresh reviewer |
| **B** | Weakens the analysis | Same — must be zero before PASS |
| **C** | Style / clarity | Arbiter PASses; executor applies before commit |

Fresh reviewer added each iteration cycle. Limits: 4/5-bot warn at 3,
strong warn at 5, hard cap at 10. 1-bot warn at 2, escalate at 3.

### Phase regression

Any review can trigger regression when a physics issue is traceable to an
earlier phase. Most common after Phase 4a/4b and Phase 5 reviews.

```
Reviewer finds physics issue from Phase M < current Phase N
  → Investigator traces impact → REGRESSION_TICKET.md
  → Fix cycle: re-run Phase M, re-run affected downstream, skip unaffected
  → Resume review at Phase N
```

### Phase 5: 5-bot review

```
Physics + Critical + Constructive + Plot Validator + Rendering + BibTeX Validator (parallel) → Arbiter
```

The rendering reviewer runs `pixi run build-pdf` and uses the Read tool to
visually inspect the PDF for figure rendering, math compilation, layout, and
cross-references.

## Key concepts

**Technique decided at Phase 1, not scaffold time.** The scaffolder only
takes `--type measurement|search`. The strategy phase selects the technique
(unfolding, template fit, etc.), which activates technique-specific
requirements in later phases.

**Conventions.** Domain knowledge in `src/conventions/` (symlinked into each
analysis). Mandatory reads at Phases 1, 4a, and 5. Updated after analysis
completion.

**Feasibility evaluation.** When hitting a limitation (missing MC, etc.),
agents must: state it → evaluate feasibility → estimate cost → decide
(attempt if it affects the core result, document if minor or infeasible) →
log the reasoning.

**Pixi everywhere.** Each analysis has its own `pixi.toml` with deps and
tasks. `pixi run all` is the reproducibility contract. `pixi run build-pdf`
compiles the analysis note via pandoc.

## Directory structure

```
jfc/
  src/                        Framework infrastructure
    methodology/              Full spec: phases, review, orchestration, appendices
    conventions/              Domain knowledge (symlinked into analyses)
    templates/                CLAUDE.md and pixi.toml templates
    scaffold_analysis.py      Scaffolder
  analyses/                   Each is its own git repo
    <name>/
      AGENTS.md               Self-contained instructions for the orchestrator
      CLAUDE.md               Self-contained instructions for the orchestrator
      pixi.toml               Environment + task graph
      .analysis_config        data_dir + allow paths
      conventions/ → src/conventions/
      phase{1..5}_*/          Phase dirs with CLAUDE.md, outputs/, src/, review/, logs/
```

## How scaffolding works

The scaffolder (`pixi run scaffold`) creates a new analysis directory from
templates in `src/templates/`:

1. **Template files** (`src/templates/root_claude.md`, `phase*_claude.md`,
   `pixi.toml`) are copied into the analysis directory with `{{name}}` and
   `{{analysis_type}}` placeholders replaced.
2. **Phase directories** (`phase1_strategy/`, `phase2_exploration/`,
   `phase3_selection/`, `phase4_inference/`, `phase5_documentation/`) are
   created with `outputs/`, `outputs/figures/`, `src/`, `review/`, and `logs/` subdirs.
3. **Conventions symlink** — `conventions/` → `../../src/conventions/` is
   created so agents can read domain knowledge.
4. **`.analysis_config`** is created with `analysis_type` set. Edit it to
   add `data_dir=` pointing to the input data.
5. **Git repo** is initialized in the analysis directory.
6. **Methodology symlink** — `methodology/` → `../../src/methodology/` is
   created so agents can consult the full methodology spec.
7. **Agents symlink** — `agents/` → `../../src/agents/` is created so the
   orchestrator can read agent role definitions.

After scaffolding, the analysis directory is self-contained: its CLAUDE.md/AGENTS.md
files carry the essential instructions for execution.

## Requirements

- [pixi](https://pixi.sh) for environment management
- [Claude Code](https://claude.ai/claude-code) as the agent runtime
