# View 5 — Historical Evolution

**Type:** Non-standard — Temporal / Evolution View
**Step produced:** Step 6
**Diagram:** `diagram.png` / `diagram.puml`

---

## Question answered

> How has Optuna's architecture evolved from 2018 to 2026?
> Which architectural decisions were added, changed, or removed over time?

---

## Method

- Read all 10 Alembic migration files in `storages/_rdb/alembic/versions/`
- Extracted `@experimental_class("X.Y.Z")` markers across all modules
- Extracted `@deprecated_class("X.Y.Z", "W.V.Z")` markers
- Read `CITATION.cff` — confirmed KDD 2019 authors and DOI
- Read `README.md` — confirmed v4.7 (Jan 2026) and v4.8 (Mar 2026)
- Read `version.py` — current version: `4.9.0.dev`

---

## DB migration timeline (exact dates from Alembic)

| Migration file | Date | Change |
|---|---|---|
| `v0.9.0.a.py` | 2019-03-12 | Initial schema — 7 tables |
| `v1.2.0.a.py` | 2020-02-05 | + system_attrs on studies & trials |
| `v1.3.0.a.py` | 2020-02-14 | Distributions → JSON in DB |
| `v2.4.0.a.py` | 2020-11-17 | Multi-objective: study_directions + trial_values plural |
| `v2.6.0.a_.py` | 2021-03-01 | MAX_STRING_LENGTH = 2048 |
| `v3.0.0.a.py` | 2021-11-21 | Unify distributions → FloatDistribution/IntDistribution |
| `v3.0.0.b.py` | 2022-04-27 | Float precision + intermediate_value nullable |
| `v3.0.0.c.py` | 2022-05-16 | +inf/-inf in intermediate values |
| `v3.0.0.d.py` | 2022-06-02 | +inf/-inf in trial_values |
| `v3.2.0.a_.py` | 2023-02-25 | INDEX on trials.study_id |

---

## The 6 major architectural decisions

| ID | Decision | Version | Evidence |
|---|---|---|---|
| D1 | Define-by-run API | v0.1 (2018) | `trial/_trial.py` — `_suggest()` dynamic dispatch |
| D2 | Stable DB schema | v0.9 (2019) | `v0.9.0.a.py` — 7 tables, all still present |
| D3 | Multi-objective native | v2.4 (2020) | `v2.4.0.a.py` — `study_directions` table |
| D4 | Distribution unification | v3.0 (2022) | `v3.0.0.a.py` + `deprecated_class("3.0.0","6.0.0")` |
| D5 | integration/ extracted | v3.0 (2022) | `integration/mlflow.py` = 5-line shim |
| D6 | GrpcStorageProxy | v4.2 (2024) | `@experimental_class("4.2.0")` in `_grpc/client.py` |

---

## Stability analysis by module

| Module | Stability | Note |
|---|---|---|
| `samplers/_base.py` | ✅ Unchanged since v1.0 | ABC contract frozen |
| `pruners/_base.py` | ✅ Unchanged since v1.0 | ABC contract frozen |
| `study/study.py` public API | ✅ Stable | `optimize()` `ask()` `tell()` unchanged |
| `trial/_trial.py` | ✅ Core stable | `suggest_*` legacy deprecated v3.0, removed v6.0 |
| `storages/_base.py` | ⚠️ Growing | 17 abstract methods — new ones added per version |
| `distributions.py` | 🔄 Refactored | Massively unified in v3.0 |
| `integration/` | 🔄 Extracted | Moved to `optuna-integration` in v3.0 |

---

## Deprecation policy — confirmed in code

```python
# distributions.py
@deprecated_class("3.0.0", "6.0.0")  # deprecated since 3.0, removed in 6.0
class UniformDistribution: ...

# study/study.py
@deprecated_func("3.1.0", "5.0.0")  # 2-version grace period
def trials_dataframe(...): ...
```

Grace period = 2 to 3 major versions.
This gives users time to migrate without breaking changes.

---

## Key insight

> The core (study/, trial/, ABCs) has been stable for 6+ years.
> All architectural evolution concentrated in:
> → storages/ (10 DB migrations)
> → distributions.py (unified in v3.0)
> → integration/ (extracted in v3.0)
>
> This proves the plugin architecture achieved its design goal:
> stable interface, evolvable implementation.
