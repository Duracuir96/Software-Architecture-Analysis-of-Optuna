# View 6 — Dependency & Coupling

**Type:** Module View — Uses Style
**Step produced:** Step 6
**Diagram:** `diagram.png` / `diagram.puml`

---

## Question answered

> Who depends on what, and how tightly?
> Where is coupling strong, where is it loose?

---

## Method

- Extracted all `from optuna.X import` statements across all modules
- Counted occurrences per module pair — `uniq -c | sort -rn`
- Identified `isinstance()` checks on concrete classes
- Read `pyproject.toml` — confirmed core external dependencies
- Verified `integration/` coupling (or lack thereof)

---

## Import count matrix (real data)

| Module | Top imports | Count |
|---|---|---|
| `study/` | → trial | 7 |
| `study/` | → storages | 3 |
| `study/` | → distributions | 2 |
| `trial/` | → distributions | 12 |
| `samplers/` | → trial | 16 |
| `samplers/` | → distributions | 15 |
| `samplers/` | → study | 10 |
| `samplers/` | → search_space | 6 |
| `pruners/` | → study | 4 |
| `pruners/` | → trial | 3 |
| `storages/` | → distributions | 20 |
| `storages/` | → trial | 18 |
| `storages/` | → study | 15 |
| `storages/` | → exceptions | 8 |
| `visualization/` | → trial | 17 |
| `visualization/` | → study | 12 |
| `visualization/` | → samplers | 8 |
| `integration/` | → _imports only | 24 |

---

## Coupling classification

### Loose coupling ✅ (by design)
study → BaseSampler (ABC only — never concrete)
study → BasePruner (ABC only — never concrete)
study → BaseStorage (ABC only — never concrete)
integration → core (zero direct import — shim layer only)

### Medium coupling (acceptable)
study ↔ trial (7 imports — same core layer)
trial → distributions (12 imports — same core layer)
storages → distributions (20 imports — persistence layer)

### Strong coupling ⚠️ (risk areas)
storages/_rdb → SQLAlchemy (deep ORM integration)
storages/_rdb → Alembic (schema migration dependency)
visualization → samplers (8 imports — reads internal state)

---

## Coupling violations identified

### Violation 1 — study imports private function

```python
# study/study.py L.31
from optuna.storages._heartbeat import is_heartbeat_enabled
```

Bypasses `BaseStorage` interface — direct import of private module.
Risk: if `_heartbeat.py` is refactored, `study.py` breaks.

### Violation 2 — PyTorchLightning checks concrete storage type

```python
# optuna_integration/pytorch_lightning.py
isinstance(self._trial.study._storage, _CachedStorage)
isinstance(self._trial.study._storage._backend, RDBStorage)
```

Only integration that breaks Liskov Substitution Principle.
Documented as necessary for DDP correctness.
Impact: adding a new storage type requires updating this check.

### Violation 3 — HyperbandPruner self-check

```python
# pruners/__init__.py L.33
isinstance(study.pruner, HyperbandPruner)
```

Concrete class check inside the pruners package itself.
Acceptable (same package), but breaks the ABC abstraction.

---

## External dependencies (core only)

```toml
# pyproject.toml — mandatory dependencies
dependencies = [
  "alembic>=1.5.0",    ← DB migrations (storages/_rdb only)
  "sqlalchemy>=1.4.2", ← ORM (storages/_rdb only)
  "numpy",             ← math (samplers/ + _gp/)
  "packaging>=20.0",   ← version comparison
  "colorlog",          ← logging
  "tqdm",              ← progress bar
  "PyYAML",            ← CLI only
]

# Optional (not installed by default):
# plotly, matplotlib → visualization/
# scipy, scikit-learn → importance/
# grpcio, protobuf  → storages/_grpc/
# redis             → storages/journal/
```

---

## Key observations

**1. `storages/` is the most coupled module.**
20 imports from distributions, 18 from trial, 15 from study.
It must know all data structures to persist them correctly.
This is acceptable — a persistence layer inherently knows its domain.

**2. `integration/` has zero coupling to core.**
24 imports — all to `_imports` (lazy import utility).
No import from `study/`, `trial/`, `samplers/`, `pruners/`, `storages/`.
The shim architecture (v3.0) achieved perfect decoupling.

**3. `visualization/` imports `samplers/` — unexpected coupling.**
8 imports from samplers — reads internal sampler state for
`plot_param_importances()` and search space visualization.
This creates a dependency on plugin internals from the optional layer.

**4. 3 coupling violations — all documented in the codebase.**
None are hidden or accidental. Each has a comment explaining the
pragmatic reason for the exception to the coupling rule.

---

## Architectural insights for the report

- **Dependency inversion confirmed:** Core depends on abstractions
  (ABCs), not concretions. Plugin layer depends on core abstractions.
- **Main coupling risk:** `storages/_rdb` is deeply coupled to
  SQLAlchemy. Replacing SQLAlchemy would require rewriting 1,224 lines.
- **Unexpected coupling:** `visualization/ → samplers/` suggests the
  boundary between optional and plugin layers is not perfectly clean.
- **Best decoupling:** `integration/` — 24 imports, all to utilities.
  Zero knowledge of core Optuna internals.
