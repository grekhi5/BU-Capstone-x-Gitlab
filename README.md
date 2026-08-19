# MLOps: a Registry-Driven ML Lifecycle on Snowflake

A working implementation for GitLab's Data Science team: an end-to-end machine learning
workflow anchored on a **centralized model registry** as the single source of truth for
production models.

Everything lives in one notebook, [`mlops_capstone_notebook.ipynb`](mlops_capstone_notebook.ipynb),
which runs top to bottom inside Snowflake and carries its own explanation and its saved
output. Read that for the detail; read on here for orientation and how to run it.

## Background

GitLab's Data Science team builds internal predictive models across marketing, sales,
finance and product: lead and free-account conversion, customer segmentation, marketing mix
modeling, customer-lifecycle analysis. Today each model runs as its own standalone process
in its own Git repository. Version control exists, but it is clunky where it matters most:

- No single view of which model version is in production.
- Hard to trace when a model shipped, or from what.
- Rolling back to a known-good version is slow.
- Nothing watches whether a model is still performing.
- Feature logic is written twice, once for training and once for scoring.

## What this does about it

The Model Registry gives a model an identity, a **name and a version**, governed by
Snowflake's own permissions and callable directly from SQL. Once a model has that, serving
it, switching it and monitoring it stop being deployments and become operations.

The notebook demonstrates the whole lifecycle, and does it for models arriving by two
different routes:

```
   trained here          ─┐
                          ├──►  MODEL REGISTRY  ──►  score
   pre-trained in a repo ─┘      name + version      automate
                                                     switch / roll back
                                                     monitor
```

- **`MQL_PROPENSITY`** is trained in the notebook on live GitLab data, in two versions so
  there is something to switch between and roll back to.
- **`PROPENSITY_TO_EXPAND`** is a real production model that already existed as a file in
  its own repository, lifted into the registry without retraining.

After registration nothing downstream cares which route a model took. That is the point.

## Running it

This is a **Snowflake notebook**. It is not run locally: it calls `get_active_session()` and
expects to execute inside a Snowflake workspace.

1. Open a Snowflake workspace and add `mlops_capstone_notebook.ipynb` to it.
2. Make sure `snowflake-ml-python`, `scikit-learn` and `xgboost` are available to the notebook.
3. Edit the parameters cell in Section 3. That is the only cell you should need to touch.
4. Run top to bottom. Section 15 is a guarded teardown if you want to start clean.

### Access it needs

| Need | Why |
|---|---|
| `READ` on `PROD.COMMON_MART_MARKETING.MART_CRM_PERSON` | build features and labels |
| Write on a sandbox schema (here the `MLOPS_SANDBOX` database) | feature views, models, predictions, tasks |
| `CREATE MODEL` on the registry schema | log model versions |
| `CREATE DYNAMIC TABLE` on the feature schema | feature views materialize as dynamic tables |
| `USAGE` on a warehouse (here `DEV_XS`) | training, inference, scheduled scoring |
| `EXECUTE TASK ON ACCOUNT` | *only* to activate the schedule; everything else works without it |

It runs as the `MLOPS_SANDBOX` role, which both owns the sandbox and can read production.
It reads from `PROD` and never writes to it.

## What the notebook covers

| Section | |
|---|---|
| 1 to 3 | The problem, what the registry changes, setup |
| 4 | Feature Store: one feature definition used by both training and scoring |
| 5 | Registering a model trained here, two versions with metrics |
| 6 | Registering a model that already exists, lifted from a repository |
| 7 | Both models side by side in the registry, called the same way |
| 8 to 9 | Scoring in-warehouse, then automating it with a stored procedure and a task |
| 10 | Switching the serving version and rolling it back |
| 11 | Monitoring, with a portable drift measure as a fallback |
| 12 to 14 | Constraints we hit, what it gives you, how to reuse it for another model |

## Known limitations

**The imported model cannot be scored on its real data from this role.** Its query reads
`PROD.RESTRICTED_SAFE_COMMON_MART_SALES`, which the sandbox role has no access to, correctly
so. Registration does not need that data, because the model's signature is fully described by
its config, so the notebook registers it and calls it without touching a restricted row. A
real scoring run needs the grant, or someone who already has it.

**Its own feature preparation stays in its own repository.** We deliberately do not copy it
here. This repository provides the bridge into the registry, not a second copy of another
team's preprocessing.

**Re-running starts clean.** Registration replaces the whole model rather than one version,
because Snowflake will not delete whichever version is currently the default, and the scoring
tables are rebuilt for the same reason. So version history accumulates within a run, not
across runs. Worth knowing before treating this registry as a permanent audit trail.

**Two findings worth carrying forward**, both discussed in the notebook's Section 12:

- The imported artifact was pickled by an older XGBoost than the one installed, so
  `get_params()` failed during registration. Nothing in the source repository recorded which
  library version produced the file.
- Several of that model's feature names contain dots and minus signs, which are not valid
  bare SQL identifiers, and those names are stored inside the booster so they cannot be
  renamed. It has to be registered case-sensitive, which also quotes its method names.

## Moving the source to a real GitLab repository

The imported model currently arrives from a Snowflake Workspace acting as the repository,
because the GitLab to Snowflake connector needs admin permissions that were out of scope. A
Workspace is structurally the same thing, a versioned folder of files.

When the connector is available, one line in Section 3 changes:

```python
MODEL_SOURCE = os.path.join(os.environ["CI_PROJECT_DIR"], "scoring")   # inside GitLab CI
```

`MODEL_SOURCE` accepts either a `snow://` URI or a plain filesystem path, and nothing else in
the notebook knows the difference.

## Roadmap

- [x] Registry on Snowflake: register, version, retrieve, score, roll back
- [x] Feature Store with one definition shared by training and scoring
- [x] Automated in-warehouse scoring via a stored procedure and a Snowflake Task
- [x] Drift and performance monitoring via Snowflake ML Observability
- [x] A real production model lifted from its repository into the registry
- [ ] Migrate the model source from Snowflake Workspaces to GitLab repositories
- [ ] Register from CI, so merging a model registers a version automatically
- [ ] Turn monitoring into alerting; the monitors exist, thresholds and routing do not
- [ ] Pin dependency versions when logging, given the version skew found above
- [ ] Document the pattern in the GitLab handbook so other teams can adopt it

## Team

- BU MSBA Capstone interns: **Mohamad Gong** (project manager), **Gurveen Rekhi**, **Abbinaya Kalidhas**
- **Sponsor:** Kevin Dietz, Staff Data Scientist, GitLab Data Science

Models are for internal use only; methods and learnings are shared with the community via the
GitLab handbook.
