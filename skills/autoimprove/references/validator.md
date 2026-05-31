# Golden Set Validation Protocol

Pre-loop validation that catches impossible expectations, missing data, and broken evals before the loop wastes rounds on phantom failures.

## When to Run

- Automatically during readiness check (Step 1, between "Eval harness" and "Test suite")
- Manually via `/autoimprove validate`
- Before any headless/overnight run

## The 4 Checks

### Check 1: Expected Data Exists

For each query in the golden set, verify that expected references are satisfiable:

- **expected_sessions**: Each session ID or prefix must exist in the index/database. Run a direct lookup (not search) to confirm.
- **expected_projects**: At least one result from that project must be findable in the corpus.
- **expected_files**: Referenced files must exist in the indexed dataset.

**Failure mode this catches:** Golden set references data that was deleted, renamed, or never indexed. The eval reports "MISS" but the search system has no way to succeed.

### Check 2: Expected Keywords Are Findable

For each query with expected keywords:

1. Run the query through the search system
2. Check if each expected keyword appears in ANY result in the top-20 (not just top-k)
3. If a keyword never appears in any result across the entire result set, flag it

**Failure mode this catches:** Expectations that assume content exists in a field where it doesn't. Example: expecting "HPE" in a session summary when the keyword only appears in the transcript body, and the search system only indexes summaries.

### Check 3: Score Sanity

Run the full eval on the unmodified codebase:

- **Score = 0.0**: Nothing works. Either the eval is broken or the system is completely non-functional. Neither is a good starting point.
- **Score = 1.0**: Everything is perfect. The eval isn't discriminating — it will report 1.0 for any change, making the loop pointless.
- **Score between 0.0 and 0.15**: Very low. Likely indicates systemic eval issues, not just poor search quality.
- **Score between 0.85 and 1.0**: Very high. Limited headroom for improvement — consider whether the loop is worth the compute.

**Failure mode this catches:** Broken eval scripts that always return the same score, or evals that don't actually test what you think they test.

### Check 4: Error Isolation

For each query that errors or misses, report the specific reason:

| Diagnosis | What it means |
|-----------|--------------|
| No results found | Query returns empty — search system can't find anything relevant |
| Results found, wrong project | Search finds results but from unexpected projects |
| Results found, missing keywords | Search finds relevant sessions but expected keywords aren't in them |
| Session ID not found | Expected session doesn't exist in the index |
| Timeout | Query took too long — performance issue, not accuracy |
| Exception | Search system threw an error — fix the bug before optimizing |

**Failure mode this catches:** Lumping all failures into "MISS" hides whether the problem is in the search system or the eval. Error isolation turns each miss into actionable diagnostics.

## Output Format

```
Golden set validation (N queries):
  ✓ K queries: expectations are satisfiable
  ⚠ W queries: expectations may be wrong
    - "<query>": <specific reason>
    - "<query>": <specific reason>
  ✗ F queries: expectations are impossible
    - "<query>": <specific reason>

Score sanity: <baseline score> — <assessment>
Error isolation: <summary of error categories>

Recommendation: <action>
Proceed anyway? [y/n]
```

### Severity Levels

- **✓ Pass**: Expected data exists, keywords are findable, expectations are satisfiable.
- **⚠ Warning**: Expectations may be wrong but aren't provably impossible. The loop might waste rounds on these queries. Recommend fixing but allow proceeding.
- **✗ Hard failure**: Expected data doesn't exist in the index. The eval WILL report a miss regardless of search quality. Refuse to start until fixed.

### Decision Logic

| Hard failures (✗) | Warnings (⚠) | Action |
|-------------------|--------------|--------|
| 0 | 0 | Proceed automatically |
| 0 | 1-3 | Recommend fixing, allow proceeding |
| 0 | 4+ | Strongly recommend fixing, allow proceeding |
| 1+ | any | Refuse to start — fix the hard failures first |

## Examples

### Example: RAG evaluation

```
Golden set validation (20 queries):
  ✓ 17 queries: expectations are satisfiable
  ⚠ 2 queries: expectations may be wrong
    - "d941e8ec": expected keywords ["skills", "HPE"] but session summary
      doesn't contain these — they appear in transcript only (not indexed)
    - "MCP server gateway authentication": expected project "zanetworker/research"
      but top-20 results are from "harness/interface" (correct project not in corpus?)
  ✗ 1 query: expected session "nonexistent-id" does not exist in index

Score sanity: 0.4398 — reasonable baseline (room for improvement)
Error isolation: 2 keyword mismatches, 1 missing session, 0 errors

Recommendation: Fix the 1 hard failure and review the 2 warnings before starting.
```

### Example: ML evaluation

```
Golden set validation (100 test samples):
  ✓ 98 samples: data exists and labels are valid
  ⚠ 2 samples: feature values outside training distribution
    - sample_042: "age" = -5 (negative age, likely data error)
    - sample_089: "income" = 0.0 (zero income, may be missing value coded as 0)
  ✗ 0 samples: no hard failures

Score sanity: 0.72 AUC-ROC — reasonable baseline
Error isolation: 0 errors

Recommendation: Review the 2 data quality warnings. Proceed when ready.
```

### Example: Prompt evaluation

```
Golden set validation (30 test cases):
  ✓ 28 cases: expected outputs are achievable
  ⚠ 1 case: expected output contains information not in the prompt context
    - "summarize Q3 results": expected output mentions "$4.2M revenue"
      but the prompt context only contains percentage growth, not absolute numbers
  ✗ 1 case: expected output contradicts the input
    - "classify sentiment": input is "Great product!" but expected label is "negative"

Score sanity: 0.65 F1 — reasonable baseline
Error isolation: 1 label error, 1 context mismatch

Recommendation: Fix the label error (hard failure) and review the context mismatch.
```

## Integration with the Loop

The validator runs between "Eval harness" (step 3) and "Test suite" (step 4) in the readiness check. This placement ensures:

1. The check command already works (verified in step 3)
2. The validator can run the check command to get per-query diagnostics
3. Validation happens before tests, so test generation (bootstrap) can account for known eval issues

If the validator finds hard failures, the loop does not start. The user must fix the golden set first. This prevents the scenario from the session-search-mcp run where 40% of experiments were wasted fixing eval errors during the loop.
