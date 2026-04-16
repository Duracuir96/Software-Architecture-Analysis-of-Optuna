# Architectural Decisions — Iterative Tracker

**Project:** Software Architecture Analysis of Optuna  
**Updated:** each step, as new evidence is found in the codebase

---

## How to read this file

Every entry has:
- a **step of discovery** (when it was found)
- a **location in the code** (file + evidence)
- a **architectural justification** (why it matters)

Nothing is written here without proof in the code.

---

## 1. Design Patterns

| # | Pattern | Location | Code Evidence | Architectural Justification | Step |
|---|---|---|---|---|---|
| 1 | **Strategy** | `study/study.py` L.93 | `self.sampler = sampler or TPESampler()` | Study delegates to an interchangeable algorithm — swapping sampler = zero core change | S1 |
| 2 | **Strategy** | `study/study.py` L.94 | `self.pruner = pruner or MedianPruner()` | Same pattern for pruning strategy | S1 |
| 3 | **Strategy** | `study/study.py` L.90 | `self._storage = storages.get_storage(storage)` | Same pattern for persistence backend | S1 |
| 4 | **Template Method** | `samplers/_base.py` L.69–166 | `infer_relative_search_space()` → `sample_relative()` → `sample_independent()` | ABC defines the sampling sequence skeleton — subclasses fill the steps | S1 |
| 5 | **Factory** | `optuna/__init__.py` | `create_study()` | Single entry point that wires Study + Sampler + Pruner + Storage — hides construction complexity | S1 |
| 6 | **Null Object** | `pruners/_nop.py` | `NopPruner.prune() → return False` | Disables pruning without any `if` in Study — clean default behavior | S1 |
| 7 | **Observer** | `study/_optimize.py` L.174–177 | `for cb in callbacks: cb(study, copy.deepcopy(frozen_trial))` | Post-trial callbacks receive immutable deepcopy — extensions cannot corrupt trial state | S3 |
| 8 | **Decorator** | `pruners/_patient.py` L.17 | `class PatientPruner(BasePruner): self._wrapped_pruner: BasePruner` | Implements AND contains a BasePruner — adds patience behavior without modifying wrapped pruner | S3 |
| 9 | **Factory + Adapter** | `storages/__init__.py` | `get_storage(None\|str\|BaseStorage) → BaseStorage` | Normalizes 3 input forms into 1 — hides RDBStorage+_CachedStorage wiring | S3 |

---

## 2. Quality Attributes

| # | QA | Definition in Optuna's context | Code Evidence | Step |
|---|---|---|---|---|
| 1 | **Extensibility** | Add a custom sampler/pruner/storage without touching core | The 3 ABCs — `BaseSampler`, `BasePruner`, `BaseStorage` | S1 |
| 2 | **Testability** | Test any objective function without a database or infrastructure | `InMemoryStorage` default + `optuna/testing/` module | S1 |
| 3 | **Scalability** | Go from 1 local trial to 1000 distributed trials without changing user code | Single `storage=` param — `InMemory` → `RDB` → `gRPC` | S1 |
| 4 | **Modifiability** | Change internal components without breaking the public API | `_deprecated.py` — 2-version grace period policy | S1 |
| 5 | **Interoperability** | Integrate with PyTorch, MLflow, LightGBM in a few lines | `integration/` module — isolated, optional, callback-based | S1 |
| 6 | **Usability** | Simple API for any ML practitioner | `create_study()` + `study.optimize()` — 2 lines to run HPO | S1 |
| 7 | **Reliability** | Workers dying mid-trial do not lose data | `storages/_heartbeat.py` + granular `set_trial_param()` per parameter | S3 |

---

## 3. Architecture Tactics

| # | QA Addressed | Tactic | Implementation | File | Step |
|---|---|---|---|---|---|
| 1 | Extensibility | **Use abstract interfaces** | `BaseSampler`, `BasePruner`, `BaseStorage` as sole contact points | `samplers/_base.py`, `pruners/_base.py`, `storages/_base.py` | S1 |
| 2 | Testability | **Dependency injection** | `Study.__init__` receives all 3 dependencies — never instantiates them | `study/study.py` L.85–94 | S1 |
| 3 | Testability | **Separate concerns** | `InMemoryStorage` = test implementation, zero infrastructure | `storages/_in_memory.py` | S1 |
| 4 | Scalability | **Introduce concurrency** | `n_jobs` in `optimize()` → `ThreadPoolExecutor` | `study/_optimize.py` | S1 |
| 5 | Scalability | **Use a broker** | `GrpcStorageProxy` sits between workers and DB | `storages/_grpc/` | S1 |
| 6 | Reliability | **Heartbeat** | Workers ping storage periodically — dead trials marked FAIL | `storages/_heartbeat.py` | S1 |
| 7 | Performance | **Cache** | `_CachedStorage` wraps any backend — reduces DB round-trips | `storages/_cached_storage.py` | S1 |
| 8 | Modifiability | **Anticipate expected changes** | Plugin points explicitly designed — ABCs are the extension contract | `samplers/`, `pruners/`, `storages/` | S1 |
| 9 | Modifiability | **Deprecation policy** | 2-version grace period before removal | `optuna/_deprecated.py` | S1 |
| 10 | Usability | **Façade** | `create_study()` hides all wiring complexity | `optuna/__init__.py` | S1 |
| 11 | Reliability | **Granular persistence** | `set_trial_param()` called immediately after each `suggest_*()` — not at end of trial | `trial/_trial.py` L.643 | S3 |
| 12 | Reliability | **Immutable data in callbacks** | `copy.deepcopy(frozen_trial)` before passing to callbacks | `study/_optimize.py` L.176 | S3 |
| 13 | Extensibility | **Two-level ABC hierarchy** | `BaseGASampler(BaseSampler)` adds GA-specific contract without modifying root | `samplers/_ga/_base.py` | S3 |
| 14 | Scalability | **Multiple inheritance mixin** | `RDBStorage(BaseStorage, BaseHeartbeat)` combines persistence + reliability contracts | `storages/_rdb/storage.py` L.106 | S3 |
| 15 | Performance | **Lazy loading** | `visualization`, `importance`, `artifacts` loaded only when accessed | `optuna/__init__.py` — `_LazyImport` | S2 |
| 16 | Testability | **Dedicated test fixtures** | `optuna/testing/` module provides standard suites for all ABCs | `testing/samplers.py`, `testing/storages.py` | S2 |

---

## 4. Formal QA Scenarios (Bass/Clements format)

*Format: Source → Stimulus → Environment → Artifact → Response → Response Measure*

| # | QA | Scenario | Step |
|---|---|---|---|
| 1 | **Extensibility** | A ML engineer needs a genetic algorithm sampler. They subclass `BaseSampler`, implement 3 methods, pass it to `create_study(sampler=...)`. **0 lines of core modified. Time: < 1h.** | S1 |
| 2 | **Testability** | A developer runs CI/CD. They call `create_study()` with no storage. `InMemoryStorage` activates. Tests run in RAM. **Execution time < 1s. No DB required.** | S1 |
| 3 | **Scalability** | A team launches 1000 trials across 10 machines. They set `storage="postgresql://..."`. Workers run independently. Storage coordinates. **Throughput scales linearly with worker count.** | S1 |
| 4 | **Modifiability** | Optuna deprecates `UniformDistribution` in v3.x. A warning is emitted for 2 versions. Migration is documented. **0 breaking change for existing users.** | S1 |
| 5 | **Interoperability** | A user wants MLflow logging per trial. They add `MLflowCallback` from `integration/`. 3 lines in their code. **0 modification to core.** | S1 |
| 6 | **Reliability** | A worker process is killed mid-trial on step 7 of 20. The heartbeat thread stops. On the next trial creation, `fail_stale_trials()` marks the dead trial as FAIL. **Parameters from steps 1–7 are fully persisted. No data loss.** | S3 |

---

## 5. Known Weaknesses

| # | Weakness | Location | Code Evidence | Impact | Step |
|---|---|---|---|---|---|
| 1 | **God Object** | `study/study.py` | 1751 lines, 15+ public methods | Violates SRP — Study orchestrates, exposes results, AND implements ask/tell | S1 |
| 2 | **No native async** | `study/_optimize.py` | `ThreadPoolExecutor` only | Python GIL limits true CPU parallelism — no `asyncio` support | S1 |
| 3 | **SQLite not parallel-safe** | `storages/_rdb/` | Mentioned in official FAQ | Lock contention risk when misconfigured for distributed use | S1 |
| 4 | **RDBStorage bottleneck** | `storages/_rdb/storage.py` | 1224 lines — heaviest storage file | Central DB becomes bottleneck at very high worker count without gRPC proxy | S1 |
| 5 | **Trial holds direct Study ref** | `trial/_trial.py` L.55 | `self.study = study` | Trial is tightly coupled to Study — cannot exist independently, complicates testing | S3 |

---

## 6. Open Questions (to investigate in later steps)

| # | Question | Status | Target Step |
|---|---|---|---|
| 1 | Does `study/_optimize.py` use Observer pattern for callbacks? | ✅ Confirmed — `deepcopy(frozen_trial)` in L.176 | S3 |
| 2 | How does `_CachedStorage` handle cache invalidation across threads? | 🔄 Partially — `_thread_local.cached_all_trials = None` resets per trial | S3 |
| 3 | What triggered the `study/` refactor from single file to package? (commit history) | ⬜ To investigate | S5 |
| 4 | Is `integration/` truly zero-coupling — does it import from `samplers/` or `pruners/`? | ✅ Confirmed — only imports `_imports._INTEGRATION_IMPORT_ERROR_TEMPLATE` | S2 |
| 5 | What is the exact memory layout of `_CachedStorage` across parallel workers? | ⬜ To investigate | S6 |
| 6 | Does NSGA-II's `before_trial` hook affect trial ordering in `RDBStorage`? | ⬜ To investigate | S6 |
