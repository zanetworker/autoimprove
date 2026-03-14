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
| `/autoimprove init` | Scaffold an improve.md (auto-detects repo type) |
| `/autoimprove init --type <type>` | Scaffold for a specific domain |
| `/autoimprove --export` | Generate agent-agnostic `program.md` |

## The `improve.md` Format

A single markdown file — part config, part prompt:

```markdown
# autoimprove: <name>

## Change
files: <what the agent can modify>
context: <read-only reference files>

## Check
run: <shell command — tests + metric>
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

### Step 2: Validate

Confirm:
- All files in `Change.files` exist
- The `Check.run` command is executable
- Git working tree is clean (no uncommitted changes)

If validation fails, report the issue and stop.

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
  4. Run the check command (with timeout)
  5. Extract the score

  6. DECIDE:
     IF check command failed (non-zero exit, timeout):
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
  Discarded: 31
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
  "status": "kept",
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
- NEVER modify files outside the `Change.files` list
- NEVER modify the check command or scoring mechanism
- NEVER skip the git commit before running the check
- ALWAYS log every experiment, including failures and errors
- ALWAYS read past experiments before proposing a new change
- ALWAYS git reset discarded experiments — leave the tree clean
- If the check command includes tests, a test failure means discard — no exceptions
- Complexity must pay for itself: 20 lines of hack for 0.001 improvement is NOT worth keeping
- Deleting code for equal results IS worth keeping
</HARD-RULES>

## Init Templates

When the user runs `/autoimprove init`, detect the repo type and suggest the right template. If `--type` is specified, use that directly.

Detection heuristics:
- `go.mod` or `Cargo.toml` or `Makefile` with benchmark targets → `perf`
- `train.py` or `*.ipynb` with torch/tensorflow imports → `ml`
- `train.py` with sklearn/xgboost/lightgbm/catboost imports → `automl`
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
files: src/handlers/checkout.go, src/db/queries.go
context: src/, test/

## Check
run: go test ./... && hey -n 1000 http://localhost:8080/checkout
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
files: Dockerfile
context: go.mod, cmd/

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
files: prompts/extract.txt
context: eval/golden_set.jsonl

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
files: .github/workflows/ci.yml, tsconfig.json, webpack.config.js
context: src/, package.json

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
files: train.py
context: prepare.py, data/

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
files: k8s/deployments/api.yaml, k8s/deployments/worker.yaml
context: k8s/

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
files: queries/dashboard.sql
context: schema.sql, indexes.sql

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
files: src/index.ts, package.json, vite.config.ts
context: src/

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
files: train.py
context: data/train.csv, data/test.csv, evaluate.py

## Check
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
