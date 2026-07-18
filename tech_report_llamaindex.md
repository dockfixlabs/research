# LlamaIndex Security Scan Report

**Date:** 2026-07-17  
**Codebase:** run-llama/llama_index (commit dbdaf89)  
**Scope:** 3,828 Python files (excl. tests/docs/.git)  
**Methodology:** Pattern-based static analysis + deep scanning of high-risk areas

---

## Executive Summary

Scanned the LlamaIndex monorepo for common security anti-patterns. Found **673 findings** across 10 categories. **2 confirmed code execution sinks**, **150 HTTP calls without timeout**, **35 Agent class implementations**, and **widespread use of f-strings in SQL execute()** that may pose injection risks. The codebase generally has good hygiene — YAML uses `safe_load` exclusively, most pickle usage is safe serialization, and the evaporate program has a proper AST-based sandbox validator.

## Findings Summary

| Category | Count | Severity |
|----------|-------|----------|
| Raw SQL execution | 384 | Medium-High |
| requests without timeout | 170 | Low-Medium |
| compile() usage | 54 | Low |
| Potential sensitive logging | 36 | Low |
| subprocess calls | 14 | Medium |
| eval() calls | 7 | Low (mostly `model.eval()`) |
| pickle serialize/dump | 4 | Low |
| pickle deserialize/load | 2 | Medium |
| exec() calls | 2 | High |
| Agent class files | 35 | Informational |

---

## Critical Finding #1: exec() in Evaporate Program Extractor (HIGH)

**File:** `llama-index-integrations/program/llama-index-program-evaporate/llama_index/program/evaporate/extractor.py`  
**Lines:** 360–361

```python
exec(fn_str, sandbox)
exec(f"__result__ = get_{function_field}_field(node_text)", sandbox)
```

LLM-generated Python code is executed via `exec()`. Mitigating factors:
- A `_validate_generated_code()` AST validator runs first (lines 160–192), blocking dunder access and restricting imports to an allowlist
- Execution is wrapped in a 1-second `time_limit()` context
- A sandbox dict is built via `_build_sandbox()` which uses `__import__` override

Despite mitigations, LLM-generated code is inherently unbounded. The AST validator is a defense layer, not a guarantee — `ast.walk` checks can be bypassed with creative code generation.

## Critical Finding #2: Unsafe pickle.load() in Two Integrations (MEDIUM)

**File:** `llama-index-integrations/indices/llama-index-indices-managed-bge-m3/.../base.py:157`
```python
index._multi_embed_store = pickle.load(
    open(Path(persist_dir) / "multi_embed_store.pkl", "rb")
)
```

**File:** `llama-index-integrations/vector_stores/llama-index-vector-stores-txtai/.../base.py:126`
```python
config = json.load(f) if jsonconfig else pickle.load(f)
```

Both load pickle files from disk without integrity verification. If an attacker can write to the persist directory (supply chain, dependency confusion, shared filesystem), RCE via pickle deserialization is trivially achievable.

## Critical Finding #3: Code Interpreter Tool — Arbitrary Code Execution (HIGH)

**File:** `llama-index-integrations/tools/llama-index-tools-code-interpreter/.../base.py`

The `CodeInterpreterToolSpec` explicitly provides LLM agents with `subprocess.run([sys.executable, "-c", code])`. The codebase documents this honestly:

```python
"""
WARNING: This tool provides the Agent access to the subprocess.run command.
Arbitrary code execution is possible on the machine running this tool.
This tool is not recommended to be used in a production setting.
"""
```

While documented, this tool is surfaced to agents and could be used by unaware developers in production. No opt-in safety flag or environment variable guard exists — any agent given this tool spec can run arbitrary Python.

## Finding #4: SQL Injection via f-strings in execute() (MEDIUM-HIGH)

Found **27 instances** of f-string interpolation directly in `.execute()` calls:

**Nebula graph store** (`nebula_graph_store.py`):
```python
self.execute(f"DESCRIBE SPACE {self._space_name}")
self.execute(f"DESCRIBE TAG `{tag_name}`")
self.execute(f"DESCRIBE EDGE `{edge_type_name}`")
```

These use `self._space_name`, `tag_name`, and `edge_type_name` which may be user-controlled if user-defined schema names flow through. In the `execute()` method (line ~248), these queries go directly to the database without parameterization.

Also found in Nebula property graph (line 142):
```python
self._client.execute(f"CLEAR SPACE {self._space};")
```

While most of the 384 raw SQL findings use SQLAlchemy parameterized queries (safe), these f-string interpolated queries bypass parameterization entirely.

## Finding #5: 170 HTTP Calls Without Timeout (MEDIUM)

Across the codebase, **170** `requests.get/post/put/delete` calls lack a `timeout` parameter. This means:
- Network hangs can block agent execution indefinitely
- In async contexts, this can exhaust the event loop
- Affects core embeddings, document loaders, API connectors, and all managed indices (DashScope, BGE-M3, etc.)

Examples:
- `llama-index-core`: image URL fetching (schema.py)
- `llama-index-integrations/embeddings`: HuggingFace, DeepInfra, YandexGPT embedding APIs
- `llama-index-integrations/indices`: DashScope managed index operations

## Finding #6: SSRF via f-string URL Injection (LOW-MEDIUM)

Three instances of f-string URLs in requests:
```python
# monsterapi/base.py:136
response = requests.get(f"{api_base}/models/info", headers=headers)
# box_api.py:262
resp = requests.get(url, headers={"Authorization": f"Bearer {access_token}"})
# pdb/utils.py:38
response = requests.get(f"{base_url}{pdb_id}")
```

The PDB reader (`pdb_id`) and Box API (`url`) are the most concerning as these accept external inputs.

## Additional Observations

- **Agent surfaces:** 35 Agent class implementations across the codebase — large attack surface for prompt injection and tool misuse
- **Pickle serialization in core:** `schema.py` uses `pickle.dumps()` for JSON serialization validation (safe, only verifying serializability)
- **YAML hygiene is good:** All `yaml.load()` calls use `safe_load` — no arbitrary object deserialization
- **No hardcoded API keys found** — the 93 "hardcoded_secret" hits were false positives (e.g., `start_token="KEYWORDS:"`)
- **compile() usage** (54 instances) is mostly `re.compile()` inside SEC filings reader — benign
- **14 subprocess calls** — mostly in development tooling (`llama-dev/`) and the documented code interpreter tool

## Risk Prioritization

1. **P0 — Immediate fix**: Evaporate `exec()` sandbox hardening — add explicit import blocklist (importlib, types, etc.)
2. **P1 — High**: `pickle.load()` in BGE-M3 and txtai integrations — add integrity verification or switch to JSON
3. **P2 — High**: Code interpreter tool — add environment-variable opt-in gate (`LLAMA_INDEX_ALLOW_CODE_INTERPRETER=1`)
4. **P3 — Medium**: Parameterize Nebula graph store f-string queries
5. **P4 — Medium**: Add `timeout=` to all `requests.*` calls (default 30s reasonable)

---

*Report generated by automated pattern analysis of 3,828 Python source files. Manual review recommended for all high-severity findings.*
