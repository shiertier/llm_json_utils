# json-repair-rust

Fast, minimal JSON "repair" implemented in Rust and exposed to Python via [PyO3](https://github.com/PyO3/pyo3) and [maturin](https://github.com/PyO3/maturin).

> 简体中文文档请见：[README.zh-CN.md](README.zh-CN.md)

## What this library does

- **Repairs only simple, structural issues** that are common in LLM / log output:
  - Auto‑closes truncated objects/arrays at end of input  
    (e.g. `{"a": 1` → behaves like `{"a": 1}`).
  - Allows **trailing commas** in objects/arrays  
    (e.g. `{"a": 1,}` or `[1, 2,]`).
- **Accepts comments** and ignores them during parsing:
  - Line comments: `// ...` and `# ...`
  - Block comments: `/* ... */`
- **Parses numbers like Python**:
  - Normal integers → `int`
  - Floats → `float`
  - Integers that overflow `i64` are delegated to Python’s `int()`
    so you keep arbitrary‑precision integers.
- **Preserves unknown escapes and broken `\u` sequences**:
  - `"\w"` stays as `\\w` in the result.
  - `"\u123z"` becomes the literal text `"\\u123z"` instead of losing `"123z"`.

## What this library intentionally does NOT do

- **No “AI‑style guessing” or content magic**:
  - Does *not* invent missing keys, colons, or quotes.
  - Does *not* try to interpret random text as JSON.
  - On real structural errors (e.g. missing `:` between key/value), it raises `ValueError`.
- **No silent data changes**:
  - Does *not* coerce non‑JSON literals (e.g. `None`, `NaN`) into JSON.
  - Does *not* drop unknown escape sequences or malformed `\u` data.
- **No multiple top‑level documents**:
  - It parses **one** JSON value from the input and ignores any trailing garbage instead of trying to stitch multiple documents together.

## Why it is designed this way

- **Determinism over “smart” heuristics**  
  A JSON repair tool should fix clearly‑defined structural glitches, not guess user intent. Guessing makes behavior non‑deterministic and hides upstream bugs (bad prompts, bad models, bad producers).

- **Safety over convenience**  
  Swallowing characters, normalizing exotic literals, or silently fixing arbitrary text looks convenient but amounts to **data corruption**. This library prefers to:
  - repair what is obviously safe (truncation, trailing commas, comments),
  - and fail loudly on everything else.

- **Performance and simplicity**  
  The parser is a small, linear, recursive‑descent implementation in Rust. It:
  - avoids complex state machines,
  - avoids per‑character heap allocations,
  - and exposes a single `repair_json(str) -> Any` API to Python.

## Python usage

```python
from json_repair_rust import repair_json

obj = repair_json('{"a": 1, "b": [1,2,],}')
assert obj == {"a": 1, "b": [1, 2]}
```

## Build locally

```bash
pip install maturin
maturin develop
python -c "from json_repair_rust import repair_json; print(repair_json('{\"x\": 1,}'))"
```
