# View 3 — Class & Inheritance

**Type:** Module View / Logical View (Kruchten Logical View)  
**Step produced:** Step 3  
**Diagram:** `diagram.png` / `diagram.puml`

---

## Question answered

> How is the type hierarchy structured?
> What are the abstract contracts, and how do concrete classes implement them?

---

## Method

- Read all 4 ABCs: `samplers/_base.py`, `pruners/_base.py`,
  `storages/_base.py`, `trial/_base.py`
- Counted `@abc.abstractmethod` decorators per class
- Verified all `class X(Y):` declarations across 200+ files
- Ran live Python tests to verify ABC enforcement
- Read `storages/_heartbeat.py` for multiple inheritance pattern
- Read `samplers/_ga/_base.py` for intermediate ABC pattern

---

## ABC summary

| ABC | Abstract methods | Optional hooks | Implementations |
|---|---|---|---|
| `BaseSampler` | 3 | 3 (no-op) | 10 |
| `BaseGASampler` | 1 (`select_parent`) | — | 2 |
| `BasePruner` | 1 | 0 | 8 |
| `BaseStorage` | 17 | 0 | 5 |
| `BaseTrial` | 21 | 0 | 3 |
| `BaseHeartbeat` | 4 (mixin) | — | 2 |

---

## Hierarchy 1 — BaseSampler

```
BaseSampler (ABC)
Abstract:  infer_relative_search_space()
           sample_relative()
           sample_independent()
Optional:  before_trial()  after_trial()  reseed_rng()
│
├── TPESampler          ← default · Bayesian · _tpe/sampler.py
├── GPSampler           ← Gaussian Process · _gp/sampler.py
├── CmaEsSampler        ← CMA-ES evolutionary · _cmaes.py
├── RandomSampler       ← infer():{} · sample_relative():{} · _random.py
├── GridSampler         ← _grid.py
├── QMCSampler          ← Quasi-Monte Carlo · _qmc.py
├── BruteForceSampler   ← _brute_force.py
├── PartialFixedSampler ← _partial_fixed.py
└── BaseGASampler (sub-ABC)
    Abstract:  select_parent()
    Concrete:  get_trial_generation()  get_population()
    │
    ├── NSGAIISampler   ← NSGA-II multi-objective · nsgaii/_sampler.py
    └── NSGAIIISampler  ← NSGA-III many-objective · _nsgaiii/_sampler.py
```

`RandomSampler` is the simplest implementation:
`infer_relative_search_space()` returns `{}` and `sample_relative()`
returns `{}` — it uses independent sampling exclusively.

---

## Hierarchy 2 — BasePruner

```
BasePruner (ABC)
Abstract:  prune(study, trial) → bool
│
├── MedianPruner           ← default · compares to median
├── HyperbandPruner        ← dynamic resource allocation
├── SuccessiveHalvingPruner ← async halving
├── PercentilePruner       ← configurable percentile
├── WilcoxonPruner         ← statistical test
├── PatientPruner ──wraps──► BasePruner   ← Decorator Pattern
├── ThresholdPruner        ← absolute threshold
└── NopPruner              ← always False · Null Object Pattern
```

`BasePruner` has exactly **1 abstract method** — the minimum possible
contract. This is the Interface Segregation Principle applied directly.

---

## Hierarchy 3 — BaseStorage

```
BaseStorage (ABC)
Abstract (17 methods):
Study management: create_new_study · delete_study · get_study_*
Trial management: create_new_trial · set_trial_param
set_trial_state_values · set_trial_intermediate_value
get_trial · get_all_trials · ...
│
├── InMemoryStorage                            ← RAM · dev/tests
├── RDBStorage(BaseStorage, BaseHeartbeat)     ← SQLAlchemy · production
├── JournalStorage                             ← file/Redis · multi-process
├── GrpcStorageProxy                           ← gRPC client · high scale
└── _CachedStorage(BaseStorage, BaseHeartbeat) ← transparent wrapper

BaseHeartbeat (mixin ABC)
Abstract: record_heartbeat()  get_heartbeat_interval()
get_failed_trial_callback()  _get_stale_trial_ids()
│
├── RDBStorage      ← multiple inheritance
└── _CachedStorage  ← multiple inheritance
```

---

## Hierarchy 4 — BaseTrial

```
BaseTrial (ABC)
Abstract (21 methods):
suggest_float · suggest_int · suggest_categorical
report · should_prune · set_user_attr
params · distributions · user_attrs · datetime_start
│
├── Trial       ← active · holds Study ref · delegates to Sampler+Pruner
├── FrozenTrial ← immutable snapshot · post-analysis
└── FixedTrial  ← fixed values · unit testing · no Study dependency
```

---

## ABC enforcement — verified by live Python tests

```python
>>> BaseSampler()
TypeError: Can't instantiate abstract class BaseSampler without
           an implementation for abstract methods
           'infer_relative_search_space', 'sample_independent', 'sample_relative'

>>> BasePruner()
TypeError: Can't instantiate abstract class BasePruner without
           an implementation for abstract method 'prune'

# Partial implementation:
>>> class PartialSampler(BaseSampler):
...     def infer_relative_search_space(...): return {}
...     def sample_relative(...): return {}
>>> PartialSampler()
TypeError: Can't instantiate abstract class PartialSampler without
           an implementation for abstract method 'sample_independent'
```

Python names the missing method precisely — the error message
is itself a guide for developers extending the framework.

---

## Special patterns found in the hierarchy

### Decorator Pattern — `PatientPruner`

```python
class PatientPruner(BasePruner):
    def __init__(self, wrapped_pruner: BasePruner | None, patience: int):
        self._wrapped_pruner = wrapped_pruner  # ← contains a BasePruner
    def prune(self, study, trial) -> bool:
        return self._wrapped_pruner.prune(study, trial)  # ← delegates
```

Implements AND contains `BasePruner` — adds patience without
modifying any existing pruner.

### Multiple Inheritance Mixin — `RDBStorage`

```python
class RDBStorage(BaseStorage, BaseHeartbeat):
    # BaseStorage: persistence contract (17 methods)
    # BaseHeartbeat: reliability contract (4 methods)
    # Two orthogonal concerns — cleanly separated ABCs
```

### `TYPE_CHECKING` pattern — avoids circular imports

```python
# samplers/_base.py
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from optuna.study import Study      # ← only for type checkers
    from optuna.trial import FrozenTrial # ← never imported at runtime
```

ABCs reference `Study` and `FrozenTrial` in signatures without
creating runtime circular imports.

---

## Key observations

### 1. BasePruner's minimal contract is intentional

One abstract method = maximum ease of extension. Writing a custom
pruner requires implementing exactly 1 method. This is a deliberate
application of the Interface Segregation Principle.

### 2. BaseGASampler proves hierarchies can grow without modifying roots

NSGA-II and NSGA-III share generation/population logic via an
intermediate ABC. The `BaseSampler` contract is unchanged.

### 3. Multiple inheritance serves orthogonal concerns

`RDBStorage(BaseStorage, BaseHeartbeat)` combines two completely
independent contracts. This is a clean use of Python multiple
inheritance — not an anti-pattern.

### 4. `FixedTrial` has no `Study` dependency

It extends `BaseTrial` without holding a reference to `Study`.
This makes it usable in unit tests without any infrastructure.
Testability is a structural property of the hierarchy.

---

## Architectural insights for the report

| Pattern / Principle | Application |
|---------------------|-------------|
| **Strategy Pattern** | All 3 plugin ABCs implement the Strategy Pattern — interchangeable algorithms behind a stable interface |
| **Template Method** | `BaseSampler`'s 3-method sequence — calling order fixed, implementations vary |
| **Interface Segregation Principle (ISP)** | `BasePruner` = 1 method. Minimal contract = maximum extensibility with minimum friction |
| **Testability by design** | `FixedTrial` exists specifically so objective functions can be tested without a running `Study` |
| **Decorator Pattern** | `PatientPruner` wraps any `BasePruner` to add patience logic |
| **Null Object Pattern** | `NopPruner` implements `prune()` as `return False` — no special-case handling needed |

