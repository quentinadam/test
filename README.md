# GitHub Actions – Condition Mechanics Test Suite

Two `workflow_dispatch` workflows that demonstrate exactly how GitHub's job-level
`if` conditions interact with dependency chains.  Run them yourself and compare
the actual coloured boxes in the Actions UI against the expected outcomes
documented below.

---

## The Four Status Functions

| Function | Returns `true` when… |
|---|---|
| `success()` | **All** direct `needs` jobs succeeded. This is the **implicit default** when no `if` is written. |
| `failure()` | **At least one** direct `needs` job has status `"failure"`. A *skipped* job does **not** count. |
| `always()` | Unconditionally, regardless of any dependency status. |
| `cancelled()` | The workflow run was cancelled. |

> **Key rule:** status functions only look at the **direct** `needs` list, not
> the full upstream chain.  A job two hops away cannot directly trigger
> `failure()` unless it is listed explicitly in `needs`.

---

## Scenario 1 – Basic Condition Mechanics

File: `.github/workflows/scenario-1-basic-conditions.yml`

```
root (FAILS)
  ├── on_success  [no if / default success()]  → SKIPPED
  ├── on_failure  [if: failure()]              → RUNS ✓
  └── on_always   [if: always()]              → RUNS ✓

on_success (SKIPPED)
  ├── after_skipped_default    [no if]          → SKIPPED
  ├── after_skipped_on_failure [if: failure()]  → SKIPPED  ⚠️  see note
  └── after_skipped_on_always  [if: always()]   → RUNS ✓
```

### Expected job statuses

| Job | Expected status | Why |
|---|---|---|
| `root` | failure | Intentionally exits 1 |
| `on_success` | skipped | `success()` is false (root failed) |
| `on_failure` | success | `failure()` is true (root failed) |
| `on_always` | success | `always()` is always true |
| `after_skipped_default` | skipped | `success()` is false (on_success was skipped, not succeeded) |
| `after_skipped_on_failure` | **skipped** ⚠️ | `failure()` is **false** because on_success has status *skipped*, not *failure* |
| `after_skipped_on_always` | success | `always()` is always true |

### The ⚠️ surprise

`failure()` only fires when a direct dependency's status is `"failure"`.
A *skipped* job has status `"skipped"`, which is distinct.  So a job that
uses `if: failure()` downstream of a skipped job will itself be skipped —
not run.  To run unconditionally after any non-success, you need `always()`.

---

## Scenario 2 – Chain Propagation

File: `.github/workflows/scenario-2-chain-propagation.yml`

```
step_a (SUCCEEDS)
  └── step_b (FAILS)
        ├── step_c  [no if]           → SKIPPED
        │     ├── step_e  [no if]     → SKIPPED
        │     ├── step_f  [failure()] → SKIPPED  ⚠️
        │     └── step_g  [always()]  → RUNS ✓
        │
        └── step_d  [failure()]       → RUNS ✓
              └── step_h  [no if]     → RUNS ✓

step_a (SUCCEEDS) ──┐
                    ├── step_i  [no if]     → SKIPPED  ⚠️
step_b (FAILS)   ───┘

step_d (SUCCEEDS) ──┐
                    ├── step_j  [no if]     → SKIPPED  ⚠️
step_b (FAILS)   ───┘

step_d (SUCCEEDS) ──┐
                    └── step_k  [failure()] → RUNS ✓
step_b (FAILS)   ───┘
```

### Expected job statuses

| Job | Expected status | Why |
|---|---|---|
| `step_a` | success | Succeeds normally |
| `step_b` | failure | Intentionally exits 1 |
| `step_c` | skipped | `success()` false (b failed) |
| `step_d` | success | `failure()` true (b failed), job body succeeds |
| `step_e` | skipped | `success()` false (c was skipped) |
| `step_f` | **skipped** ⚠️ | `failure()` false — c was *skipped*, not *failed* |
| `step_g` | success | `always()` true |
| `step_h` | success | `success()` true (d succeeded) — shows failure-recovery can feed normal work |
| `step_i` | **skipped** ⚠️ | `success()` requires ALL direct needs (a AND b) to succeed; b failed |
| `step_j` | **skipped** ⚠️ | Same — b is still a direct need and it failed |
| `step_k` | success | `failure()` true because b (direct need) failed |

### Key rules illustrated

1. **Skip propagates like failure** for the `success()` default.  Once a job
   is skipped, every downstream job that uses `success()` (explicit or
   implicit) is also skipped.

2. **`failure()` does NOT see skips.**  Only an actual `"failure"` status on
   a direct `needs` entry triggers it.  Two hops of skip do not accumulate
   into a failure.

3. **`always()` is the only way to unconditionally break out of a skip chain.**

4. **A failure-recovery job (`failure()`) that succeeds** restores the chain:
   its dependents with the default `success()` condition run normally
   (see `step_d → step_h`).

5. **Multi-dependency `success()`** requires every listed job to have
   succeeded.  One failed entry is enough to skip the dependent job, even if
   other entries succeeded.

6. **Multi-dependency `failure()`** returns `true` if ANY listed job failed,
   regardless of the others (see `step_k`).

---

## Quick Reference

```
Upstream status │ success() │ failure() │ always()
────────────────┼───────────┼───────────┼─────────
success         │   true    │   false   │  true
failure         │   false   │   true    │  true
skipped         │   false   │   false   │  true   ← the gotcha
cancelled       │   false   │   false   │  true
```
