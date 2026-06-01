# SQLAS: SQL Agent Scoring Framework

**A RAGAS-equivalent evaluation library for Text-to-SQL and Agentic SQL agents.**

[![PyPI](https://img.shields.io/pypi/v/sqlas)](https://pypi.org/project/sqlas/)
[![Python](https://img.shields.io/pypi/pyversions/sqlas)](https://pypi.org/project/sqlas/)
[![Tests](https://img.shields.io/badge/tests-140%20passing-brightgreen)](https://github.com/thepradip/NexusSQL)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](../LICENSE)

Evaluate SQL agents across **50+ metrics**: correctness, quality, safety, agentic reasoning, schema retrieval, prompt versioning, guardrails, and cache ROI. Aligned with Spider, BIRD, RAGAS, and MLflow standards.

**Author:** [thepradip](https://github.com/thepradip)

---

## Install

```bash
pip install sqlas                # core
pip install "sqlas[mlflow]"      # + MLflow tracing
pip install "sqlas[ui]"          # + Streamlit dashboard
pip install "sqlas[all]"         # everything
```

---

## Quick Start

```python
from sqlas import evaluate

def llm_judge(prompt: str) -> str:
    return openai_client.chat.completions.create(
        model="gpt-4o", messages=[{"role": "user", "content": prompt}]
    ).choices[0].message.content

scores = evaluate(
    question      = "How many active users?",
    generated_sql = "SELECT COUNT(*) FROM users WHERE active = 1",
    gold_sql      = "SELECT COUNT(*) FROM users WHERE active = 1",
    db_path       = "my.db",
    llm_judge     = llm_judge,
    response      = "There are 1,523 active users.",
    result_data   = {"columns": ["COUNT(*)"], "rows": [[1523]], "row_count": 1, "execution_time_ms": 2.1},
)

print(scores.overall_score)        # 0.95
print(scores.correctness_score)    # 0.88
print(scores.verdict)              # PASS
print(scores.hardness)             # "easy"
print(scores.exact_match_score)    # 1.0
print(scores.to_markdown_report()) # Markdown for PR comments
```

---

## Three-Dimension Scoring

`PASS` only when **all three** dimensions meet their thresholds:

```python
from sqlas import evaluate_correctness, evaluate_quality, evaluate_safety

c = evaluate_correctness(question, sql, llm_judge, gold_sql=gold, execute_fn=db)
q = evaluate_quality(question, sql, llm_judge, response=text, result_data=data)
s = evaluate_safety(sql, question=question, pii_columns=["email", "ssn"])

print(c.score, c.verdict)   # 0.85  PASS   (threshold 0.5)
print(q.score, q.verdict)   # 0.72  PASS   (threshold 0.6)
print(s.score, s.verdict)   # 0.45  FAIL   (threshold 0.9, PII detected)
```

`evaluate_safety()` makes **zero LLM calls**: pure regex + sqlglot AST.

---

## Failure Classification

Know exactly *why* a query failed, not just a score:

```python
from sqlas import classify_failure, FailureCategory

analysis = classify_failure(
    sql     = "SELECT id FROM users LIMIT 100",
    scores  = {"execution_accuracy": 1.0, "row_count_match": 0.12},
    details = {"row_count_match": {"pred_count": 100, "gold_count": 839}},
)

print(analysis.primary)    # FailureCategory.LIMIT_TRUNCATION
print(analysis.top_hint()) # "Remove LIMIT, the question asks for full results, not top-N"
print(analysis.evidence)   # {"limit_truncation": "LIMIT in SQL, 100 rows vs 839 expected"}
```

| Category | Cause |
|---|---|
| `LIMIT_TRUNCATION` | LIMIT silently cut result (100 vs 839 rows) |
| `WRONG_TABLE` | Wrong table used (similar name, wrong data) |
| `WRONG_AGGREGATION` | MAX instead of SUM, AVG instead of SUM |
| `SCALAR_MISMATCH` | Single value differs from gold |
| `ROW_EXPLOSION` | 1:N join inflated row count |
| `SCHEMA_HALLUCINATION` | Invented table/column names |
| `FULL_TABLE_SCAN` | SELECT * with no WHERE/LIMIT |
| `TRIM_ON_NUMERIC` | TRIM() on numeric column (invalid SQL) |
| `UNSAFE_QUERY` | DDL/DML attempted |
| `CURRENCY_NOT_CLEANED` | Single REPLACE missed commas in `$1,234` |
| `NULL_IN_AGGREGATION` | AVG/SUM without IS NOT NULL |
| `JOIN_WITHOUT_FK` | JOIN with no valid foreign key |
| `FAITHFULNESS_DROP` | Narration not grounded in SQL result |

---

## Multi-Gold SQL

Evaluate against all valid SQL formulations, take the best score:

```python
from sqlas import TestCase

test_case = TestCase(
    question  = "Count active users",
    gold_sqls = [
        "SELECT COUNT(*) FROM users WHERE active = 1",
        "SELECT COUNT(*) FROM users WHERE status = 'active'",
        "SELECT COUNT(id) FROM users WHERE is_active = true",
    ],
)
```

---

## Hardness Classification

```python
from sqlas import auto_classify_hardness

auto_classify_hardness("SELECT COUNT(*) FROM users")
# → "easy"

auto_classify_hardness("SELECT u.id, SUM(o.total) FROM users u JOIN orders o ON u.id=o.user_id GROUP BY u.id HAVING SUM(o.total) > 1000")
# → "hard"
```

Follows BIRD benchmark criteria. Auto-set on every `evaluate()` call.

---

## Guardrail Pipeline

```python
from sqlas import GuardrailPipeline

pipeline = GuardrailPipeline(pii_columns=["email", "ssn", "password"])

pipeline.check_input("List every user's SSN")   # Stage 1: NL intent
pipeline.check_sql(generated_sql)               # Stage 2: AST + injection
pipeline.check_output(response, result_data)    # Stage 3: PII leakage
```

Detects: SQL injection, prompt injection, PII access, PII in response, DDL/DML attempts.

---

## Batch Evaluation & Reporting

```python
from sqlas import run_suite, generate_report, TestCase, WEIGHTS_V4, build_schema_info

tables, columns = build_schema_info(db_path="my.db")

results = run_suite(
    test_cases     = test_cases,
    agent_fn       = my_agent,
    llm_judge      = llm_judge,
    execute_fn     = execute_fn,
    valid_tables   = tables,
    valid_columns  = columns,
    weights        = WEIGHTS_V4,
    pass_threshold = 0.6,
)

print(generate_report(results, format="markdown"))  # paste into PR comments
print(generate_report(results, format="json"))      # artifact storage
```

---

## Spider / BIRD Benchmark

```python
from sqlas.benchmarks import run_spider_benchmark

results = run_spider_benchmark(
    agent_fn   = my_agent,
    llm_judge  = llm_judge,
    spider_dir = "./spider",
    n_samples  = 50,
    mlflow_run = True,
)
print(results["summary"]["overall_score"])
```

---

## Prompt Versioning

```python
from sqlas import PromptRegistry

registry = PromptRegistry()
registry.register("You are a SQL analyst...", version_id="v1")
registry.record("v1", scores)

status = registry.detect_regression("v1", window=50, threshold=0.05)
if status["regressed"]:
    for hint in status["hints"]:
        print(hint["hint"])
```

---

## LLM Judge Cache

```python
from sqlas import enable_judge_cache, clear_judge_cache

enable_judge_cache()       # identical prompts return cached result in CI
results = evaluate_batch(...)
clear_judge_cache()
```

---

## Observability

```python
from sqlas.integrations import log_all

log_all(results,
    mlflow_experiment = "sql-agent-v2",
    wandb_project     = "sql-evals",
    langsmith_project = "my-sql-agent",
)
```

---

## System Architecture

![SQL Agent System Architecture](../assets/architecture.png)

The right panel shows how SQLAS slots into a production SQL agent, evaluating execution accuracy, schema retrieval quality, safety, failure classification, and overall verdict across every query.

---

## Metrics Overview

| Dimension | Key Metrics |
|---|---|
| **Correctness** | Execution accuracy, exact match, multi-gold SQL, semantic equivalence, result set similarity |
| **SQL Quality** | SQL quality (LLM), schema compliance, complexity match, data scan efficiency |
| **Context (RAGAS)** | Context precision, recall, entity recall, noise robustness |
| **Response** | Faithfulness, answer relevance, completeness, fluency |
| **Agentic** | Steps efficiency, schema grounding, planning quality, tool use accuracy, plan compliance, first attempt success |
| **Safety** | Read-only compliance, SQL injection, prompt injection, PII access, PII leakage |
| **Production** | Execution success, VES efficiency, row explosion detection, empty result, result coverage |
| **Cache** | Cache hit score, tokens saved, few-shot examples used |
| **Visualization** | Chart spec validity, data alignment, LLM chart validation |

## Weight Profiles

| Profile | Metrics | Best for |
|---|---|---|
| `WEIGHTS` | 15 | Standard NL→SQL pipeline |
| `WEIGHTS_V2` | 20 | + RAGAS context quality |
| `WEIGHTS_V3` | 30 | + Guardrails + visualization |
| `WEIGHTS_V4` | 28 | + Agentic quality (ReAct agents) |

---

## License

MIT, by [thepradip](https://github.com/thepradip) · [pypi.org/project/sqlas](https://pypi.org/project/sqlas/)
