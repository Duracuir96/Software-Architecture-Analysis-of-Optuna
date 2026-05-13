# View 4 — Deployment

**Type:** Allocation View (Kruchten Physical View)
**Step produced:** Step 6
**Diagram:** `diagram.png` / `diagram.puml`

---

## Question answered

> How are Optuna's software components mapped onto physical
> infrastructure? What processes run on which machines?

---

## Method

- Read `storages/__init__.py` — `get_storage()` factory
- Read `storages/_grpc/client.py` — GrpcStorageProxy source
- Read `storages/journal/__init__.py` — JournalStorage backends
- Read `storages/_rdb/storage.py` — SQL backends
- Read `storages/_heartbeat.py` — HeartbeatThread
- Read `integration/pytorch_lightning.py` — DDP constraint
- Read `optuna_integration.mlflow.MLflowCallback` source
- Read `optuna_integration.sklearn.OptunaSearchCV` source

---

## The 5 deployment modes

| Mode | Storage | Machines | Persistent | Heartbeat |
|---|---|---|---|---|
| 1 Local mono-process | `InMemoryStorage` | 1 | ❌ | ❌ |
| 2 Local multi-thread | `InMemoryStorage` | 1 | ❌ | ❌ |
| 3 Multi-process | `JournalStorage` | 1 (NFS±) | ✅ | ❌ |
| 4 Distributed | `RDBStorage` | N | ✅ | ✅ |
| 5 Very large scale | `GrpcStorageProxy` | N + proxy | ✅ | ✅ |

---
<img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/705bb2e5-9eac-47cf-ad40-7788e56c2723" />
<img width="700" height="508" alt="image" src="https://github.com/user-attachments/assets/f9780932-8481-455c-8a3c-633b8bb79d4f" />


## The architectural key

```python
# Only 1 parameter changes across all 5 modes:
study = optuna.create_study(storage=???)

# Mode 1+2 : storage=None
# Mode 3   : storage=JournalStorage(JournalFileBackend("/tmp/..."))
# Mode 4   : storage="postgresql://user:pass@host/db"
# Mode 5   : storage=GrpcStorageProxy(host="proxy", port=13000)

# Objective function: NEVER changes.
```

---

## Integration layer

Three types of integration patterns identified:

**Observer (post-trial callbacks):**
`MLflowCallback`, `WeightsAndBiasesCallback`, `TensorBoardCallback`
→ Implement `Callable[[Study, FrozenTrial], None]`
→ Zero core modification

**Pruning hooks (mid-training):**
`PyTorchLightningPruningCallback`, `LightGBMPruningCallback`,
`XGBoostPruningCallback`
→ Call `trial.report()` and `trial.should_prune()` from framework hooks

**Adapter (drop-in replacement):**
`OptunaSearchCV`
→ Exposes sklearn interface (`fit()`, `best_params_`, `cv_results_`)
→ Uses Optuna internally

---

## Coupling violation found

```python
# pytorch_lightning.py — DDP mode only
if not (
    isinstance(self._trial.study._storage, _CachedStorage)
    and isinstance(self._trial.study._storage._backend, RDBStorage)
):
    raise ValueError("Only RDBStorage supported in DDP.")
```

Only integration that breaks the BaseStorage abstraction.
Documented as a pragmatic constraint for DDP correctness.

---

## Key observations

**1. `get_storage()` is a Factory + Adapter.**
Normalizes `None`, `str`, `BaseStorage` into one output type.
Silently wraps `RDBStorage` in `_CachedStorage` — user never sees it.

**2. Heartbeat is opt-in via multiple inheritance.**
`RDBStorage(BaseStorage, BaseHeartbeat)` — only storages that
support heartbeat implement `BaseHeartbeat`. `InMemoryStorage` does not.

**3. `GrpcStorageProxy` is experimental (`@experimental_class("4.2.0")`).**
Introduced in 2024. Still being validated by the community.
The API may change before becoming stable.

**4. `integration/` is a shim layer since v3.0.**
Every file in `optuna/integration/` is 5 lines — a redirect to
`optuna-integration` package. Core Optuna has zero ML framework deps.

---

## Architectural insights for the report

- **Scalability tactic:** `BaseStorage` abstraction = the single mechanism
  that enables all 5 deployment modes without user code changes.
- **Reliability tactic:** heartbeat + granular `set_trial_param()` = no
  data loss even in distributed failure scenarios.
- **Interoperability tactic:** `integration/` shim layer = zero coupling
  between core Optuna and any ML framework.
