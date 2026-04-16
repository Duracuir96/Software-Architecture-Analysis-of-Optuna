# View 2 — Component & Connector

**Type:** C&C View (Kruchten Process View)  
**Step produced:** Step 3  
**Diagram:**
<img width="500" height="400" alt="image" src="https://github.com/user-attachments/assets/b71fe3b5-229c-4f1b-9704-248248b22d6d" />


---

## Question answered

> At runtime, which components are active, how do they communicate,
> and in what order are calls made during a single trial?

---

## Method

- Read `study/_optimize.py` — `_run_trial()`, `_optimize_sequential()`, `_optimize_parallel()`
- Read `study/_tell.py` — `_tell_with_warning()`
- Read `trial/_trial.py` — `_suggest()`, `report()`, `should_prune()`
- Read `storages/_heartbeat.py` — `HeartbeatThread`, `record_heartbeat()`
- Read `storages/_callbacks.py` — `RetryFailedTrialCallback`
- Read `_callbacks.py` — `MaxTrialsCallback`
- Extracted all 18 connectors with exact method signatures

---

## Components

| Component | File | Role |
|---|---|---|
| `Study` | `study/study.py` | Orchestrator — delegates to all three ABCs |
| `Trial` | `trial/_trial.py` | Proxy — user interface to Sampler + Pruner + Storage |
| `BaseSampler` | `samplers/_base.py` | Algorithm — decides which params to try |
| `BasePruner` | `pruners/_base.py` | Guard — decides when to stop a trial |
| `BaseStorage` | `storages/_base.py` | Memory — persists all state |
| `ObjectiveFunc` | user code | Business logic — trains the model |
| `HeartbeatThread` | `storages/_heartbeat.py` | Background thread — pings storage periodically |
| `Callbacks` | `_callbacks.py` | Observers — react post-trial |

---

## Trial lifecycle — 5 phases

### Phase 1 — Creation

```
Storage.create_new_trial(study_id)   → trial_id
Storage.get_trial(trial_id)          → FrozenTrial (initial cache)
Trial.init(study, trial_id)          → Trial object
```

### Phase 2 — Pre-trial

```
Sampler.before_trial(study, frozen_trial)   ← no-op by default
← NSGA-II uses it for population init
```

### Phase 3 — Execution

```
For each suggest_*(name, ...):
    [lazy, 1×] Sampler.infer_relative_search_space(study, trial)
    [lazy, 1×] Sampler.sample_relative(study, trial, search_space)
    [or]        Sampler.sample_independent(study, trial, name, dist)
    Storage.set_trial_param(trial_id, name, value, dist)   ← immediate

For each report(value, step):
    Storage.set_trial_intermediate_value(trial_id, step, value)   ← immediate

For each should_prune():
    Pruner.prune(study, frozen_trial)          → bool
    └─ Pruner reads Storage.get_all_trials() to compare history
```

### Phase 4 — Post-trial

```
Sampler.after_trial(study, frozen_trial, state, values)   ← before state stored
Storage.set_trial_state_values(trial_id, state, values)   ← COMPLETE/PRUNED/FAIL
```

### Phase 5 — Callbacks (Observer Pattern)

```
frozen_trial = Storage.get_trial(trial_id)              ← re-read final state
for cb in callbacks:
    cb(study, copy.deepcopy(frozen_trial))              ← immutable deepcopy
```

---

## 18 connectors catalogue

| ID | Method | From → To | Phase |
|---|---|---|---|
| C1 | `create_new_trial()` | Study → Storage | 1 |
| C2 | `get_trial()` | Study/Trial → Storage | 1, 5 |
| C3 | `get_all_trials()` | Pruner → Storage | 3C |
| C4 | `set_trial_state_values()` | Study → Storage | 4 |
| C5 | `set_trial_param()` | Trial → Storage | 3A |
| C6 | `set_trial_intermediate_value()` | Trial → Storage | 3B |
| C7 | `infer_relative_search_space()` | Trial → Sampler | 3A |
| C8 | `sample_relative()` | Trial → Sampler | 3A |
| C9 | `sample_independent()` | Trial → Sampler | 3A |
| C10 | `prune()` | Trial → Pruner | 3C |
| C11 | `before_trial()` | Study → Sampler | 2 |
| C12 | `after_trial()` | Study → Sampler | 4 |
| C13 | `callback(study, frozen_trial)` | Study → Callbacks | 5 |
| C14 | `study.stop()` | MaxTrialsCallback → Study | 5 |
| C15 | `study.add_trial(WAITING)` | RetryFailedTrialCallback → Study | 5 |
| C16 | `study.stop()` | TerminatorCallback → Study | 5 |
| C17 | `record_heartbeat()` | HeartbeatThread → Storage | async |
| C18 | `fail_stale_trials()` | Study → Storage | 1 |

---

## Parallel mode

```python
# study/_optimize.py
with ThreadPoolExecutor(max_workers=n_jobs) as executor:
    # Each thread runs its own _optimize_sequential loop
    # Storage is the ONLY shared state — thread-safe by contract
```

`TPESampler` uses `constant_liar` parameter to avoid duplicate
suggestions across concurrent workers.

---

## Key observations

### 1. Storage is the single point of truth

All components read and write exclusively through `BaseStorage`.
No direct component-to-component state sharing exists.
This is what makes parallel workers safe.

### 2. Granular persistence protects against crashes

`set_trial_param()` is called after every single `suggest_*()` — not
at the end of the trial. A worker killed mid-trial loses nothing.

### 3. Callbacks receive a deepcopy — immutability enforced

`copy.deepcopy(frozen_trial)` in `_optimize_sequential()` L.176.
Callbacks cannot corrupt the trial state. This is a deliberate
reliability decision confirmed in the source code.

### 4. Sampler wraps the trial on both ends

`before_trial()` at init + `after_trial()` before state commit.
TPE updates its density model in `after_trial()`.
NSGA-II manages its population in `before_trial()`.

---

## Architectural insights for the report

| Tactic | Implementation |
|---|---|
| **Reliability** | Granular Storage writes + heartbeat = no data loss |
| **Observer Pattern** | Callbacks are the extension point for post-trial logic |
| **Strategy Pattern** | All three plugin ABCs are called via interface only |
| **Scalability** | Storage abstraction is the only change needed to go from a single laptop to a 100-machine cluster |


