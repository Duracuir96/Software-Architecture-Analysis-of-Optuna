# Software Architecture Analysis of Optuna

**Course:** Software Architecture — Wuhan University  
**Instructor:** Prof. Peng Liang  
**Student:** VOLDI少坎  
**Subject repo:** https://github.com/optuna/optuna

---

## Research Question

> *How does Optuna's architecture achieve maximum extensibility and scalability
> while keeping its core API simple enough for any ML practitioner to use?*

---

## What is Optuna?

Optuna is a define-by-run hyperparameter optimization framework developed by
Preferred Networks (Japan), published at KDD 2019. It allows ML engineers to
optimize model hyperparameters automatically using algorithms like TPE and
CMA-ES, with native support for distributed execution and pruning of
unpromising trials.

**Core architectural principle:**

> Stable core (`Study` + `Trial`) + interchangeable plugins (`Sampler` +
> `Pruner` + `Storage`) via explicit Abstract Base Classes = simple API +
> infinite extensibility.

---

## The 4 Key Modules

| Module | Role | ABC |
|---|---|---|
| `samplers/` | Decides *which* hyperparameters to try next | `BaseSampler` |
| `pruners/` | Decides *when* to stop a trial early | `BasePruner` |
| `storages/` | Decides *where* to persist results | `BaseStorage` |
| `study/` | Orchestrates all three — the central coordinator | *(none)* |

**The golden rule:** `Study` only ever talks to the ABCs — never to concrete
implementations. This is the Strategy Pattern applied systematically across
the entire framework.

---

## 6 Architectural Views

Each view lives in its own folder with a `notes.md` and a `diagram.png`.

| # | View | Question answered | Folder | Status |
|---|---|---|---|---|
| 1 | Module Decomposition | How is the code organized statically? | `views/view-1-module-decomposition/` | ✅ Step 2 |
| 2 | Component & Connector | How do components interact at runtime? | `views/view-2-component-connector/` | ✅ Step 3 |
| 3 | Class & Inheritance | How is the type hierarchy structured? | `views/view-3-class-inheritance/` | ✅ Step 3 |
| 4 | Deployment | How is Optuna deployed in real systems? | `views/view-4-deployment/` | ⬜ Step 6 |
| 5 | Historical Evolution | How has the architecture evolved 2018→2024? | `views/view-5-historical-evolution/` | ⬜ Step 6 |
| 6 | Dependency & Coupling | Who depends on what, and how tightly? | `views/view-6-dependency-coupling/` | ⬜ Step 6 |

---

## Architectural Decisions (updated each step)

Full tracker →
[`architectural-decisions/patterns-tactics-qa.md`](architectural-decisions/patterns-tactics-qa.md)

### Design Patterns (current)

| Pattern | Location | Evidence |
|---|---|---|
| **Strategy ×3** | `study/study.py` L.90–94 | `self.sampler`, `self.pruner`, `self._storage` — all injected interfaces |
| **Template Method** | `samplers/_base.py` L.69–166 | `BaseSampler` defines the 3-step sampling sequence skeleton |
| **Factory** | `optuna/__init__.py` | `create_study()` wires all 4 components together |
| **Observer** | `study/_optimize.py` L.176 | `callback(study, deepcopy(frozen_trial))` post-trial |
| **Null Object** | `pruners/_nop.py` | `NopPruner.prune() → False` — disables pruning without conditionals |
| **Decorator** | `pruners/_patient.py` | `PatientPruner` wraps any `BasePruner` instance |

### Tactics (current)

| Quality Attribute | Tactic | Implementation |
|---|---|---|
| Extensibility | Abstract interfaces | `BaseSampler`, `BasePruner`, `BaseStorage` |
| Testability | Dependency injection | `Study.__init__` receives all 3 dependencies |
| Testability | Separate concerns | `InMemoryStorage` for tests — zero infrastructure |
| Scalability | Introduce concurrency | `n_jobs` → `ThreadPoolExecutor` in `_optimize.py` |
| Scalability | Use a broker | `GrpcStorageProxy` between workers and DB |
| Reliability | Heartbeat | `storages/_heartbeat.py` detects dead workers |
| Performance | Cache | `_CachedStorage` reduces DB round-trips |
| Modifiability | Anticipate expected changes | Plugin ABCs are explicit extension contracts |
| Modifiability | Deprecation policy | 2-version grace period via `_deprecated.py` |
| Usability | Façade | `create_study()` hides all wiring complexity |

### Known Weaknesses (current)

| Weakness | Location | Impact |
|---|---|---|
| God Object | `study/study.py` — 1751 lines, 15+ public methods | Violates SRP, potential maintenance bottleneck |
| No native async | `_optimize.py` — ThreadPoolExecutor only | Python GIL limits true CPU parallelism |
| SQLite not parallel-safe | `storages/_rdb/` | Risk of lock contention if misconfigured |
| RDBStorage bottleneck | `storages/_rdb/storage.py` — 1224 lines | Central DB bottleneck at high worker count |

---

## Progress

### ✅ Step 1 — Setup & Initial Exploration

- [x] Read official Optuna documentation
- [x] Cloned repo, explored full directory structure
- [x] Read Akiba et al., KDD 2019
- [x] Identified 4 key modules: `samplers/`, `pruners/`, `storages/`, `study/`
- [x] Identified 6 architectural views to produce
- [x] Started iterative architectural decisions tracker

**Key insight:**
> The three ABCs (`BaseSampler`, `BasePruner`, `BaseStorage`) are the central
> architectural contract of Optuna. Everything else is swappable.

---

### ✅ Step 2 — View 1: Module Decomposition

- [x] Analyzed full `optuna/` directory tree — 15 modules, 5 architectural layers identified
- [x] Classified modules: Core / Plugin / Optional / Infrastructure / Private
- [x] Extracted real dependency graph from import statements
- [x] Identified lazy-loading strategy (`_LazyImport`) for heavy dependencies
- [x] Produced Module Decomposition diagram → `views/view-1-module-decomposition/`
- [x] Updated architectural decisions tracker

**Key insight:**
> The underscore convention (`_median.py`, `_optimize.py`) is enforced
> rigorously — only `__init__.py` files expose the public API.
> `storages/` is heavier than `samplers/` (8,058 vs 6,665 lines) —
> persistence is architecturally more complex than the algorithms.

---

### ✅ Step 3 — Views 2 & 3: C&C + Class Inheritance

- [x] Traced full trial lifecycle — 5 phases, 18 connectors documented with exact signatures
- [x] Identified Observer Pattern in callbacks (`deepcopy` enforcement verified)
- [x] Identified Decorator Pattern in `PatientPruner`
- [x] Identified multiple inheritance: `RDBStorage(BaseStorage, BaseHeartbeat)`
- [x] Identified `BaseGASampler` as intermediate ABC — 2-level hierarchy
- [x] Verified ABC enforcement by live Python tests (TypeError messages captured)
- [x] Documented `TYPE_CHECKING` pattern — avoids circular imports at runtime
- [x] Documented `get_storage()` as Factory + Adapter combined
- [x] Produced C&C Runtime diagram → `views/view-2-component-connector/`
- [x] Produced Class & Inheritance diagram → `views/view-3-class-inheritance/`
- [x] Updated architectural decisions tracker

**Key insight:**
> `Storage` is called a minimum of 4 times per trial — at creation,
> per parameter suggested, per intermediate value reported, and at
> final state. This granular persistence is a deliberate reliability
> decision: no data is lost even if a worker crashes mid-trial.
> The heartbeat mechanism in `storages/_heartbeat.py` confirms this
> was an explicit architectural concern, not an accident.

---

## References

- Akiba et al. (2019). *Optuna: A Next-generation HPO Framework.* KDD 2019.
- Bass, Clements, Kazman (2012). *Software Architecture in Practice, 3rd Ed.*
- Kruchten, P. (1995). *Architectural Blueprints — The 4+1 View Model.* IEEE Software.
- Optuna Documentation. https://optuna.readthedocs.io
- Optuna GitHub. https://github.com/optuna/optuna
