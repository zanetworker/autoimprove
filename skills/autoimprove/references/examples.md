# Autoimprove Examples

Complete `improve.md` templates for different domains. Use `/autoimprove init --type <type>` to scaffold these.

## Types

| Type | What it optimizes | Typical metric | Typical budget |
|------|-------------------|----------------|----------------|
| `perf` | Code performance | Requests/sec, time, allocations | 4-8h |
| `ml` | ML training | Validation loss, accuracy, BPB | 8-24h |
| `docker` | Container image | Image size in bytes | 1-2h |
| `k8s` | Cluster health | Running pod count, error rate | 1-2h |
| `prompt` | LLM prompt quality | F1, accuracy, similarity score | 1-2h |
| `sql` | Query performance | Execution time in ms | 1-2h |
| `frontend` | Bundle size | Bundle bytes | 2-4h |
| `ci` | CI/build speed | Build time in seconds | 2-4h |
| `automl` | Tabular ML (churn, fraud, scoring) | AUC-ROC, F1, accuracy | 4-8h |
| `rag` | RAG pipeline (retrieval + generation) | Answer relevancy, faithfulness, RAGAS | 4-8h |
| `skill` | Claude Code skill quality | Eval pass rate, trigger accuracy | 4-8h |
| `image` | Image generation prompt quality | ImageReward, CLIP Score, aesthetic score | 1-3h |

## perf — Code Performance

```markdown
# autoimprove: faster-<component>

## Change
files: <hot-path source files>
context: <test files, benchmarks>

## Check
run: <test suite> && <benchmark command>
score: <benchmark metric extraction>
goal: lower
timeout: 5m

## Stop
budget: 4h
stale: 15

## Instructions

Focus on reducing allocations and avoiding unnecessary work in hot paths.

Patterns to try:
- Fast-path with fallback — handle common cases with optimized code, fall back to general path for edge cases
- Reduce object allocations — reuse buffers, avoid temporary arrays/maps
- Byte-level operations instead of string operations in parsing
- Cache repeated computations (small integer to_s, frozen strings)
- Splat-free method dispatch for frequently called methods
- instance_of? over is_a? in hot loops (skip inheritance chain)

What NOT to try:
- Don't change the public API
- Don't add external dependencies
- Don't sacrifice readability for marginal gains (<0.1%)
- Complexity must pay for itself

## Experiments

### Structural (propose to human first)
- Switch from string scanning to byte-level lexer
- Add a two-pass parsing strategy (fast path + fallback)
- Replace recursive descent with table-driven parser
- Add memoization cache for repeated subexpressions

### Parametric (autonomous loop)
- Tune buffer pool size
- Adjust cache expiration threshold
- Change initial allocation capacity
- Tune fast-path character set detection threshold
```

## ml — ML Training

```markdown
# autoimprove: better-<model>

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

Improve validation loss within the fixed training time budget.

Patterns to try:
- Architecture (attention patterns, activation functions, normalization layers)
- Optimizer (learning rate schedules, weight decay, momentum)
- Embeddings (positional encoding variants, value embeddings)
- Training dynamics (warmup, gradient clipping, batch size)
- Regularization (dropout, label smoothing, data augmentation)

What NOT to try:
- Don't modify the data pipeline or evaluation
- Don't add dependencies
- Simplicity over marginal gains
- Deleting code for equal results IS an improvement

## Experiments

### Structural (propose to human first)
- Switch attention mechanism (multi-head to grouped-query)
- Replace LayerNorm with RMSNorm
- Add rotary positional embeddings
- Switch optimizer (Adam to Muon or SOAP)

### Parametric (autonomous loop)
- Tune learning rate and warmup steps
- Adjust weight decay coefficient
- Change gradient clipping threshold
- Tune dropout rate
- Adjust batch size
```

## docker — Container Image Size

```markdown
# autoimprove: slim-<image>

## Change
files: Dockerfile
context: <build context files>

## Check
run: docker build -t test . && echo SCORE: $(docker image inspect test --format '{{.Size}}')
goal: lower
timeout: 5m

## Stop
rounds: 30

## Instructions

Reduce image size without breaking runtime behavior.

Patterns to try:
- Multi-stage builds (build stage + minimal runtime stage)
- Alpine or distroless base images
- Combine RUN layers to reduce intermediate layer count
- Remove build tools, caches, and dev deps from final stage
- Use scratch for static binaries
- .dockerignore to exclude unnecessary build context

What NOT to try:
- Don't remove runtime dependencies
- Don't use UPX (breaks debugging and coredumps)
- The app must still start and pass health checks

## Experiments

### Structural (propose to human first)
- Switch base image family (debian to alpine/distroless/scratch)
- Add multi-stage build
- Switch to a different package manager (apk vs apt)

### Parametric (autonomous loop)
- Combine RUN layers
- Remove unused packages from install list
- Tune .dockerignore entries
- Reorder COPY instructions for cache efficiency
```

## k8s — Kubernetes Health

```markdown
# autoimprove: fix-<issue>

## Change
files: <deployment/service YAML files>
context: k8s/

## Check
run: kubectl apply -f k8s/ && sleep 60 && <health check command>
score: SCORE: {value}
goal: higher
timeout: 3m

## Stop
target: <desired healthy count>
rounds: 20

## Instructions

Get pods to healthy Running state.

Patterns to try:
- Resource requests and limits (memory/CPU)
- Liveness and readiness probe tuning
- Pod anti-affinity rules
- Environment variable and config fixes
- HPA min/max replicas and target utilization
- Init container dependencies

What NOT to try:
- Don't change container images
- Don't delete and recreate namespaces
- Don't modify service mesh config
- One manifest change per experiment

## Experiments

### Structural (propose to human first)
- Add init containers for dependency ordering
- Add pod disruption budgets
- Switch from Deployment to StatefulSet
- Add horizontal pod autoscaler

### Parametric (autonomous loop)
- Adjust memory requests and limits
- Tune CPU requests and limits
- Adjust liveness/readiness probe intervals and thresholds
- Change HPA target utilization percentage
- Tune replica count
```

## prompt — Prompt Engineering

```markdown
# autoimprove: better-<task>-prompt

## Change
files: prompts/<task>.txt
context: eval/golden_set.jsonl

## Check
run: python eval/run_eval.py --prompt prompts/<task>.txt
score: <metric>: ([\d.]+)
goal: higher
timeout: 2m

## Stop
budget: 1h
target: 0.95

## Instructions

Improve prompt accuracy on the evaluation set.

Patterns to try:
- Few-shot examples (pick diverse, representative cases)
- Chain-of-thought before final answer
- Structured output format (JSON with typed fields)
- Negative examples (what NOT to do)
- Simplify — shorter prompts often outperform verbose ones
- Role/persona framing

What NOT to try:
- Don't modify the evaluation script
- Don't overfit to specific test cases
- Keep prompt under 2000 tokens
- Don't use model-specific tricks that break portability

## Experiments

### Structural (propose to human first)
- Add chain-of-thought reasoning step
- Switch from single prompt to multi-turn conversation
- Add a self-verification step (generate then critique)
- Switch output format (free text to JSON schema)

### Parametric (autonomous loop)
- Adjust few-shot example count and selection
- Tune system prompt wording
- Vary instruction specificity (concise vs detailed)
- Adjust output length constraints
```

## sql — Query Performance

```markdown
# autoimprove: faster-<query>

## Change
files: queries/<name>.sql
context: schema.sql, indexes.sql

## Check
run: psql -f queries/<name>.sql -c '\timing' 2>&1
score: Time: ([\d.]+) ms
goal: lower
timeout: 1m

## Stop
stale: 10

## Instructions

Reduce query time without changing result set.

Patterns to try:
- Replace correlated subqueries with JOINs
- Use EXISTS instead of IN for existence checks
- Add covering index hints via query restructuring
- Partition large scans with range predicates
- Materialize expensive CTEs if they're scanned multiple times

What NOT to try:
- Don't modify schema or indexes
- Results must be identical (same rows, same order)
- Don't use database-specific extensions not in the schema

## Experiments

### Structural (propose to human first)
- Rewrite correlated subquery as a JOIN
- Replace UNION with UNION ALL where duplicates are impossible
- Materialize expensive CTE as a temp table
- Add window functions to replace self-joins

### Parametric (autonomous loop)
- Reorder WHERE clause predicates for selectivity
- Adjust LIMIT/OFFSET values
- Tune date range partition boundaries
- Adjust aggregate grouping granularity
```

## frontend — Bundle Size

```markdown
# autoimprove: smaller-<app>

## Change
files: src/index.ts, package.json, <build config>
context: src/

## Check
run: npm run build && echo SCORE: $(stat -f%z dist/index.js)
goal: lower
timeout: 2m

## Stop
budget: 2h

## Instructions

Reduce production bundle size.

Patterns to try:
- Dynamic imports for routes and heavy components
- Replace heavy libraries with lighter alternatives (moment→dayjs, lodash→lodash-es)
- Tree shaking configuration
- Remove unused exports
- Code splitting by route or feature
- Externalize large dependencies

What NOT to try:
- Don't remove features
- Don't break lazy loading
- Don't switch build tools
- App must still work in the browser

## Experiments

### Structural (propose to human first)
- Replace a heavy library with a lighter alternative (moment to dayjs)
- Add code splitting by route
- Switch bundler tree-shaking strategy
- Externalize large dependencies via CDN

### Parametric (autonomous loop)
- Remove unused imports and exports
- Adjust chunk size thresholds
- Tune dynamic import boundaries
- Adjust minification settings
```

## ci — Build Speed

```markdown
# autoimprove: faster-<pipeline>

## Change
files: .github/workflows/ci.yml, <build config files>
context: src/, package.json

## Check
run: time <build command> 2>&1 | tail -1
score: real\s+(\d+\.\d+)
goal: lower
timeout: 10m

## Stop
stale: 15

## Instructions

Reduce build/CI time without skipping quality checks.

Patterns to try:
- Parallel jobs in CI workflow
- Caching (node_modules, build artifacts, docker layers)
- Incremental compilation
- Split test suites into parallel shards
- Conditional steps (skip unchanged paths)

What NOT to try:
- Don't skip tests or linting
- Don't remove type checking
- Don't change output artifacts
- Pipeline must remain correct

## Experiments

### Structural (propose to human first)
- Split test suite into parallel shards
- Add build artifact caching layer
- Switch to incremental compilation mode
- Add conditional steps (skip unchanged paths)

### Parametric (autonomous loop)
- Adjust parallelism level (concurrent jobs)
- Tune cache key granularity
- Adjust timeout values
- Change retry count for flaky steps
```

## automl — Tabular ML (Churn, Fraud, Scoring)

The most common ML task across companies. Unlike traditional AutoML (AutoSklearn, FLAML, AutoGluon) which searches a predefined parameter grid, autoimprove can engineer new features, rewrite preprocessing pipelines, swap models, and combine approaches creatively.

Applies to: churn prediction, fraud detection, credit scoring, lead conversion, demand forecasting, insurance pricing, recommendation ranking, customer lifetime value.

```markdown
# autoimprove: better-<prediction>-model

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

## Instructions

Improve AUC-ROC on the holdout test set.
The training data has structured columns: demographics, usage metrics,
billing history, support tickets, engagement signals.

Feature engineering to try:
- Ratio features (e.g., support_tickets / months_active)
- Rolling aggregates (7d, 30d, 90d windows over usage metrics)
- Interaction terms between high-importance features
- Binning continuous variables (tenure buckets, spend tiers)
- Target encoding for high-cardinality categoricals
- Recency features (days since last login, last purchase)
- Trend features (is usage increasing or decreasing?)

Model changes to try:
- XGBoost, LightGBM, CatBoost — compare all three
- Hyperparameters: learning_rate, max_depth, n_estimators, subsample, colsample_bytree
- Class imbalance: scale_pos_weight, SMOTE, undersampling
- Ensemble: stack top 2-3 models with logistic regression meta-learner
- Calibration: isotonic or Platt scaling

Preprocessing to try:
- Missing values: median, mode, or indicator columns
- Log-transform skewed features
- Remove highly correlated features (>0.95)
- Feature selection: drop low-importance features to reduce noise

What NOT to try:
- Don't modify evaluate.py or the test data
- Don't use the test set during training (no leakage)
- Don't add deep learning for tabular data — tree models win here
- Don't add more than 2 new dependencies
- Keep the pipeline reproducible (set random seeds)

## Experiments

### Structural (propose to human first)
- Switch model family (XGBoost to LightGBM or CatBoost)
- Add ensemble stacking with meta-learner
- Add target encoding for categoricals
- Add rolling aggregate feature engineering pipeline

### Parametric (autonomous loop)
- Tune learning_rate, max_depth, n_estimators
- Adjust subsample and colsample_bytree ratios
- Tune scale_pos_weight for class imbalance
- Adjust regularization (reg_alpha, reg_lambda)
- Tune min_child_weight
```

## rag — RAG Pipeline Optimization

RAG pipelines have many interacting knobs — chunking, embedding, retrieval, reranking,
prompt template, context window management. Small changes compound: better chunking
improves retrieval which improves generation quality. Autoimprove can explore this
combinatorial space much faster than manual tuning.

Applies to: internal knowledge bases, customer support bots, documentation search,
legal document retrieval, code search, research assistants, enterprise Q&A.

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

## Instructions

Improve answer relevancy on the evaluation set of 50 question-answer pairs.

Chunking strategies to try:
- Vary chunk size (256, 512, 1024, 2048 tokens) and overlap (50, 100, 200)
- Semantic chunking — split on topic boundaries instead of fixed token counts
- Hierarchical chunking — parent chunks for context, child chunks for retrieval
- Document-aware splitting — respect headers, paragraphs, code blocks, tables
- Sentence-level chunking with sliding window for dense passages

Retrieval strategies to try:
- Adjust top-k (3, 5, 10, 15)
- Hybrid search — combine dense (embedding) and sparse (BM25) retrieval
- Add a cross-encoder reranker after initial retrieval
- Query expansion — rephrase the query multiple ways, merge results
- Query decomposition — split complex questions into sub-questions
- MMR (Maximal Marginal Relevance) — diversify retrieved chunks

Embedding changes to try:
- Switch embedding model (nomic-embed, bge-large, e5-mistral, cohere-embed-v3)
- Instruction-prefixed embeddings
- Normalize embeddings for cosine similarity

Generation prompt to try:
- Structured context presentation (numbered sources with metadata)
- Chain-of-thought before answering
- Citation-required format ("Answer based on sources. Cite [Source N].")
- "If context doesn't contain the answer, say so" (reduce hallucination)
- Concise vs. detailed instruction tuning

Context window management to try:
- Reorder chunks — most relevant first vs. most relevant in the middle
- Compress context — summarize long chunks before passing to LLM
- Dynamic context sizing — more chunks for complex questions
- Deduplication — remove near-duplicate chunks

What NOT to try:
- Don't modify the evaluation script or golden answers
- Don't change the LLM used for generation (separate variable)
- Don't add more than 3 new dependencies
- Don't switch to a different document corpus
- Keep inference cost per query reasonable

## Experiments

### Structural (propose to human first)
- Add a cross-encoder reranker after initial retrieval
- Switch from fixed-size to semantic chunking
- Add query decomposition for complex questions
- Add hybrid search (combine dense and sparse retrieval)
- Add hierarchical chunking (parent for context, child for retrieval)

### Parametric (autonomous loop)
- Adjust chunk size and overlap
- Tune top-k retrieval count
- Adjust RRF constant for hybrid search blending
- Tune reranker score threshold
- Adjust context window token budget
- Tune MMR diversity lambda
```

Swap the metric based on what matters most: `faithfulness` to reduce hallucination,
`context_precision` to improve retrieval accuracy, or a composite RAGAS score for
overall pipeline quality.

## skill — Claude Code Skill Optimization

Optimizes Claude Code skills — their instructions, examples, trigger descriptions, protocol flow, progressive disclosure, and guard rails. The eval harness runs test prompts through Claude with the skill loaded and grades outputs against assertions. Supports both improving existing skills and building new ones from a minimal skeleton.

Default scope is the entire skill directory (SKILL.md, references/, scripts/, assets/). The user can narrow this in `improve.md`.

```markdown
# autoimprove: better-<skill-name>

## Change
scope: <skill-directory>/
exclude: eval/

## Check
test: python eval/run_skill_eval.py --check-assertions
test-files: eval/
run: python eval/run_skill_eval.py
score: pass_rate: ([\d.]+)
goal: higher
guard: trigger_false_positive: ([\d.]+) < 0.1
guard: token_usage: (\d+) < 50000
timeout: 5m

## Stop
budget: 4h
target: 0.95
stale: 15

## Instructions

Improve the skill's quality measured by eval query pass rate.
The eval runs test prompts through Claude with the skill loaded
and grades outputs against assertions.

Instruction clarity to try:
- Replace vague directives with specific, actionable steps
- Explain the WHY behind ALWAYS/NEVER rules — models follow reasoning better than commands
- Restructure protocol for clearer decision points and branching
- Simplify verbose sections — shorter often outperforms longer
- Use imperative form ("Run the test" not "You should run the test")

Trigger optimization to try:
- Make description focus on WHEN to trigger, not WHAT the skill does
- Include specific symptoms, contexts, and user phrases as triggers
- Add near-miss disambiguation to reduce false positives
- Slightly pushy descriptions combat under-triggering

Example quality to try:
- One excellent example beats many mediocre ones
- Examples should be complete, from real scenarios, well-commented
- Remove examples that duplicate the same pattern
- Add examples for failure modes agents hit repeatedly

Progressive disclosure to try:
- Move rarely-needed content from SKILL.md body to references/
- Keep SKILL.md under 500 lines — reference files for the rest
- Bundle repeated helper code into scripts/ directory
- Cross-reference instead of duplicating content across files

Guard rails to try:
- Close specific loopholes, not just state rules
- Add rationalization tables for discipline-enforcing rules
- Preempt "spirit vs letter" workarounds with explicit reasoning
- Build from observed agent failure patterns in eval runs

What NOT to try:
- Don't modify eval queries or assertions
- Don't overfit to specific eval cases — skill must generalize
- Don't add model-specific behavior
- Don't exceed 500 lines in SKILL.md without progressive disclosure
- Don't repeat content that exists in referenced skills or files
```

## image — Image Generation Prompt Optimization

Optimizes image generation prompts — the text passed to any image generation model (DALL-E, Flux, Stable Diffusion, Midjourney, or custom pipelines). The generation method and evaluation method are both agnostic: the user provides their own `run:` command that calls their pipeline and produces a score.

Image generation is stochastic — the same prompt produces different quality images across runs. The eval harness should generate multiple images with fixed seeds and average scores to reduce variance.

```markdown
# autoimprove: better-<task>-images

## Change
scope: prompts/<task>.txt
exclude: eval/, data/

## Check
test: python eval/validate_prompt.py --prompt prompts/<task>.txt
test-files: eval/
run: python eval/run_image_eval.py --prompt prompts/<task>.txt
score: image_reward: ([\d.-]+)
goal: higher
guard: clip_score: ([\d.]+) > 0.20
guard: nsfw_count: (\d+) < 1
timeout: 5m

## Stop
budget: 2h
target: 1.5
stale: 10

## Instructions

Improve the image generation prompt to produce higher-quality images
that better match the intended subject and style.

The eval harness generates multiple images from the prompt and
averages their scores to reduce variance from stochastic generation.

Prompt structure to try:
- Add specific medium/style (oil painting, cinematic photography, 3D render, watercolor)
- Specify lighting (golden hour, studio lighting, rim light, dramatic shadows)
- Add composition cues (rule of thirds, shallow depth of field, close-up, wide angle)
- Include mood/atmosphere (ethereal, dramatic, serene, gritty, cinematic)
- Specify color palette (warm tones, muted pastels, high contrast, monochromatic)
- Add artistic references for style anchoring

Prompt refinement to try:
- Reorder — put the most important detail first
- Replace vague words with specific descriptors
- Add negative context via phrasing ("without text or watermarks")
- Simplify — shorter prompts often outperform verbose ones
- Add few concrete details rather than many abstract ones
- Test: comma-separated keywords vs natural language prose

What NOT to try:
- Don't exceed ~200 tokens (diminishing returns, each word contributes less)
- Don't include model-specific syntax (--ar, (weight:1.5)) unless targeting that model
- Don't include NSFW or harmful content
- Don't hardcode for specific random seeds in the prompt text
- Keep the core subject and intent stable across iterations
```

Common scoring options:
- **ImageReward** (pip: `image-reward`): Best correlation with human preference. Scores range ~-2 to +2.
- **HPS v2** (pip: `hpsv2`): Strong generalization across styles. 83% accuracy predicting human preferences.
- **CLIP Score** (`torchmetrics.multimodal.CLIPScore`): Measures prompt-image alignment. Good as a guard metric.
- **LAION Aesthetic Score**: Lightweight (linear layer on CLIP). Good for coarse quality filtering.
- **MLLM-as-Judge**: GPT-4o or Claude evaluating images. Expensive but explainable. Best for periodic checkpoints.
