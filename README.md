# autoimprove

Autonomous optimization loop for AI agents. Describe what to improve, how to measure it, and walk away.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch) and [Shopify Liquid PR #2056](https://github.com/Shopify/liquid/pull/2056).

## How it works

```
You write improve.md     →  Agent reads it
                         →  Generates safety tests (bootstrap)
                         →  Establishes baseline score
                         →  LOOP:
                              Propose change → git commit → run tests
                              → tests pass? → run benchmark → score improved?
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

Once confirmed, the boundary is locked. You can also use explicit paths or globs:

```markdown
## Change
scope: src/handlers/**/*.go
```

### Check — how to measure

Tests and scoring are separated. Tests gate correctness, the score measures improvement:

```markdown
## Check
test: go test ./...                    # must pass — hard gate
test-files: test/                      # immutable during loop
run: go test -bench=. -benchmem       # produces the score
score: ns/op:\s+([\d.]+)              # how to extract the number
goal: lower                            # or higher
timeout: 3m
```

Score extraction supports three formats:
- Convention: `SCORE: 42.5` in stdout
- Regex: `ns/op:\s+([\d.]+)` with a capture group
- jq: `.results.mean_time` for JSON output

### Stop — when to quit

```markdown
## Stop
budget: 4h       # wall-clock time
rounds: 100      # max experiments
target: 500      # stop when score reaches this
stale: 15        # stop after 15 consecutive failures
```

Whichever condition hits first.

### Agent — for headless mode

```markdown
## Agent
provider: claude
model: sonnet
```

### Instructions — domain guidance

Everything after `## Instructions` is free-form. Tell the agent what to try, what to avoid, and any domain knowledge that helps:

```markdown
## Instructions

Focus on reducing object allocations — GC is 74% of CPU time.

Patterns to try:
- Fast-path with fallback for common cases
- Byte-level parsing instead of regex
- Cache small repeated allocations

What NOT to try:
- Don't change the public API
- Don't add dependencies
```

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

Each experiment records: what was tried, why, what changed, the score, and whether it was kept or discarded. The agent reads its own history to avoid repeating failed ideas and build on what worked.

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
