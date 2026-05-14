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

**The golden rule:** `Study` only ever talks to the ABCs — never to concrete
implementations. This is the Strategy Pattern applied systematically across
the entire framework.

---

## The 4 Key Modules

| Module | Role | ABC |
|---|---|---|
| `samplers/` | Decides *which* hyperparameters to try next | `BaseSampler` |
| `pruners/` | Decides *when* to stop a trial early | `BasePruner` |
| `storages/` | Decides *where* to persist results | `BaseStorage` |
| `study/` | Orchestrates all three — the central coordinator | *(none)* |

---

## 6 Architectural Views

| # | View | Kruchten | Question answered | Folder | Status |
|---|---|---|---|---|---|
| 1 | Module Decomposition | Development View | How is the code organized statically? | `views/view-1-module-decomposition/` | ✅ Step 2 |
| 2 | Component & Connector | Process View | How do components interact at runtime? | `views/view-2-component-connector/` | ✅ Step 3 |
| 3 | Class & Inheritance | Logical View | How is the type hierarchy structured? | `views/view-3-class-inheritance/` | ✅ Step 3 |
| 4 | Deployment | Physical View | How is Optuna deployed in real systems? | `views/view-4-deployment/` | ✅ Step 6 |
| 5 | Historical Evolution | *(bonus)* | How has the architecture evolved 2018→2026? | `views/view-5-historical-evolution/` | ✅ Step 6 |
| 6 | Dependency & Coupling | *(bonus)* | Who depends on what, and how tightly? | `views/view-6-dependency-coupling/` | ✅ Step 6 |

---

## Architectural Decisions — 7 Categories

Full tracker →
[`architectural-decisions/patterns-tactics-qa.md`](architectural-decisions/patterns-tactics-qa.md)

| Category | Decision | QA | Defect |
|---|---|---|---|
| Allocation of Responsibilities | Study = sole orchestrator | Usability | God Object — violates SRP |
| Coordination Model | Sync phases 1–4 + async callbacks phase 5 | Modifiability | Slow callback = trial latency |
| Data Model | FrozenTrial = immutable DTO | Reliability | deepcopy overhead per trial |
| Management of Resources | _CachedStorage wraps all backends | Performance | Stale read risk across threads |
| Mapping Among Architectural Elements | Plugin ABCs decouple core from implementations | Extensibility | BaseStorage contract too heavy (17 methods) |
| Choice of Technology | SQLAlchemy + gRPC + protobuf | Scalability | SQLite not parallel-safe |
| Quality Attribute Tradeoffs | God Object accepted for API simplicity | Usability | SRP violation — 1751 lines |

---

## Quality Attributes — SMART Format

| QA | Stimulus | Measure | Defect |
|---|---|---|---|
| **Extensibility** | Add custom sampler | 0 lines core modified · < 1h | — |
| **Scalability** | 1000 trials · 10 machines | Linear throughput scaling | RDBStorage bottleneck at high worker count |
| **Testability** | CI/CD full test suite | < 1s · zero dependencies | — |
| **Modifiability** | Deprecate UniformDistribution | 0 breaking changes | — |
| **Reliability** | Worker killed mid-trial | 0 data loss · recovery < next trial | Heartbeat thread overhead |
| **Interoperability** | MLflow logging per trial | 3 lines user code · 0 core modified | — |

---

## Architectural Patterns

| Pattern | QA Addressed | Defect |
|---|---|---|
| Microkernel | Extensibility · Modifiability | Performance penalty per layer crossing |
| Layered (5 layers) | Modifiability · Extensibility | Layer violations exist (study/→_heartbeat) |
| Shared Data | Scalability · Reliability | RDBStorage = central bottleneck |
| Broker (GrpcProxy) | Scalability · Performance | +2 network hops · experimental |
| Client-Server | Scalability · Separation | Server = potential SPOF |
| Publish-Subscribe | Interoperability · Modifiability | Sync — slow callback = latency |
| Repository | Testability · Portability | 17 abstract methods = heavy contract |
| Pipeline (_run_trial) | Modifiability · Extensibility | Synchronous only — no streaming |
| Multi-Tier (gRPC mode) | Scalability · Separation | 3 components to deploy |

---

## Architecture Tactics

| Tactic | QA Addressed | Defect |
|---|---|---|
| Abstract interfaces (ABCs) | Extensibility | — |
| Two-level ABC hierarchy | Extensibility | — |
| Dependency injection | Testability | — |
| Separate concerns (InMemoryStorage) | Testability | — |
| Dedicated test fixtures (testing/) | Testability | — |
| Introduce concurrency (ThreadPoolExecutor) | Scalability | Python GIL limits CPU parallelism |
| Use a broker (GrpcStorageProxy) | Scalability | Experimental — operational complexity |
| constant_liar (TPESampler) | Scalability | Approximate — not exact |
| Heartbeat (_heartbeat.py) | Reliability | Periodic thread overhead |
| Granular persistence (set_trial_param/suggest) | Reliability | — |
| Immutable data in callbacks (deepcopy) | Reliability | Memory overhead per trial |
| Retry on failure (RetryFailedTrialCallback) | Reliability | — |
| Cache (_CachedStorage) | Performance | Stale read risk across threads |
| Lazy loading (_LazyImport) | Performance | Import errors hidden until access |
| Anticipate expected changes (ABCs) | Modifiability | — |
| Deprecation policy (_deprecated.py) | Modifiability | — |
| Façade (create_study()) | Usability | — |
| Shim layer (integration/) | Interoperability | — |

---

## Known Weaknesses

| # | Weakness | Location | Impact |
|---|---|---|---|
| 1 | God Object | `study/study.py` — 1751 lines, 15+ methods | Violates SRP |
| 2 | No native async | `_optimize.py` — ThreadPoolExecutor only | GIL limits CPU parallelism |
| 3 | RDBStorage bottleneck | `storages/_rdb/storage.py` | Central bottleneck at scale |
| 4 | SQLite not parallel-safe | `storages/_rdb/` | Lock contention risk |
| 5 | Layer violations | `study/` → `_heartbeat`, `visualization/` → `samplers/` | Breaks layered contract |
| 6 | LSP violation | `isinstance(storage, _CachedStorage)` in codebase | Tight concrete coupling |
| 7 | BaseStorage contract too heavy | `storages/_base.py` — 17 abstract methods | Custom storage = expensive |
| 8 | Trial → Study tight coupling | `trial/_trial.py` L.55 `self.study = study` | Trial cannot exist independently |

---



## References

- Akiba et al. (2019). *Optuna: A Next-generation HPO Framework.* KDD 2019.
- Bass, Clements, Kazman (2012). *Software Architecture in Practice, 3rd Ed.*
- Kruchten, P. (1995). *Architectural Blueprints — The 4+1 View Model.* IEEE Software.
- Optuna Documentation. https://optuna.readthedocs.io
- Optuna GitHub. https://github.com/optuna/optuna
