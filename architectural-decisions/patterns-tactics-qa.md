# Architectural Decisions — Final Tracker

**Project:** Software Architecture Analysis of Optuna  
**Updated through:** Step 10 — Final Presentation  
**Method:** Every entry traced to a file + line in the source code.

---

## 1. Architecture Decisions — 7 Categories (Bass/Clements)

---

### Category 1 — Allocation of Responsibilities

**Decision:** `Study` is the sole orchestrator of the entire optimization loop.

**Rationale:**
Single entry point concentrates control flow in one place, keeping
the user API to two lines: `create_study()` + `study.optimize()`.
Study delegates everything — it never computes, never stores, never samples.

**Evidence:**
```python
# study/study.py
def optimize(self, func, n_trials, ...):
    _optimize_sequential(self, func, ...)  # delegates
```

**QA addressed:** Usability · Modifiability

**Defect:**
Study becomes a God Object — 1751 lines, 15+ public methods.
Violates Single Responsibility Principle.
Should be split into `StudyOrchestrator` + `StudyResults`.

---

### Category 2 — Coordination Model

**Decision:** Synchronous method calls for phases 1–4 of each trial.
Asynchronous callbacks (Observer Pattern) for phase 5.

**Rationale:**
Synchronous phases guarantee predictable, traceable execution order.
Async callbacks allow post-trial extensions without modifying Study.

**Evidence:**
```python
# study/_optimize.py
# Phases 1–4: sync call chain
_run_trial(study, func)
# Phase 5: async observer
for cb in callbacks:
    cb(study, copy.deepcopy(frozen_trial))
```

**QA addressed:** Modifiability · Reliability

**Defect:**
A slow callback blocks the next trial from starting.
No timeout mechanism on callback execution.

---

### Category 3 — Data Model

**Decision:** `FrozenTrial` as the immutable data transfer object
shared across all components and threads.

**Rationale:**
Immutability prevents race conditions in parallel execution.
All components receive the same snapshot — no inconsistent reads.

**Evidence:**
```python
# study/_optimize.py L.176
for cb in callbacks:
    cb(study, copy.deepcopy(frozen_trial))  # immutable copy
```

**QA addressed:** Reliability · Testability

**Defect:**
`deepcopy(frozen_trial)` on every trial adds memory overhead.
For high-frequency optimization, this is measurable.

---

### Category 4 — Management of Resources

**Decision:** `_CachedStorage` wraps all database backends transparently,
reducing DB round-trips without user awareness.

**Rationale:**
RDBStorage incurs network/IO cost per call.
The cache absorbs repeated reads (get_trial, get_all_trials)
within a single trial's lifetime.

**Evidence:**
```python
# storages/__init__.py
def get_storage(storage):
    if isinstance(storage, RDBStorage):
        return _CachedStorage(storage)  # auto-wrapped
```

**QA addressed:** Performance · Scalability

**Defect:**
Cache invalidation across threads is manual — thread_local reset.
Risk of stale reads if invalidation logic is missed.
Evidence: `self._thread_local.cached_all_trials = None`

---

### Category 5 — Mapping Among Architectural Elements

**Decision:** Plugin ABCs (`BaseSampler`, `BasePruner`, `BaseStorage`)
are the sole coupling points between Core and Plugin layers.

**Rationale:**
Core depends only on interfaces, never on implementations.
Adding a new algorithm requires zero changes outside its own module.

**Evidence:**
```python
# study/study.py imports
from optuna.samplers._base import BaseSampler   # interface only
from optuna.pruners._base import BasePruner     # interface only
from optuna.storages._base import BaseStorage   # interface only
# Never: from optuna.samplers._tpe import TPESampler
```

**QA addressed:** Extensibility · Modifiability

**Defect:**
`BaseStorage` contract is too heavy — 17 abstract methods.
Writing a custom storage implementation requires implementing
all 17 — many of which may not be needed for a specific use case.
Violates Interface Segregation Principle at the plugin level.

---

### Category 6 — Choice of Technology

**Decision:** SQLAlchemy + Alembic for relational storage.
gRPC + protobuf for the high-scale proxy.

**Rationale:**
SQLAlchemy gives DB-agnosticism across SQLite, PostgreSQL, MySQL.
Alembic provides schema versioning — migrations are tracked.
gRPC gives high-throughput binary protocol for massive worker counts.

**Evidence:**
```python
# storages/_rdb/storage.py L.106
class RDBStorage(BaseStorage, BaseHeartbeat):
    # SQLAlchemy engine
    self.engine = sqlalchemy.create_engine(url, **engine_kwargs)

# storages/_grpc/api.proto
# protobuf contract between proxy and workers
```

**QA addressed:** Scalability · Portability

**Defects:**
- SQLite is not parallel-safe — lock contention under concurrent writes.
  Documented in official FAQ: avoid SQLite for distributed use.
- gRPC proxy is experimental since v4.2.0 — not production-stable.
- Migrating away from SQLAlchemy would require rewriting
  `storages/_rdb/storage.py` (1224 lines).

---

### Category 7 — Quality Attribute Tradeoffs

**Decision:** Accept Study as a God Object to preserve API simplicity.

**Rationale:**
Splitting Study into StudyOrchestrator + StudyResults + StudyAskTell
would complicate the user-facing API. The tradeoff: internal SRP
violation is hidden behind the simple `create_study()` façade.

**Evidence:**
```python
# study/study.py
# 1751 lines · 15+ public methods
# Responsibilities: orchestration, result access, ask/tell, metadata
```

**QA addressed:** Usability (preserved at cost of Maintainability)

**Defect:**
As Study grows, adding features requires modifying the same 1751-line file.
Every new capability becomes technical debt.
Recommended fix: split into 3 classes with clear boundaries.

---

## 2. Quality Attributes — SMART Format

---

### QA 1 — Extensibility

| Field | Value |
|---|---|
| **Stimulus** | An ML engineer external to the Optuna core team needs to add a custom genetic algorithm sampler |
| **Source** | External developer |
| **Artifact** | `samplers/` plugin layer — `BaseSampler` ABC |
| **Environment** | Development time, normal conditions |
| **Response** | Developer subclasses `BaseSampler`, implements 3 methods, passes instance to `create_study(sampler=...)` |
| **Measure** | **0 lines of core modified · completed in < 1 hour** |

**QA addressed by:** Abstract interfaces tactic · Microkernel pattern · Strategy Pattern × 3

**Defect:** None at the ABC level.
Defect at the Storage level: implementing a custom `BaseStorage` requires 17 methods — too heavy.

---

### QA 2 — Scalability

| Field | Value |
|---|---|
| **Stimulus** | MLOps team launches parallel optimization across 10 machines |
| **Source** | MLOps team — production environment |
| **Artifact** | `storages/` layer — `RDBStorage` + PostgreSQL |
| **Environment** | Production, distributed environment, 1000 trials |
| **Response** | All workers connect to shared PostgreSQL · Storage coordinates trial creation · No worker-to-worker communication |
| **Measure** | **Linear throughput scaling with worker count** |

**QA addressed by:** Shared Data pattern · Broker pattern (GrpcProxy) · Introduce concurrency tactic

**Defect:** RDBStorage is a central bottleneck at very high worker count.
GrpcStorageProxy mitigates but is experimental.
SQLite must not be used in this scenario.

---

### QA 3 — Testability

| Field | Value |
|---|---|
| **Stimulus** | CI/CD pipeline runs the full Optuna test suite |
| **Source** | Automated CI system |
| **Artifact** | `InMemoryStorage` + `optuna/testing/` module |
| **Environment** | Development environment, no infrastructure available |
| **Response** | All tests run in RAM · No database connection required · Full ABC contract verified |
| **Measure** | **Execution time < 1s · zero external dependencies** |

**QA addressed by:** Dependency injection tactic · Separate concerns tactic · Repository pattern

**Defect:** None for InMemoryStorage.
Defect: `FixedTrial` exists for testing but `Trial` still holds a hard reference to `Study` — `self.study = study` — making true unit isolation incomplete.

---

### QA 4 — Modifiability

| Field | Value |
|---|---|
| **Stimulus** | Core maintainer deprecates `UniformDistribution` in favor of `FloatDistribution(log=True)` |
| **Source** | Core maintainer — planned API evolution |
| **Artifact** | `optuna/_deprecated.py` + public API surface |
| **Environment** | Active user base across multiple Optuna versions |
| **Response** | Deprecation warning emitted for 2 versions · migration documented · old API still functional during grace period |
| **Measure** | **0 breaking changes for existing users across 2 version cycle** |

**QA addressed by:** Deprecation policy tactic · Anticipate expected changes tactic

**Defect:** The 2-version grace period is a convention, not enforced by the type system.
A maintainer could skip it accidentally.

---

### QA 5 — Reliability

| Field | Value |
|---|---|
| **Stimulus** | Worker process is killed mid-trial after 7 of 20 `suggest_*` calls |
| **Source** | Infrastructure failure — OOM kill or network partition |
| **Artifact** | `storages/_heartbeat.py` + `trial/_trial.py` granular persistence |
| **Environment** | Runtime, distributed cluster, PostgreSQL backend |
| **Response** | Heartbeat thread stops · next trial creation calls `fail_stale_trials()` · dead trial marked FAIL · parameters from steps 1–7 fully preserved in DB |
| **Measure** | **0 data loss · recovery completed before next trial starts** |

**QA addressed by:** Heartbeat tactic · Granular persistence tactic · Retry on failure tactic

**Defect:** Heartbeat adds a background thread per worker — overhead at very high worker counts.
Grace period between heartbeat stop and FAIL marking is configurable but defaults may be too long for fast-failing workloads.

---

### QA 6 — Interoperability

| Field | Value |
|---|---|
| **Stimulus** | ML engineer using MLflow wants automatic logging of every Optuna trial |
| **Source** | External ML engineer |
| **Artifact** | `integration/mlflow.py` — `MLflowCallback` |
| **Environment** | Runtime, user's existing MLflow-instrumented codebase |
| **Response** | User adds `MLflowCallback` to `study.optimize(callbacks=[MLflowCallback()])` |
| **Measure** | **3 lines of user code added · 0 lines of core modified** |

**QA addressed by:** Shim layer tactic · Publish-Subscribe pattern · Lazy loading tactic

**Defect:** Integration modules use lazy imports — import errors are hidden until first use.
A missing MLflow installation raises an error at callback execution time, not at import time.

---

## 3. Architectural Patterns — System Level

---

### Pattern 1 — Microkernel

**Where:** Stable kernel = `study/` + `trial/` + `distributions.py` + 3 ABCs.
Everything else = plugins (samplers, pruners, storages, visualization).

**QA addressed:** Extensibility · Modifiability

**Defect:**
Performance penalty at each kernel boundary crossing.
3 confirmed layer violations where plugins import kernel internals directly.

---

### Pattern 2 — Layered (5 layers)

**Where:**
Layer 1: PUBLIC API     (init.py)
Layer 2: CORE           (study/, trial/, distributions.py)
Layer 3: PLUGIN         (samplers/, pruners/, storages/)
Layer 4: OPTIONAL       (visualization/, integration/, artifacts/)
Layer 5: INFRASTRUCTURE (testing/, _gp/, _hypervolume/)

**QA addressed:** Modifiability · Extensibility · Testability

**Defect:**
Layer violations exist in the codebase:
- `study/` imports from `_heartbeat` (infra layer)
- `visualization/` imports from `samplers/` (plugin layer)
These break the strict layering contract.

---

### Pattern 3 — Shared Data

**Where:** All workers share a single `BaseStorage` instance (PostgreSQL/RDB)
as the single source of truth for trial state.

**QA addressed:** Scalability · Reliability · Consistency

**Defect:**
RDBStorage is a central bottleneck at very high worker count.
No sharding or partitioning strategy — single DB handles all writes.
GrpcProxy is the workaround, not a structural fix.

---

### Pattern 4 — Broker

**Where:** `GrpcStorageProxy` sits between 1000+ workers and PostgreSQL.
Workers connect to the proxy — proxy connects to DB.

**QA addressed:** Scalability · Performance

**Defect:**
Adds 2 network hops per storage call.
Experimental since v4.2.0 — not production-stable.
Introduces a new potential single point of failure.

---

### Pattern 5 — Client-Server

**Where:** `Study` (client) sends all persistence requests to
`BaseStorage` (server) via unidirectional method calls.

**QA addressed:** Scalability · Separation of Concerns

**Defect:**
Storage server is a potential single point of failure.
No built-in failover strategy in the pattern itself.

---

### Pattern 6 — Publish-Subscribe

**Where:** `study.optimize(callbacks=[...])` — after each trial,
all registered callbacks receive `(study, deepcopy(frozen_trial))`.

**QA addressed:** Interoperability · Modifiability

**Defect:**
Callbacks are synchronous — a slow callback blocks the next trial.
No timeout or circuit-breaker on callback execution.

---

### Pattern 7 — Repository

**Where:** `BaseStorage` — 17 abstract methods forming a complete
persistence contract. All reads and writes go through this interface.

**QA addressed:** Testability · Portability

**Defect:**
17 abstract methods is a heavy contract for custom implementations.
Violates Interface Segregation Principle at the storage level.
A lightweight read-only storage still must implement all write methods.

---

### Pattern 8 — Pipeline

**Where:** `_run_trial()` in `study/_optimize.py` — 5 sequential filters:
suggest → evaluate → prune → persist → callback

**QA addressed:** Modifiability · Extensibility

**Defect:**
Pipeline is synchronous only — no streaming between filters.
Each phase must complete before the next starts.

---

### Pattern 9 — Multi-Tier (gRPC mode)

**Where:** 3-tier architecture in distributed mode:
Tier 1: Workers (Study + Sampler + objective)
Tier 2: GrpcStorageProxy (coordination layer)
Tier 3: PostgreSQL (persistence layer)

**QA addressed:** Scalability · Separation of Concerns

**Defect:**
3 components to deploy and monitor.
Operational complexity increases significantly vs single-node.
Experimental stability of GrpcProxy.

---

## 4. Architecture Tactics

---

### Extensibility Tactics

**Tactic 1 — Use abstract interfaces**
Implementation: `BaseSampler`, `BasePruner`, `BaseStorage` as sole contact points.
File: `samplers/_base.py`, `pruners/_base.py`, `storages/_base.py`
QA: Extensibility
Defect: None at this level. Defect at storage level: 17 abstract methods too heavy.

**Tactic 2 — Two-level ABC hierarchy**
Implementation: `BaseGASampler(BaseSampler)` adds GA-specific contract
without modifying root `BaseSampler`.
File: `samplers/_ga/_base.py`
QA: Extensibility · Modifiability
Defect: None identified.

**Tactic 3 — Plugin registration via ABCs**
Implementation: No central registry needed — Python ABC enforcement
at instantiation guarantees contract compliance.
QA: Extensibility
Defect: Discovery of available plugins requires manual inspection of `__init__.py`.

---

### Testability Tactics

**Tactic 4 — Dependency injection**
Implementation: `Study.__init__` receives `sampler`, `pruner`, `storage`
as parameters — never instantiates them internally.
File: `study/study.py` L.85–94
QA: Testability
Defect: None.

**Tactic 5 — Separate concerns (InMemoryStorage)**
Implementation: `InMemoryStorage` is a full `BaseStorage` implementation
in RAM — enables complete testing without any infrastructure.
File: `storages/_in_memory.py`
QA: Testability
Defect: None.

**Tactic 6 — Dedicated test fixtures**
Implementation: `optuna/testing/` module provides standard suites —
any custom `BaseSampler` can be validated automatically.
File: `testing/samplers.py`, `testing/storages.py`
QA: Testability
Defect: None.

---

### Scalability Tactics

**Tactic 7 — Introduce concurrency**
Implementation: `n_jobs` parameter in `optimize()` maps to
`ThreadPoolExecutor(max_workers=n_jobs)`.
File: `study/_optimize.py`
QA: Scalability
Defect: Python GIL limits true CPU parallelism.
CPU-bound objective functions do not benefit from threading.

**Tactic 8 — Use a broker**
Implementation: `GrpcStorageProxy` absorbs DB connection load
from thousands of workers.
File: `storages/_grpc/`
QA: Scalability
Defect: Experimental. Adds operational complexity. 2 extra network hops.

**Tactic 9 — constant_liar (TPESampler)**
Implementation: In parallel mode, TPESampler fills in-progress trial
values with a constant "liar" value to avoid suggesting duplicates.
File: `samplers/_tpe/sampler.py`
QA: Scalability
Defect: Approximate — liar value introduces bias in the TPE model.

---

### Reliability Tactics

**Tactic 10 — Heartbeat**
Implementation: Background thread pings `Storage.record_heartbeat(trial_id)`
periodically. On next trial creation, `fail_stale_trials()` marks dead trials.
File: `storages/_heartbeat.py`
QA: Reliability
Defect: Thread overhead per worker. Default grace period may be
too long for fast-failing workloads.

**Tactic 11 — Granular persistence**
Implementation: `Storage.set_trial_param()` called immediately after
each `suggest_*()` — not at end of trial.
File: `trial/_trial.py` L.643
QA: Reliability
Defect: None — this is the correct behavior.

**Tactic 12 — Immutable data in callbacks**
Implementation: `copy.deepcopy(frozen_trial)` before passing to callbacks.
File: `study/_optimize.py` L.176
QA: Reliability
Defect: Memory overhead per trial for large trial objects.

**Tactic 13 — Retry on failure**
Implementation: `RetryFailedTrialCallback` re-queues failed trials
as WAITING state automatically.
File: `storages/_callbacks.py`
QA: Reliability
Defect: Risk of infinite retry loop without a max_retry guard.

---

### Performance Tactics

**Tactic 14 — Cache**
Implementation: `_CachedStorage` wraps `RDBStorage` transparently,
caching `get_trial()` and `get_all_trials()` results per trial scope.
File: `storages/_cached_storage.py`
QA: Performance
Defect: Stale read risk if cache invalidation logic is missed
across threads. Thread-local invalidation via `_thread_local`.

**Tactic 15 — Lazy loading**
Implementation: `visualization`, `importance`, `artifacts` loaded
via `_LazyImport` — Plotly and sklearn never imported at startup.
File: `optuna/__init__.py`
QA: Performance · Usability
Defect: Import errors hidden until first access.
A missing Plotly installation only fails when user calls `plot_*()`.

**Tactic 16 — Minimize kernel size**
Implementation: Core = `study/` + `trial/` + `distributions.py` only.
All algorithms, visualizations, integrations are external to kernel.
QA: Performance · Extensibility
Defect: None.

---

### Modifiability Tactics

**Tactic 17 — Anticipate expected changes**
Implementation: The three ABCs are explicit extension contracts.
Plugin points are designed upfront — not added reactively.
File: `samplers/_base.py`, `pruners/_base.py`, `storages/_base.py`
QA: Modifiability
Defect: None.

**Tactic 18 — Deprecation policy**
Implementation: `_deprecated.py` emits warnings for 2 version cycles
before any removal. Documented in changelog.
File: `optuna/_deprecated.py`
QA: Modifiability
Defect: Policy is a convention — not enforced by the type system.

---

### Usability Tactics

**Tactic 19 — Façade**
Implementation: `create_study()` wires `Study` + default `TPESampler`
+ default `MedianPruner` + `InMemoryStorage` in one call.
File: `optuna/__init__.py`
QA: Usability
Defect: None.

---

### Interoperability Tactics

**Tactic 20 — Shim layer**
Implementation: `integration/` module contains 5-line adapter files
for each framework — lazy import, zero core coupling.
File: `integration/mlflow.py`, `integration/lightgbm.py`, etc.
QA: Interoperability
Defect: Import errors deferred to runtime — not caught at startup.

---

## 5. Known Weaknesses — Full Catalogue

| # | Weakness | Location | Evidence | Impact | Step |
|---|---|---|---|---|---|
| 1 | **God Object** | `study/study.py` | 1751 lines, 15+ public methods | Violates SRP — 3 responsibilities in 1 class | S1 |
| 2 | **No native async** | `study/_optimize.py` | `ThreadPoolExecutor` only | GIL limits CPU parallelism for compute-bound trials | S1 |
| 3 | **SQLite not parallel-safe** | `storages/_rdb/` | Official FAQ warning | Lock contention under concurrent writes | S1 |
| 4 | **RDBStorage bottleneck** | `storages/_rdb/storage.py` | 1224 lines, single DB endpoint | Central bottleneck at high worker count | S1 |
| 5 | **Trial → Study tight coupling** | `trial/_trial.py` L.55 | `self.study = study` | Trial cannot exist independently — limits unit testing | S3 |
| 6 | **Layer violations** | `study/` → `_heartbeat`, `visualization/` → `samplers/` | Import graph analysis | Breaks strict layering contract | S6 |
| 7 | **LSP violation** | `storages/__init__.py` | `isinstance(storage, _CachedStorage)` | Concrete type check — violates Liskov Substitution | S6 |
| 8 | **BaseStorage too heavy** | `storages/_base.py` | 17 abstract methods | Custom storage = implement all 17 — violates ISP | S6 |
| 9 | **Slow callback = latency** | `study/_optimize.py` | No timeout on callbacks | Synchronous callbacks block next trial start | S6 |
| 10 | **Stale cache reads** | `storages/_cached_storage.py` | Thread-local invalidation | Race condition risk in high-concurrency scenarios | S6 |

---

## 6. Open Questions — Resolved & Remaining

| # | Question | Status | Resolution |
|---|---|---|---|
| 1 | Observer Pattern in callbacks? | ✅ Confirmed | `deepcopy(frozen_trial)` in `_optimize.py` L.176 |
| 2 | _CachedStorage invalidation? | ✅ Confirmed | `_thread_local.cached_all_trials = None` per trial |
| 3 | study/ refactor trigger? | ✅ Confirmed | v3.0 — single file hit 1700 lines, separated into package |
| 4 | integration/ zero-coupling? | ✅ Confirmed | Lazy import only — nothing from core at load time |
| 5 | NSGA-II before_trial → trial ordering? | ⬜ Open | Not fully resolved — needs commit history analysis |
| 6 | GrpcProxy stability in v4.2? | ⬜ Open | Marked experimental — production stability unknown |
