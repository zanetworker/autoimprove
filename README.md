# autoimprove

Autonomous optimization loop for AI agents. Describe what to improve, how to measure it, and walk away.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and [Shopify Liquid PR #2056](https://github.com/Shopify/liquid/pull/2056).

## How it works

```
You write improve.md     →  Agent reads it
                         →  Scaffolds eval if needed (eval-init)
                         →  Generates safety tests (bootstrap)
                         →  Establishes baseline score
                         →  LOOP:
                              Propose change → git commit (verified)
                              → run tests → pass? → run benchmark
                              → check guards → score improved?
                              → yes: keep → no: git reset
                              → log experiment → repeat
                         →  Print summary
```

No human in the loop. The agent decides what to try based on your instructions and its own experiment history.

## Quick start

1. Create an `improve.md` in your project:

```markdown
# autoimprove: make-it-faster

## Change
scope: the checkout handler and its database queries

## Check
test: go test ./...
run: go test -bench=. -benchmem
score: ns/op:\s+([\d.]+)
goal: lower
guard: allocs/op: ([\d.]+) < 500
timeout: 3m

## Stop
budget: 4h
stale: 15

## Instructions

Reduce allocations and avoid unnecessary work in hot paths.
Try buffer reuse, fast-path patterns, and byte-level operations.
Don't change the public API.
```

2. Run it:

```bash
# Inside Claude Code
/autoimprove

# Headless (overnight)
claude -p "run /autoimprove on improve.md" --allowedTools bash,read,write,edit
```

## The `improve.md` format

A single markdown file — part config, part prompt. You describe what you want in natural language, and the agent handles the rest.

### Change — what to optimize

Describe the scope naturally. The agent resolves it to specific files and confirms before starting:

```markdown
## Change
scope: the template parsing engine
exclude: test/, benchmark/
```

The agent will scan the codebase and propose:

```
Resolved scope "the template parsing engine" to:
  - lib/liquid/parser.rb
  - lib/liquid/lexer.rb
  - lib/liquid/variable.rb

These are the ONLY files that will be modified. Confirm? [y/n]
```

Once confirmed, the boundary is locked. `exclude` prevents the agent from touching files it should never modify — especially evaluation code. Without `exclude: eval/`, an agent could "improve" the score by modifying how it's measured rather than improving the actual code. The agent should never grade its own homework.

You can also use explicit paths or globs:

```markdown
## Change
scope: src/handlers/**/*.go
```

### Check — how to measure

Tests, scoring, and guards are separated. Tests gate correctness, the score measures improvement, guards prevent regressions on secondary metrics:

```markdown
## Check
test: go test ./...                    # must pass — hard gate
test-files: test/                      # immutable during loop
run: go test -bench=. -benchmem       # produces the score
score: ns/op:\s+([\d.]+)              # how to extract the number
goal: lower                            # or higher
guard: allocs/op: ([\d.]+) < 500       # secondary metric that must not regress
keep_if_equal: true                    # keep bug fixes and simplifications
timeout: 3m
```

Score extraction supports three formats:
- Convention: `SCORE: 42.5` in stdout
- Regex: `ns/op:\s+([\d.]+)` with a capture group
- jq: `.results.mean_time` for JSON output

**Guard metrics** protect against improving one metric by tanking another. If the guard threshold is violated, the experiment is discarded regardless of the primary score.

**keep_if_equal** (default: false) keeps changes that don't regress, even if they don't improve the score. Useful for bug fixes, code simplifications, and UX improvements that don't affect the measured metric.

### Stop — when to quit

```markdown
## Stop
budget: 4h       # wall-clock time (starts at first experiment, not during setup)
rounds: 100      # max experiments
target: 500      # stop when score reaches this
stale: 15        # stop after 15 consecutive failures
```

Whichever condition hits first. Budget time counts from the first experiment, not from bootstrap or eval setup.

### Agent — for headless mode

```markdown
## Agent
provider: claude
model: sonnet
```

### Instructions — domain guidance

Everything after `## Instructions` is free-form. Tell the agent what to try, what to avoid, and any domain knowledge that helps.

## Bootstrap — test generation

Before running the optimization loop, bootstrap generates a test suite tailored to your optimization goal:

```bash
/autoimprove bootstrap           # analyze and suggest tests
/autoimprove bootstrap --generate  # create the test files
```

The key insight: **the optimization goal predicts what the agent will break.**

- Optimizing for **speed**? The agent will skip edge cases and remove safety checks. Bootstrap generates tests for unicode, nil handling, error messages, and concurrency.
- Optimizing for **smaller size**? The agent will remove things. Bootstrap tests that features still work and runtime deps are present.
- Optimizing for **ML accuracy**? The agent will overfit. Bootstrap tests for data leakage, reproducibility, and valid predictions.
- Optimizing for **RAG quality**? The agent will game retrieval or over-stuff context. Bootstrap tests for answer format consistency, hallucination on out-of-scope questions, and handling of empty retrieval results.

Tests are mutable during bootstrap, **immutable during the loop**. Two phases, never mixed.

## Eval init — for domains that need a golden set

Some domains have objective metrics (bytes, seconds, pod count). Others need human judgment about what a good result looks like (search relevance, answer quality, prediction accuracy). For the second group, use eval-init:

```bash
/autoimprove eval-init
```

This scaffolds an eval script and golden set by running the system with sample inputs, asking you to label the results, and building a test set from your judgments. Needed for: RAG, prompt engineering, AutoML. Not needed for: perf, Docker, frontend, CI, SQL, K8s.

## Templates

Scaffold an `improve.md` for your domain:

```bash
/autoimprove init                # auto-detect from repo
/autoimprove init --type perf    # code performance
/autoimprove init --type ml      # ML training
/autoimprove init --type automl  # tabular ML (churn, fraud, scoring)
/autoimprove init --type docker  # container image size
/autoimprove init --type k8s     # Kubernetes health
/autoimprove init --type prompt  # LLM prompt quality
/autoimprove init --type sql     # SQL query performance
/autoimprove init --type frontend  # bundle size
/autoimprove init --type ci      # build/CI speed
/autoimprove init --type rag     # RAG pipeline (chunking, retrieval, generation)
```

## Agent-agnostic

The skill works inside Claude Code, but the format works with any agent. Export a standalone protocol:

```bash
/autoimprove --export    # generates program.md
```

Then point any agent at it:

```bash
codex -p "follow program.md"
gemini -p "follow program.md"
aider --message "follow program.md"
```

## Experiment log

Every experiment is logged to `.autoimprove/experiments/` as structured JSON:

```
.autoimprove/
  baseline.json
  experiments/
    001-reduce-allocations.json
    002-cache-integer-tostring.json
```

Each experiment records: what was tried, why, what changed, the score, whether it was kept or discarded, and which earlier experiments it supersedes (so the agent doesn't retry obsolete approaches).

```bash
# View results outside Claude Code
ls .autoimprove/experiments/*.json | xargs jq '{id, title, status, score, delta_pct}'
```

## How is this different from AutoML?

AutoML searches a predefined parameter grid. Autoimprove lets an AI agent make creative, open-ended changes — rewrite algorithms, delete code, try novel approaches. The search space is unbounded.

| | AutoML | Autoimprove |
|---|---|---|
| Search space | Predefined grid | Open-ended |
| Changes | Numeric knobs | Creative — rewrite code, try new algorithms |
| Strategy | Bayesian optimization | AI reasoning about what might work |
| Ceiling | Best from your grid | Unbounded discovery |

## License

MIT
