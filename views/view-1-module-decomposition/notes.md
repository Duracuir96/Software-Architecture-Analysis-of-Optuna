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

## 5-layer architecture
