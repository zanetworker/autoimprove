---
name: autoimprove
description: "Autonomous optimization loop that improves any measurable thing. Point it at files to change, a command to check, and a number to improve — then walk away. Works with any AI agent. Use when the user wants to autonomously optimize code performance, ML training, Docker images, SQL queries, prompts, CI speed, bundle size, Kubernetes configs, or any domain with a measurable score. Triggers include requests like 'optimize this', 'improve performance', 'make this faster', 'reduce allocations', 'autoimprove', 'run the optimization loop', 'let it run overnight', or when the user has an improve.md file."
---

# Autoimprove

Autonomous optimization loop. You modify files, run a check, keep improvements, discard regressions, and repeat — no human in the loop.

## When to Use

- User wants to optimize something measurable (speed, size, accuracy, cost)
- User has an `improve.md` file
- User says "autoimprove", "optimize", "improve", "make faster/smaller/better"
- User wants overnight autonomous optimization

## Commands

| Command | What it does |
|---------|-------------|
| `/autoimprove` | Run the loop on `improve.md` in cwd |
| `/autoimprove <path>` | Run the loop on a specific improve.md |
| `/autoimprove bootstrap` | Analyze codebase, generate tests needed for safe optimization |
| `/autoimprove bootstrap --generate` | Create the test files (review before committing) |
| `/autoimprove init` | Scaffold an improve.md (auto-detects repo type) |
| `/autoimprove init --type <type>` | Scaffold for a specific domain |
| `/autoimprove --export` | Generate agent-agnostic `program.md` |

## The `improve.md` Format

A single markdown file — part config, part prompt:

```markdown
# autoimprove: <name>

## Change
scope: <natural language description, explicit paths, or globs>
exclude: <paths to never modify (optional)>

## Check
test: <command that verifies correctness — must pass for any experiment to be kept>
test-files: <paths to test files — read-only during the loop, mutable only during bootstrap>
run: <command that produces the score — only runs if test passes>
score: <how to extract the number — "SCORE: {value}" or regex or jq>
goal: <lower | higher>
timeout: <max time per experiment>

## Stop
budget: <total wall-clock limit>
rounds: <max experiments>
target: <stop when score reaches this>
stale: <stop after N failures in a row>

## Agent
provider: <claude | codex | gemini | any>
model: <model to use>

## Instructions

<free-form: what to try, what to avoid, domain knowledge>
```

## The Loop

When invoked, follow this protocol exactly:

### Step 1: Parse

Read the `improve.md` file. Extract all structured fields from the headers. Everything after `## Instructions` is the domain prompt.

### Step 2: Validate and Resolve Scope

1. **Resolve scope**: Read the `Change.scope` field.
   - If it contains explicit paths or globs (e.g., `src/handlers/*.go`): resolve directly
   - If it contains natural language (e.g., "the template parsing engine"): scan the codebase, identify matching files, and present them for confirmation:
     ```
     Resolved scope "the template parsing engine" to:
       - lib/liquid/parser.rb
       - lib/liquid/lexer.rb
       - lib/liquid/variable.rb

     These are the ONLY files that will be modified. Confirm? [y/n]
     ```
   - Apply `Change.exclude` to filter out any paths that should not be touched
   - Once confirmed, the resolved file list is LOCKED for the entire loop

2. **Validate**:
   - All resolved files exist
   - The `Check.run` command is executable
   - Git working tree is clean (no uncommitted changes)
   - If `Check.test` is specified, run it — tests must pass before the loop starts

If validation fails, report the issue and stop.

If tests don't exist or fail, suggest running `/autoimprove bootstrap` first.

### Step 2.5: Bootstrap (when invoked via `/autoimprove bootstrap`)

This is a separate pre-loop phase. It generates the test suite needed for safe optimization.

**Tests are mutable during bootstrap, immutable during the loop.** These two phases never mix.

#### Bootstrap Protocol

1. **Analyze**: Read all files in the resolved `Change.scope` AND the `improve.md`. Identify:
   - The optimization goal (`Check.goal`) and what the agent will be chasing
   - Public API surface (exported functions, methods, classes)
   - Critical code paths (hot loops, core logic, data transformations)
   - Edge cases (nil/null handling, empty inputs, boundary values)
   - Existing test coverage (check `Check.test-files` if specified)

2. **Goal-aware threat modeling**: The optimization goal predicts what the agent will try, which predicts what it will break. Generate tests that guard against the failure modes of THAT specific goal:

   **When goal = faster (lower latency, fewer allocations):**
   The agent will skip work, take shortcuts, and remove safety checks.
   - Unicode/multibyte input still works (fast paths assume ASCII)
   - Empty, nil, zero-length inputs don't crash (nil checks removed for speed)
   - Error messages are still correct and informative (error formatting skipped)
   - Concurrent access is safe (locks removed for throughput)
   - Large inputs don't OOM or hang (buffer reuse may break on edge sizes)
   - Output is bit-for-bit identical to baseline (fast path might truncate or round)

   **When goal = smaller (image size, bundle size):**
   The agent will remove things, strip features, and swap dependencies.
   - All features still work at runtime (removed code might have been needed)
   - Lazy-loaded routes/components still render (code splitting may break references)
   - Runtime dependencies are present (dev deps removed, but some were runtime)
   - Static assets still load (paths may change with restructuring)
   - App starts successfully and passes health checks

   **When goal = higher accuracy (ML, prompt quality):**
   The agent will overfit, leak data, and add complexity.
   - No test/validation data leaks into training (train/test split integrity)
   - Pipeline is reproducible with fixed seeds (randomness controlled)
   - Missing values, outliers, and edge inputs handled correctly
   - Model outputs are valid (probabilities sum to 1, no NaN predictions)
   - Feature engineering doesn't introduce future data leakage (no lookahead)
   - Predictions work on single samples, not just batches

   **When goal = lower cost (infra, compute):**
   The agent will downsize, reduce redundancy, and cut resources.
   - Service still handles expected load (reduced replicas may not suffice)
   - Failover still works (redundancy removed)
   - Latency stays within acceptable bounds (smaller instances = slower)
   - Health checks pass under load, not just at idle
   - Data durability unchanged (storage reductions may lose backups)

   **When goal = higher score (prompt engineering, config tuning):**
   The agent will game the metric and overfit to the eval set.
   - Output format is consistent (not just correct for eval cases)
   - Edge inputs produce reasonable output (not just golden set inputs)
   - Output length/verbosity stays reasonable (gaming token limits)
   - No hardcoded responses for known eval cases

3. **Gap analysis**: Cross-reference existing tests with the goal-aware threat model:
   - **Critical gaps**: Failure modes with NO test coverage — these must be filled
   - **Weak coverage**: Tests exist but don't cover edge cases for this goal
   - **Sufficient**: Already tested well for this optimization direction

4. **Generate** (when `--generate` is passed):
   - Write test files to the paths specified in `Check.test-files`
   - Prioritize tests for critical gaps first, weak coverage second
   - Run the tests to confirm they pass on the current unmodified code
   - If any test fails, fix it — tests must pass on the CURRENT code before optimization
   - Present the generated tests for review, grouped by threat category

5. **Commit**: After human review, commit the test files:
   `git commit -m "autoimprove bootstrap: add goal-aware test suite for <goal> optimization"`

6. **Report**: Print a readiness summary:
   ```
   Bootstrap complete.
     Goal: faster (lower is better)
     Threat model: shortcut breakage, edge case skipping, safety check removal
     Test files: test/variable_test.rb, test/parser_test.rb
     Tests: 47 passing
       Output equivalence: 12
       Edge cases (unicode, nil, empty): 15
       Error handling: 8
       Concurrency safety: 5
       API contracts: 7
     Ready for: /autoimprove
   ```

#### What makes a good test suite for autoimprove

The tests don't need to be exhaustive — they need to catch the specific kinds of breakage
that an agent chasing YOUR optimization goal is likely to introduce.

The key insight: **the goal tells you what the agent will try, and that tells you what to test.**

Tests SHOULD:
- Guard against the failure modes predicted by the goal (see threat model above)
- Verify output equivalence: same input produces same output regardless of internals
- Cover edge cases the agent's fast paths will skip: empty, nil, unicode, huge, concurrent
- Assert on API contracts: signatures, return types, side effects
- Be fast — slow tests make the loop slower and waste experiment budget

Tests should NOT:
- Assert on performance (that's what the score is for)
- Assert on internal implementation details (the agent SHOULD change those)
- Be flaky or timing-dependent (false failures poison the loop)
- Be so numerous they dominate experiment time (keep test suite under 30s)

### Step 3: Baseline

Run the check command. Extract the score. Save to `.autoimprove/baseline.json`:

```json
{
  "score": 1.230,
  "commit": "abc1234",
  "timestamp": "2026-03-14T10:00:00Z"
}
```

Print: `Baseline established: <score>`

### Step 4: Loop

```
current_baseline = baseline score
consecutive_failures = 0
experiment_id = 1

WHILE stopping conditions not met:
  1. Read past experiments from .autoimprove/experiments/
  2. Propose a change to the target files
     - Use the Instructions section for domain guidance
     - Review past experiments to avoid repeating failures
     - Review past kept experiments to build on what worked
  3. Stage and commit: git commit -m "autoimprove: <short title>"
  4. Run the test command (if Check.test is specified)
  5. IF tests fail:
       → git reset --hard HEAD~1
       → Log as "test_failed"
       → consecutive_failures += 1
       → Print: "✗ Experiment {id}: {title} — tests failed"
       → CONTINUE to next iteration
  6. Run the score command (Check.run) with timeout
  7. Extract the score

  8. DECIDE:
     IF score command failed (non-zero exit, timeout):
       → git reset --hard HEAD~1
       → Log as "error"
       → consecutive_failures += 1

     ELIF score improved (lower if goal=lower, higher if goal=higher):
       → Log as "kept"
       → current_baseline = new score
       → consecutive_failures = 0
       → Print: "✓ Experiment {id}: {title} — {score} ({delta_pct}%)"

     ELSE:
       → git reset --hard HEAD~1
       → Log as "discarded"
       → consecutive_failures += 1
       → Print: "✗ Experiment {id}: {title} — {score} (no improvement)"

  7. Save experiment to .autoimprove/experiments/{NNN}-{slug}.json
  8. experiment_id += 1
  9. Check stopping conditions
```

### Step 5: Summary

When the loop ends, print:

```
Autoimprove complete.
  Experiments: 47
  Kept: 12
  Discarded: 25
  Test failures: 6
  Errors: 4
  Baseline: 1.230 → Best: 0.847 (-31.1%)
  Duration: 3h 42m
```

## Stopping Conditions

The loop stops when ANY of these is true:
- `budget` time has elapsed
- `rounds` experiments have been run
- Score has reached `target`
- `stale` consecutive experiments failed to improve
- Manual interrupt (Ctrl+C or agent termination)

## Score Extraction

Try these in order:
1. **Convention**: look for `SCORE: <number>` in stdout
2. **Regex**: if `score:` field contains a regex pattern (has parentheses), apply it to stdout
3. **jq**: if `score:` field starts with `.`, treat as jq expression applied to stdout as JSON

## Experiment Logging

Each experiment is saved as JSON in `.autoimprove/experiments/`:

```json
{
  "id": 1,
  "title": "reduce allocations in variable parsing",
  "status": "kept | discarded | test_failed | error",
  "commit": "a1b2c3d",
  "score": 0.847,
  "baseline_score": 1.230,
  "delta_pct": -31.1,
  "duration_s": 287,
  "reasoning": "Variable parsing allocates a new array per filter. Switching to byte-level scanning avoids the allocation entirely.",
  "files_changed": ["lib/liquid/variable.rb"],
  "timestamp": "2026-03-14T10:23:00Z"
}
```

## Rules

<HARD-RULES>
- NEVER modify files outside the resolved `Change.scope`
- NEVER modify test files during the optimization loop — tests are immutable guardrails
- NEVER modify the check command or scoring mechanism
- NEVER skip the git commit before running the check
- ALWAYS log every experiment, including failures and errors
- ALWAYS read past experiments before proposing a new change
- ALWAYS git reset discarded experiments — leave the tree clean
- Test failure means immediate discard — no exceptions, no "but the score improved"
- If many experiments fail tests, the change strategy is wrong — try a different approach
- Complexity must pay for itself: 20 lines of hack for 0.001 improvement is NOT worth keeping
- Deleting code for equal results IS worth keeping
</HARD-RULES>

## Init Templates

When the user runs `/autoimprove init`, detect the repo type and suggest the right template. If `--type` is specified, use that directly.

Detection heuristics:
- `go.mod` or `Cargo.toml` or `Makefile` with benchmark targets → `perf`
- `train.py` or `*.ipynb` with torch/tensorflow imports → `ml`
- `train.py` with sklearn/xgboost/lightgbm/catboost imports → `automl`
- Python files with langchain/llama_index/chromadb/pinecone/weaviate/qdrant imports → `rag`
- `Dockerfile` → `docker`
- `*.yaml` with `kind: Deployment` → `k8s`
- `prompts/` directory or `eval/` with score outputs → `prompt`
- `*.sql` files → `sql`
- `package.json` with build script → `frontend`
- `.github/workflows/` → `ci`

See `references/examples.md` for all 8 templates.

## Export Mode

When `/autoimprove --export` is invoked, generate a `program.md` file that any agent can follow without the skill. This makes autoimprove agent-agnostic.

See `references/protocol.md` for the exported protocol template.

## Multi-Shot Examples

The following examples demonstrate how autoimprove applies across domains. Use these to understand the SHAPE of good experiments — what kinds of changes to try, how to structure the check command, and what good `improve.md` instructions look like.

### Example 1: API Latency

```markdown
# autoimprove: faster-checkout-api

## Change
scope: the checkout handler and its database queries
exclude: test/, vendor/

## Check
test: go test ./...
test-files: test/
run: hey -n 1000 http://localhost:8080/checkout
score: Requests/sec:\s+([\d.]+)
goal: higher
timeout: 3m

## Stop
budget: 4h
stale: 15

## Instructions

Focus on reducing latency in the checkout handler.

Patterns to try:
- Query batching — combine multiple DB round-trips into one
- Connection pooling — reuse DB connections
- Response caching — cache unchanged data
- Reduce serialization overhead — smaller structs, avoid reflection

What NOT to try:
- Don't change the API contract
- Don't add external dependencies
- Don't change the database schema
```

### Example 2: Docker Image Size

```markdown
# autoimprove: slim-container

## Change
scope: Dockerfile

## Check
run: docker build -t test . && echo SCORE: $(docker image inspect test --format '{{.Size}}')
goal: lower
timeout: 5m

## Stop
rounds: 30

## Instructions

Reduce the Docker image size without breaking functionality.

Patterns to try:
- Multi-stage builds
- Alpine or distroless base images
- Combine RUN layers to reduce intermediate layers
- Remove dev dependencies and build tools from final stage
- Use scratch as final stage for static binaries

What NOT to try:
- Don't remove runtime dependencies the app needs
- Don't use UPX compression (breaks debugging)
```

### Example 3: LLM Prompt Quality

```markdown
# autoimprove: better-extraction-prompt

## Change
scope: prompts/extract.txt

## Check
run: python eval/run_eval.py --prompt prompts/extract.txt
score: f1: ([\d.]+)
goal: higher
timeout: 2m

## Stop
budget: 1h
target: 0.95

## Instructions

Improve the entity extraction prompt's F1 score on the golden set.

Patterns to try:
- Add few-shot examples from the golden set
- Chain-of-thought reasoning before extraction
- Structured output format (JSON with specific keys)
- Negative examples showing what NOT to extract
- Simplify instructions — shorter prompts often perform better

What NOT to try:
- Don't change the evaluation script
- Don't overfit to specific golden set entries
- Keep the prompt under 2000 tokens
```

### Example 4: Build/CI Speed

```markdown
# autoimprove: faster-ci

## Change
scope: CI workflow and build configuration
exclude: src/

## Check
run: time npm run build 2>&1 | tail -1
score: real\s+(\d+\.\d+)
goal: lower
timeout: 10m

## Stop
stale: 15

## Instructions

Reduce CI build time without breaking the build output.

Patterns to try:
- Parallel jobs in CI workflow
- Incremental TypeScript compilation
- Webpack cache configuration
- Tree shaking and dead code elimination
- Split large bundles into smaller parallel builds

What NOT to try:
- Don't skip tests
- Don't remove type checking
- Don't change output format
```

### Example 5: ML Training (Karpathy-style)

```markdown
# autoimprove: better-gpt

## Change
scope: train.py
exclude: prepare.py, data/

## Check
run: python train.py
score: SCORE: {value}
goal: lower
timeout: 5m

## Stop
budget: 8h
stale: 20

## Instructions

Improve validation bits-per-byte (val_bpb) on the training run.

Patterns to try:
- Architecture changes (attention patterns, activation functions, normalization)
- Optimizer modifications (learning rate schedules, weight decay)
- Embedding strategies (RoPE variants, positional encoding)
- Training efficiency (gradient accumulation, mixed precision)
- Regularization (dropout placement, label smoothing)

What NOT to try:
- Don't modify prepare.py or the evaluation harness
- Don't add new dependencies
- Complexity must pay for itself
- Deleting code for equal results IS an improvement
```

### Example 6: Kubernetes Cluster Health

```markdown
# autoimprove: fix-oom-crashes

## Change
scope: the API and worker deployment manifests
exclude: k8s/services/, k8s/ingress/

## Check
run: kubectl apply -f k8s/ && sleep 60 && kubectl get pods --no-headers | grep -c Running
score: SCORE: {value}
goal: higher
timeout: 3m

## Stop
target: 5
rounds: 20

## Instructions

Get all 5 pods to Running state. Currently some are OOMKilled or CrashLoopBackOff.

Patterns to try:
- Adjust resource requests and limits
- Add or tune liveness/readiness probes
- Configure pod anti-affinity
- Adjust HPA thresholds
- Fix environment variables or config references

What NOT to try:
- Don't change the container images
- Don't modify the service definitions
- Don't delete pods manually
```

### Example 7: SQL Query Performance

```markdown
# autoimprove: optimize-dashboard-queries

## Change
scope: queries/dashboard.sql

## Check
run: psql -f queries/dashboard.sql -c '\timing' 2>&1
score: Time: ([\d.]+) ms
goal: lower
timeout: 1m

## Stop
stale: 10

## Instructions

Reduce query execution time without changing the result set.

Patterns to try:
- Replace correlated subqueries with JOINs
- Use CTEs for readability but test if they hurt performance
- Add index hints or restructure WHERE clauses for index usage
- Partition large scans with date ranges
- Materialize expensive aggregations

What NOT to try:
- Don't change the schema or indexes (those are read-only context)
- Query results must remain identical
- Don't use database-specific extensions not in the current schema
```

### Example 8: Frontend Bundle Size

```markdown
# autoimprove: smaller-bundle

## Change
scope: the entry point, dependencies, and build config
exclude: src/components/

## Check
run: npm run build && echo SCORE: $(stat -f%z dist/index.js)
goal: lower
timeout: 2m

## Stop
budget: 2h

## Instructions

Reduce the production bundle size without breaking functionality.

Patterns to try:
- Dynamic imports for routes and heavy components
- Replace heavy libraries with lighter alternatives
- Tree shaking configuration
- Remove unused exports and dead code
- Code splitting by route or feature

What NOT to try:
- Don't remove features
- Don't break lazy loading
- Don't switch build tools entirely
```

### Example 9: Tabular ML / AutoML (Churn, Fraud, Scoring)

This is the most common ML task across companies — predicting outcomes on structured data. Traditional AutoML (AutoSklearn, FLAML, AutoGluon) searches a predefined hyperparameter grid. Autoimprove goes further: it can engineer new features, rewrite preprocessing, swap model architectures, and delete dead code.

```markdown
# autoimprove: better-churn-model

## Change
scope: the training pipeline
exclude: data/, evaluate.py

## Check
test: python -m pytest tests/ -x
test-files: tests/
run: python train.py && python evaluate.py
score: auc_roc: ([\d.]+)
goal: higher
timeout: 3m

## Stop
budget: 4h
target: 0.95
stale: 20

## Agent
provider: claude
model: sonnet

## Instructions

Improve AUC-ROC on the holdout test set for customer churn prediction.
The training data has ~50 columns: demographics, usage metrics, billing history,
support tickets, and engagement signals.

Feature engineering to try:
- Ratio features (e.g., support_tickets / months_active, revenue / logins)
- Rolling aggregates (7d, 30d, 90d windows over usage metrics)
- Interaction terms between high-importance features
- Binning continuous variables (tenure buckets, spend tiers)
- Target encoding for high-cardinality categoricals
- Recency features (days since last login, last purchase, last support ticket)
- Trend features (is usage increasing or decreasing over time?)

Model changes to try:
- XGBoost, LightGBM, CatBoost — compare all three
- Hyperparameters: learning_rate, max_depth, n_estimators, subsample, colsample_bytree
- Class imbalance handling: scale_pos_weight, SMOTE, undersampling
- Ensemble: stack top 2-3 models with a logistic regression meta-learner
- Calibration: isotonic or Platt scaling for probability calibration

Preprocessing to try:
- Handle missing values: median, mode, or indicator columns
- Log-transform skewed features
- Remove highly correlated features (>0.95 correlation)
- Feature selection: drop low-importance features to reduce noise

What NOT to try:
- Don't modify evaluate.py or the test data
- Don't use the test set during training (no leakage)
- Don't add deep learning — this is tabular data, tree models win here
- Don't add more than 2 new dependencies
- Keep the pipeline reproducible (set random seeds)
```

This example works for any tabular prediction task — swap "churn" for fraud detection,
credit scoring, lead conversion, demand forecasting, or insurance pricing. The structure
is the same: features in columns, a target to predict, a metric to maximize.

### Example 10: RAG Pipeline Optimization

RAG (Retrieval-Augmented Generation) pipelines have many interacting knobs — chunking,
embedding, retrieval, reranking, prompt template, context window management. Small
changes compound: better chunking improves retrieval which improves generation quality.
Autoimprove can explore this space much faster than manual tuning.

```markdown
# autoimprove: better-rag-answers

## Change
scope: the RAG pipeline — chunking, retrieval, and generation
exclude: data/, eval/

## Check
test: python -m pytest tests/test_pipeline.py -x
test-files: tests/
run: python eval/run_eval.py
score: answer_relevancy: ([\d.]+)
goal: higher
timeout: 5m

## Stop
budget: 6h
target: 0.92
stale: 15

## Agent
provider: claude
model: sonnet

## Instructions

Improve answer relevancy on the evaluation set of 50 question-answer pairs.
The pipeline currently uses LangChain with a vector store, but you can
restructure it however you want.

Chunking strategies to try:
- Vary chunk size (256, 512, 1024, 2048 tokens) and overlap (50, 100, 200)
- Semantic chunking — split on topic boundaries instead of fixed token counts
- Hierarchical chunking — parent chunks for context, child chunks for retrieval
- Document-aware splitting — respect headers, paragraphs, code blocks, tables
- Sentence-level chunking with sliding window for dense passages

Retrieval strategies to try:
- Increase or decrease top-k (3, 5, 10, 15)
- Hybrid search — combine dense (embedding) and sparse (BM25) retrieval
- Reranking — add a cross-encoder reranker after initial retrieval
- Query expansion — rephrase the query multiple ways, merge results
- Query decomposition — split complex questions into sub-questions
- Metadata filtering — use document metadata to narrow the search space
- MMR (Maximal Marginal Relevance) — diversify retrieved chunks

Embedding changes to try:
- Switch embedding model (nomic-embed, bge-large, e5-mistral, cohere-embed-v3)
- Fine-tune embedding on domain-specific data if training pairs available
- Normalize embeddings for cosine similarity
- Instruction-prefixed embeddings ("Represent this document for retrieval: ...")

Generation prompt to try:
- Structured context presentation (numbered sources with metadata)
- Chain-of-thought before answering ("First, let me identify relevant information...")
- Citation-required format ("Answer based on the sources. Cite [Source N].")
- Concise vs. detailed instruction tuning
- System prompt variations for tone, length, and specificity
- "If the context doesn't contain the answer, say so" (reduce hallucination)

Context window management to try:
- Reorder chunks — most relevant first vs. most relevant in the middle
- Compress context — summarize long chunks before passing to LLM
- Dynamic context sizing — use more chunks for complex questions, fewer for simple
- Deduplication — remove near-duplicate chunks that waste context space

What NOT to try:
- Don't modify the evaluation script or golden answers
- Don't change the LLM used for generation (that's a separate variable)
- Don't add more than 3 new dependencies
- Don't download or switch to a different document corpus
- Keep inference cost per query reasonable — no 5-call chains for a single answer
```

This works for any RAG system — internal knowledge bases, customer support bots,
documentation search, legal document retrieval, or code search. The score metric
can be swapped: use `faithfulness` to reduce hallucination, `context_precision` to
improve retrieval, or a composite RAGAS score for overall quality.
