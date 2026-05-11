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
| 4 | Deployment | How is Optuna deployed in real systems? | `views/view-4-deployment/` | ✅ Step 6 |
| 5 | Historical Evolution | How has the architecture evolved 2018→2026? | `views/view-5-historical-evolution/` | ✅ Step 6 |
| 6 | Dependency & Coupling | Who depends on what, and how tightly? | `views/view-6-dependency-coupling/` | ✅ Step 6 |

---

## Architectural Decisions (updated each step)

Full tracker →
[`architectural-decisions/patterns-tactics-qa.md`](architectural-decisions/patterns-tactics-qa.md)

---

### Architectural Patterns

> Distinct from Design Patterns — these operate at **system level**.
> Source: Bass, Clements, Kazman — *Software Architecture in Practice, 3rd Ed.*

| # | Pattern | Where in Optuna | QA Addressed | Key Tradeoff |
|---|---|---|---|---|
| 1 | **Layered** | 5-layer structure: API / Core / Plugin / Optional / Infra | Modifiability · Extensibility | Performance penalty per layer crossing · 3 violations |
| 2 | **Shared Data** | All workers share `BaseStorage` (single source of truth) | Scalability · Reliability | `RDBStorage` central bottleneck at high worker count |
| 3 | **Broker** | `GrpcStorageProxy` between 1000+ workers and PostgreSQL | Scalability · Performance | +2 network hops · experimental since v4.2.0 |
| 4 | **Client-Server** | `Study` (client) → `BaseStorage` (server) — unidirectional | Scalability · Separation | Server = potential single point of failure |
| 5 | **Publish-Subscribe** | `study.optimize(callbacks=[...])` post-trial events | Interoperability · Modifiability | Synchronous — slow callback = trial latency |
| 6 | **Microkernel** | Stable kernel (study/+trial/+ABCs) + lazy-loaded plugins | Extensibility · Performance | Import errors hidden until access via `_LazyImport` |
| 7 | **Repository** | `BaseStorage` — 18 abstract methods, full persistence contract | Testability · Portability | Heaviest contract — 18 methods required for custom storage |
| 8 | **Pipeline (Pipe & Filter)** | 5-filter sequence in `_run_trial()`: suggest→evaluate→prune→persist→callback | Modifiability · Extensibility | Synchronous only — no streaming between filters |
| 9 | **Multi-Tier** | 3 tiers in gRPC mode: Worker / GrpcProxy / PostgreSQL | Scalability · Separation | Operational complexity — 3 components to deploy and maintain |

---

### Design Patterns

> Solutions at **class/object level** — distinct from Architectural Patterns.

| Pattern | Location | Evidence | Key Tradeoff |
|---|---|---|---|
| **Strategy ×3** | `study/study.py` L.90–94 | `self.sampler` · `self.pruner` · `self._storage` — all injected interfaces | Defaults instantiated unconditionally even if overridden |
| **Template Method** | `samplers/_base.py` L.69–166 | `infer_relative_search_space()` → `sample_relative()` → `sample_independent()` | Hidden coupling via `before_trial()` / `after_trial()` hooks |
| **Factory** | `optuna/__init__.py` | `create_study()` wires all 4 components | Silently wraps `RDBStorage` in `_CachedStorage` — user unaware |
| **Observer** | `study/_optimize.py` L.176 | `callback(study, deepcopy(frozen_trial))` post-trial | Synchronous — no timeout mechanism |
| **Null Object** | `pruners/_nop.py` | `NopPruner.prune() → False` — no conditionals in Study | None |
| **Decorator** | `pruners/_patient.py` | `PatientPruner` wraps any `BasePruner` | Wrapping chain obscures actual pruner type |
| **Factory + Adapter** | `storages/__init__.py` | `get_storage(None\|str\|BaseStorage) → BaseStorage` | `isinstance(RDBStorage)` check — violates LSP for custom subclasses |
| **Proxy (Cache)** | `storages/_cached_storage.py` | `_CachedStorage` transparent DB cache | Thread-local cache — stale read risk between threads |
| **Façade** | `optuna/__init__.py` | 5 names exposed from 200+ files | Masks the God Object complexity of `Study` |

---

### Architecture Tactics

| Quality Attribute | Tactic | Implementation | Tradeoff |
|---|---|---|---|
| Extensibility | Abstract interfaces | `BaseSampler`, `BasePruner`, `BaseStorage` | Runtime enforcement only — `TypeError` at instantiation |
| Extensibility | Two-level ABC hierarchy | `BaseGASampler(BaseSampler)` | Added cognitive overhead for contributors |
| Extensibility | Plugin registration via ABCs | Subclass + pass to `create_study()` — no registry needed | No discovery mechanism — user must know the class exists |
| Testability | Dependency injection | `Study.__init__` receives all 3 dependencies | Constructor grows with each new plugin axis |
| Testability | Separate concerns | `InMemoryStorage` — zero infrastructure | Cannot be disabled even for debug purposes |
| Testability | Dedicated test fixtures | `optuna/testing/` standard suites for all ABCs | None |
| Scalability | Introduce concurrency | `n_jobs` → `ThreadPoolExecutor` | GIL limits CPU-bound parallelism — `constant_liar` needed |
| Scalability | Use a broker | `GrpcStorageProxy` between workers and DB | +2 network hops · still experimental (v4.2.0) |
| Scalability | Multiple inheritance mixin | `RDBStorage(BaseStorage, BaseHeartbeat)` | None — clean orthogonal contracts |
| Reliability | Heartbeat | `storages/_heartbeat.py` detects dead workers | Periodic thread overhead |
| Reliability | Granular persistence | `set_trial_param()` per `suggest_*()` — immediate write | 20 parameters = 20 DB writes before objective runs |
| Reliability | Immutable data | `copy.deepcopy(frozen_trial)` before callbacks | Memory cost per trial |
| Reliability | Retry on failure | `RetryFailedTrialCallback` re-enqueues FAIL trials | Infinite retry loop if objective always fails |
| Performance | Cache | `_CachedStorage` reduces DB round-trips | Cache invalidation risk across threads |
| Performance | Lazy loading | `_LazyImport` for `visualization`, `importance`, `artifacts` | Import errors hidden until access |
| Performance | Minimize kernel size | Core = study/ + trial/ + distributions.py only | God Object risk — Study accumulates responsibilities |
| Modifiability | Anticipate expected changes | Plugin ABCs = explicit extension contracts | None |
| Modifiability | Deprecation policy | 2-version grace period via `_deprecated.py` | Dead code accumulates for 2–3 versions |
| Modifiability | Separate pipeline stages | Each `_run_trial()` phase independently replaceable | Synchronous coupling between stages |
| Usability | Façade | `create_study()` hides all wiring complexity | Masks God Object — complexity hidden, not resolved |
| Interoperability | Shim layer | `integration/` = 5-line redirects to `optuna-integration` | Extra indirection — harder to trace errors |
| Interoperability | Abstract persistence contract | `BaseStorage` — 18 methods — any backend pluggable | Heaviest ABC contract in the codebase |

---

### Known Weaknesses

| Weakness | Location | Impact |
|---|---|---|
| **God Object** | `study/study.py` — 1751 lines, 15+ public methods | Violates SRP — orchestrates, exposes results, AND implements ask/tell |
| **No native async** | `_optimize.py` — ThreadPoolExecutor only | Python GIL limits true CPU parallelism — no `asyncio` support |
| **SQLite not parallel-safe** | `storages/_rdb/` | Lock contention risk when misconfigured |
| **RDBStorage bottleneck** | `storages/_rdb/storage.py` — 1224 lines | Central DB bottleneck at high worker count without gRPC proxy |
| **Trial → Study tight coupling** | `trial/_trial.py` L.55 — `self.study = study` | Trial cannot exist independently — complicates unit testing |
| **Layer violation #1** | `study/study.py` L.31 | `from optuna.storages._heartbeat import is_heartbeat_enabled` — bypasses `BaseStorage` |
| **Layer violation #2** | `visualization/` — 8 imports from `samplers/` | Optional layer depends on Plugin internals — impure layering |
| **LSP violation** | `optuna_integration/pytorch_lightning.py` | `isinstance(storage, _CachedStorage)` — breaks LSP in DDP mode |
| **Synchronous callbacks** | `study/_optimize.py` L.174 | Slow MLflow/W&B server = trial latency — no timeout mechanism |
| **GrpcStorageProxy experimental** | `storages/_grpc/client.py` — `@experimental_class("4.2.0")` | API unstable — not safe for production SLA |
| **Repository contract too heavy** | `storages/_base.py` — 18 abstract methods | Custom storage requires implementing all 18 — no partial contract |
| **Pipeline synchronous only** | `study/_optimize.py` | No streaming between pipeline stages — each stage blocks the next |


## References

- Akiba et al. (2019). *Optuna: A Next-generation HPO Framework.* KDD 2019.
  DOI: 10.1145/3292500.3330701
- Bass, Clements, Kazman (2012). *Software Architecture in Practice, 3rd Ed.*
- Kruchten, P. (1995). *Architectural Blueprints — The 4+1 View Model.* IEEE Software.
- Optuna Documentation. https://optuna.readthedocs.io
- Optuna GitHub. https://github.com/optuna/optuna
- Bergstra et al. (2013). *Making a Science of Model Search.* ICML 2013.
