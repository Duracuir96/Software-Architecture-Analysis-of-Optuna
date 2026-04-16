# View 1 — Module Decomposition

**Type:** Module View (Kruchten Development View)  
**Step produced:** Step 2  
**Diagram:** :
<img width="500" height="400" alt="image" src="https://github.com/user-attachments/assets/9a97b416-30b0-486e-b067-390a61a9d32b" />


---

## Question answered

> How is the `optuna/` codebase organized statically?
> What are the modules, their responsibilities, and their dependencies?

---

## Method

- Cloned repo at HEAD (April 2025)
- Ran `find optuna/ -name "*.py"` — 200+ files analyzed
- Counted lines per module with `wc -l`
- Extracted real dependency graph by reading `import` statements in each `__init__.py`
- Verified lazy-loading via `_LazyImport` in `optuna/__init__.py`

---

## Module metrics (real data)

| Module | Files | Lines | Layer |
|---|---|---|---|
| `samplers/` | 37 | 6,665 | Plugin |
| `storages/` | 31 | 8,058 | Plugin |
| `visualization/` | 30 | 5,361 | Optional |
| `study/` | 10 | 2,873 | Core |
| `trial/` | 6 | 1,759 | Core |
| `testing/` | 11 | 2,148 | Infrastructure |
| `terminator/` | 8 | 958 | Optional |
| `importance/` | 10 | 1,396 | Optional |
| `artifacts/` | 10 | 746 | Optional |
| `integration/` | 24 | 402 | Optional |
| `search_space/` | 3 | 231 | Infrastructure |
| `_gp/` | 9 | 1,638 | Private |
| `_hypervolume/` | 4 | 520 | Private |

---


```markdown
## 5-layer architecture

┌──────────────────────────────────────────────────────┐
│  PUBLIC API   __init__.py                            │
│  create_study() · Study · Trial · TrialPruned        │
├──────────────────────────────────────────────────────┤
│  CORE  (stable — changes rarely)                     │
│  study/  2,873 lines    trial/  1,759 lines          │
│  distributions.py  765 lines                         │
├──────────────────────────────────────────────────────┤
│  PLUGIN LAYER  (interchangeable via ABCs)            │
│  samplers/  6,665 lines    BaseSampler  (3 abstract) │
│  pruners/   1,530 lines    BasePruner   (1 abstract) │
│  storages/  8,058 lines    BaseStorage (17 abstract) │
├──────────────────────────────────────────────────────┤
│  OPTIONAL  (zero impact on core)                     │
│  integration/  visualization/  artifacts/            │
├──────────────────────────────────────────────────────┤
│  INFRASTRUCTURE                                      │
│  testing/  _gp/  _hypervolume/  _deprecated.py       │
└──────────────────────────────────────────────────────┘
```

---

## Dependency graph (extracted from imports)

```
study/    → samplers/ (via BaseSampler only)
study/    → pruners/  (via BasePruner only)
study/    → storages/ (via BaseStorage only)
study/    → trial/
trial/    → distributions.py
samplers/ → distributions.py, search_space/, _gp/, _hypervolume/
pruners/  → trial/ (TrialState), study/ (StudyDirection)
storages/ → distributions.py, trial/, study/ (FrozenStudy)
integration/ → nothing from optuna core at load time
visualization/ → study/, trial/, samplers/ (read only)
```

**Dependency rule verified:** Core never imports concrete plugin implementations.

---

## Key observations

### 1. The underscore convention is enforced rigorously

All implementation files are prefixed `_` (`_median.py`, `_optimize.py`, `_cached_storage.py`). Only `__init__.py` files expose the public API. This is Python's encapsulation convention applied consistently.

### 2. `storages/` is heavier than `samplers/` (8,058 vs 6,665 lines)

Persistence is architecturally more complex than the optimization algorithms. The `_rdb/alembic/versions/` folder contains 10 migration files — a direct trace of schema evolution since v0.9.0.

### 3. Lazy-loading strategy for heavy optional dependencies

`visualization`, `importance`, and `artifacts` are lazy-loaded from the root `__init__.py` via `_LazyImport`. Plotly and scikit-learn are never imported at startup — only when the user calls a visualization function.

### 4. `testing/` is a first-class architectural citizen

A dedicated test module means any custom `BaseSampler` implementation can be validated against the standard Optuna test suite automatically. This makes testability a structural property, not just a convention.

---

## Architectural insights for the report

| Tactic | Implementation |
|--------|----------------|
| **Extensibility** | Plugin layer is fully isolated behind ABCs — adding a new sampler requires zero changes outside `samplers/`. |
| **Usability** | `__init__.py` exposes only 5 names at top level — the complexity of 200+ files is completely hidden from the user. |
| **Performance** | Lazy imports keep startup time minimal regardless of installed optional dependencies. |
```

---
