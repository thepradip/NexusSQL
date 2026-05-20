# Changelog

## [Agent] - 2026-05-20

### Fixed — Visualization pipeline after standalone service migration

- **`backend/react_agent.py`** — replaced `from visualization import build_visualization` (broken after `visualization.py` was deleted in the standalone service PR) with `from visualization_client import infer_visualization`; made `_build_result` async so it can await the HTTP call. Agentic mode was crashing with `ImportError` on first use.
- **`frontend/src/components/DataVisualization.jsx`** — removed forced zero baseline on line chart Y-axis (`Math.min(...values, 0)` → `Math.min(...values)`). High-value trend data (e.g. revenue in thousands) was rendering as a flat line.

### Added — Docker support

- `services/visualization_service/Dockerfile` — builds the chart inference microservice from the project root (required for package-relative imports)
- `docker-compose.yml` — adds `ariasql-visualization` container on port 8011; backend `depends_on` visualization with healthcheck; overrides `VISUALIZATION_SERVICE_URL` to `http://visualization:8011` for container networking
- `start.sh` — updated to start the visualization service (port 8011) alongside the backend and frontend

---

## [2.4.0] - 2026-05-04

### Added — Prompt versioning, schema retrieval quality

**New module: `sqlas/prompt_registry.py`**
- `PromptRegistry.register(prompt_text, version_id, description)` — version your SQL agent prompts
- `PromptRegistry.record(version_id, scores)` — tag every evaluation with its active prompt version
- `PromptRegistry.compare(v1, v2)` — head-to-head comparison with per-metric deltas
- `PromptRegistry.detect_regression(version_id, window, threshold)` — fires when recent scores drop vs baseline
- `PromptRegistry.improvement_hints(version_id)` — data-driven prompt fix suggestions for failing metrics
- Backed by SQLite WAL, in-memory for fast lookup

**New module: `sqlas/schema_quality.py`**
- `schema_retrieval_quality(retrieved_tables, generated_sql, gold_tables)` — precision, recall, F1 for schema index
- `batch_retrieval_quality(evaluations)` — aggregate retrieval metrics across a test suite
- FK-penalty: missing JOIN tables are penalised more than missing non-JOIN tables

**New `SQLASScores` fields**
- `schema_retrieval_f1`, `schema_retrieval_precision`, `schema_retrieval_recall`, `schema_retrieval_missing`
- `prompt_id` — which prompt version produced this result

**`evaluate()` new parameters**
- `retrieved_tables` — enables schema retrieval scoring
- `prompt_id` — tagged into scores for registry recording

---

## [2.3.0] - 2026-05-03

### Added — Three-stage guardrails, feedback learning loop

**New module: `sqlas/guardrails.py`**
- `GuardrailPipeline.check_input(query)` — Stage 1: prompt injection, PII requests, malicious intent
- `GuardrailPipeline.check_sql(sql, valid_tables, valid_columns)` — Stage 2: AST read-only, injection, PII column access
- `GuardrailPipeline.check_sql_quality(question, sql, llm_judge)` — Stage 2b: optional LLM quality gate
- `GuardrailPipeline.check_output(response, result_data)` — Stage 3: PII leakage, PII in result rows
- `GuardrailPipeline.run_pipeline(...)` — runs all 3 stages, returns first blocked stage
- `GuardrailResult` dataclass with stage, safe, score, issues, blocked, block_reason

**New module: `sqlas/feedback.py`**
- `FeedbackStore.store(FeedbackEntry)` — stores verified (question→SQL) pairs from thumbs-up
- `FeedbackStore.get_gold_sql(question)` — auto-supplies gold SQL in `evaluate_correctness()`
- `FeedbackEntry` dataclass: question, sql, is_correct, score, source, notes
- `evaluate_correctness(feedback_store=store)` — auto-lookup eliminates manual gold_sql argument

**New standalone evaluators**
- `evaluate_correctness()` — only correctness metrics (1 LLM call)
- `evaluate_quality()` — only quality metrics (6 LLM calls)
- `evaluate_safety()` — zero LLM calls, pure regex + AST
- Each returns its own typed result dataclass: `CorrectnessResult`, `QualityResult`, `SafetyResult`

---

## [2.2.0] - 2026-05-03

### Added — Three-dimension scoring with AND verdict logic

- Three independent composite scores: `correctness_score`, `quality_score`, `safety_composite_score`
- `verdict` field: `PASS` only when all three exceed thresholds (0.5 / 0.6 / 0.9)
- `WEIGHTS_CORRECTNESS`, `WEIGHTS_QUALITY`, `WEIGHTS_SAFETY` dicts
- `compute_verdict(correctness, quality, safety)` — AND logic
- `THRESHOLDS` dict — configurable per deployment
- `overall_score` = 0.50×correctness + 0.30×quality + 0.20×safety (backward compat)

---

## [2.1.0] - 2026-05-02

### Added — Large schema support (100+ tables)

- `build_schema_info(db_path, execute_fn)` — auto-extract valid_tables + valid_columns from any DB
- `_auto_schema_context(sql, valid_columns)` — injects only tables referenced in SQL (not all 100+)
- `result_coverage(result_data, sql)` — truncation-aware metric: truncated GROUP BY = 0.3
- `data_scan_efficiency` now uses `truncated` flag (not capped row_count) for row explosion detection
- `run_suite()` — new `schema_context` param + `TestCase.schema_context` field

### Fixed
- `execution_accuracy` no longer returns 1.0 when no gold_sql provided — now 0.5 with `unverified=True`

---

## [2.0.0] - 2026-04-30

### Added — Agentic SQL Agent support

**New module: `sqlas/agentic.py`**
- `steps_efficiency(steps_taken, optimal_steps=3)` — penalises agents that used more steps than necessary
- `schema_grounding(steps)` — checks whether the agent inspected schema before writing SQL
- `planning_quality(question, steps, llm_judge)` — LLM judge on reasoning sequence quality
- `tool_use_accuracy(question, steps, llm_judge)` — LLM judge on tool selection correctness
- `agentic_score(question, steps, llm_judge)` — composite (30% efficiency + 30% grounding + 40% planning)

**New module: `sqlas/cache.py`**
- `cache_hit_score(agent_result)` — 1.0 if served from semantic/exact cache
- `tokens_saved_score(agent_result)` — normalised token savings vs full pipeline
- `few_shot_score(agent_result)` — were verified few-shot examples injected?

**New weight profile: `WEIGHTS_V4`**
- Extends V3 with a 10% agentic quality dimension
- Core correctness adjusted to 25% to accommodate

**`evaluate()` new parameters (fully backward compatible)**
- `agent_steps: list[dict] | None` — ReAct loop steps for agentic quality scoring
- `agent_result: dict | None` — full agent result dict for cache metric extraction
- `optimal_steps: int = 3` — step count considered ideal for efficiency scoring

**`SQLASScores` new fields**
- Agentic: `agent_mode`, `steps_taken`, `steps_efficiency`, `schema_grounding`, `planning_quality`, `tool_use_accuracy`, `agentic_score`
- Cache: `cache_hit`, `cache_type`, `tokens_saved`, `few_shot_count`

### Changed — Security upgrade

**`read_only_compliance` (breaking improvement)**
- Upgraded from keyword regex (bypassable) to **sqlglot AST parsing**
- Now correctly blocks write operations buried inside CTE definitions and after SQL comments
- Falls back to keyword scan gracefully if sqlglot parse fails
- `sqlglot` was already a required dependency — no new install needed

## [1.3.0] - 2026-04-29

### Added
- `execute_fn` parameter on `execution_accuracy()`, `result_set_similarity()`, `evaluate()`, `evaluate_batch()`, and `run_suite()`
- `ExecuteFn = Callable[[str], list[tuple]]` type alias exported from the public API
- `execute_fn` takes precedence over `db_path` when both are supplied, enabling evaluation against **any database** — Postgres, MySQL, Snowflake, BigQuery, Databricks, and more
- `db_path` existence check is skipped when `execute_fn` is provided (no false "file not found" errors for remote DBs)
- 63 new tests in `tests/test_execute_fn.py` covering: basic execution, error handling, precedence rules, batch/suite evaluation, 200-table large-schema performance, and a simulated non-SQLite executor

### Changed
- `execution_accuracy` and `result_set_similarity` signatures updated: `db_path` is now `str | None` (was `str`) — fully backward compatible

## [1.2.0] - 2026-04-15

### Added
- New `visualization.py` module with 4 chart quality metrics:
  - `chart_spec_validity` — validates renderable chart payload structure
  - `chart_data_alignment` — checks chart keys align with SQL result columns
  - `chart_llm_validation` — LLM-as-judge for chart relevance and commentary
  - `visualization_score` — composite visualization score
- `WEIGHTS_V3` weight profile (30 metrics across 8 categories) with explicit guardrail and visualization weights
- Granular guardrail sub-metrics now exported individually: `sql_injection_score`, `prompt_injection_score`, `pii_access_score`, `pii_leakage_score`, `guardrail_score`
- `SQLASScores` dataclass extended with visualization fields: `chart_spec_validity`, `chart_data_alignment`, `chart_llm_validation`, `visualization_score`
- `validate_chart_with_llm` parameter on `evaluate()` and `run_suite()` to toggle LLM chart validation
- `visualization` parameter on `evaluate()` to pass chart payload for scoring

### Changed
- `summary()` output now includes Visualization and Guardrails categories
- `to_dict()` now exports all v1/v2/v3 metric keys

## [1.1.0] - 2026-03-30

### Added
- New `context.py` module with 4 RAGAS-mapped context quality metrics:
  - `context_precision` — schema element precision vs gold SQL
  - `context_recall` — schema element recall vs gold SQL
  - `entity_recall` — strict entity-level recall (tables, columns, literals, functions)
  - `noise_robustness` — resistance to irrelevant schema elements
- `result_set_similarity` metric in `correctness.py` (Jaccard similarity on result sets)
- `WEIGHTS_V2` weight profile with all 20 metrics across 8 categories
- `py.typed` marker for PEP 561 type checking support
- Configurable `pass_threshold` parameter in `run_suite()`
- Input validation in `evaluate()` (empty SQL, missing db_path, weight sum check)
- Structured logging via `logging` module across all modules

### Changed
- `SQLASScores` dataclass now includes 5 new context quality fields (all default 0.0, backward compatible)
- `evaluate()` conditionally computes context metrics when `gold_sql` is provided
- `evaluate_batch()` now forwards all parameters (`schema_context`, `expected_nonempty`, `pii_columns`)
- `summary()` output now includes Context Quality category
- Development status upgraded from Beta to Production/Stable

### Fixed
- **Security**: Database connections now use read-only mode (`?mode=ro`) to prevent data corruption
- **Security**: SQL execution timeout guard prevents indefinite hangs (default 30s)
- **Resilience**: All LLM judge calls wrapped in try/except — returns 0.0 with error details instead of crashing
- **PII detection**: Uses regex word boundaries (`\b`) to prevent false positives (e.g. `ip_address_log` no longer matches `address`)
- **`syntax_valid`**: Simplified to trust sqlglot parse result instead of fragile keyword checks
- **Score parsing**: Consolidated duplicated `_parse_score` helper into `core.py`

## [1.0.1] - 2026-03-26

### Initial release
- 15 production metrics across 6 categories
- LLM-agnostic judge interface
- MLflow integration via optional dependency
- Test suite for automated metrics
