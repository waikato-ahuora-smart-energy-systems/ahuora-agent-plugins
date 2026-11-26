---
name: run-solves
description: Preview, start, monitor, inspect, stop, and cancel Ahuora solves using durable SolveRun identity and explicit revision checks.
---

# Run Solves

For P-Graph alternative processes, use [analyse-pgraph-alternatives](../analyse-pgraph-alternatives/SKILL.md) for decision modes, generation, structure and row identities, and applying a solved alternative.

In user-facing text, use each returned name, label, display name, or run number
with clear process context. Use raw database IDs only for tool calls and exact
matching. Show an ID to the user only when they ask for it, it resolves real
ambiguity, or no useful name exists. Put it after the plain-language reference,
and never invent a name.

1. Call `set_agent_task_purpose`, then call `get_client_permissions` and
   `get_capabilities`; require solve permission. Django creates and renews the
   visible session from the JWT.
2. If the client permission lacks the target scope or solve capability, call
   `request_client_permission_change` as defined in the shared client
   permission workflow.
3. After approval, follow that workflow to verify the active resource records
   and capabilities for every target flowsheet.
4. Read the current snapshot and retain its content revision and root grouping ID.
5. Call `check_flowsheet_readiness`. Resolve its blockers before solve planning.
6. For a full flowsheet solve, skip `preview_solve_plan`. Call `start_solve` without `selection`, `selectionStrategy`, or `executionPlanSignature`. This uses the normal full solve path.
7. Use `preview_solve_plan` only when the user wants a true subset of the flowsheet. Resolve blockers and retain its execution-plan signature.
8. Call `start_solve` with a fresh request ID and the exact current content revision. For a subset, pass the same selection intent and the validated execution-plan signature. Retain the returned SolveRun ID. Omit `performDiagnostics` or set it to `false` for normal and iterative solves. Set it to `true` only when the user needs the full IDAES diagnostic suite: on a large model or a poor initial point, those checks can take much longer than the solve and consume most of the same task time limit.
9. Use `get_solve_run` for durable progress and outcome. For a long-running solve, give the user periodic updates from both progress layers: the current phase or unit-initialisation stage, and the latest solver iteration, objective, primal infeasibility, and convergence health. Do not report only that the task is still running.
10. Use `list_solve_run_children` for P-Graph alternatives or aggregate multi-steady-state rows. For a multi-period P-Graph run, traverse alternatives first, then each structure’s rows. For a running child whose detailed iterations are needed, call `enable_solve_run_convergence_stream`, then inspect its updated `get_solve_run` progress.
11. If an iteration has `restoration_mode: true`, or convergence health is `restoring`, tell the user at once. IPOPT entered recovery to seek feasibility; this usually indicates that the solve is in trouble. It can recover, so keep the wording diagnostic, but watch whether primal infeasibility improves and whether health becomes stalled or diverging.
12. Read immutable outputs with `get_solve_run_results` and findings with `get_solve_run_diagnostics`.
13. For a solve with recycle cuts, call `get_solve_run_recycle_cut_report` and inspect calculated-versus-captured closure errors.
14. If cancellation is requested and permitted, call `cancel_solve_run` with the SolveRun ID.
15. To end a running parameter fit or design search early and keep its best trial, call
    `stop_solve_run` with the SolveRun ID instead. It needs the same permission as a cancel.
    The run ends as a normal result that says it was stopped, once its best trial passes a
    full-accuracy check; follow it with `get_solve_run`. Any other run can only be cancelled.

## Subset selection

For a manual subset, use `mode: selected_units` and seed fewer than all flowsheet unit IDs. Expansion resolves the full model dependency graph. It follows material streams and also pulls in units coupled by active constraints, variable replacements, formulas, optimization relations, and recycle closure. Select dependency expansion from the process effect that the user needs:

- `none`: Keep the chosen units plus units that the planner must include for coupling or recycle closure. Use this for a local calculation when valid boundary values isolate the selected area.
- `upstream`: Add all transitive feed-side predecessors. Use this when the selected units need upstream state recalculation or their inlet boundary values are not a sound fixed basis.
- `downstream`: Add all transitive product-side successors. Use this to calculate how a local change affects later equipment and outputs.
- `both`: Add both transitive directions. Use this for a tightly coupled process section where both inlet state and downstream effects must change. This can grow to the full connected flowsheet.

Use `requested_recycle_cut_arc_ids` to open eligible material recycle tear edges before expansion. A cut stops graph traversal through that recycle edge, so upstream, downstream, or both-direction expansion does not pull in the whole recycle loop or a larger section than intended. Use cuts to isolate a process segment, debug one side of a recycle, or test recycle convergence. Select only current recycle tear arc IDs. The preview validates each cut and reports the resolved boundary; it does not delete or disconnect the flowsheet stream. If other dependencies still include both ends of the cut, the planner can keep it as an internal logical tear instead of a boundary cut.

Use `selectionStrategy: changed_units` for the “modified process only” mode. Omit `selection`; Ahuora finds semantic edits since the latest successful, applied, base full steady-state SolveRun and adds their downstream closure. Use it for fast recalculation after a small base-flowsheet edit. It does not support scenarios, needs a valid full-run baseline, and fails when no unit changed or the edit history is incomplete.

After any subset preview, compare `included_unit_model_ids` with all active unit IDs in the snapshot. If the plan includes every unit, discard the subset plan and run the normal full solve instead.

Planning failures return `code`, `stage`, `retryable`, `detail`, `field_errors`, and `current_revision`. Correct the stated fields when `retryable` is false. Retry only after a fresh readiness and revision check when `retryable` is true.

Read [the SolveRun contract](references/solve-run-contract.md) before interpreting state.
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.

## Experimental solve options

Each scenario has two experimental options. Both are off by default. Read them
from `get_scenario` or the flowsheet snapshot.

The user may not know these options exist. In the situations below, suggest the
option that fits and say what it does. Set an option only with the user's
permission. Use `update_scenario` with `enable_continuation` or
`enable_elastic_retry`. A change marks the scenario's earlier results as out of
date. Use the screen names with the user.

| Screen name | Field | What it does |
| --- | --- | --- |
| Ramp to new inputs | `enable_continuation` | Skips initialisation and moves the saved solution to the new inputs. If it finds a limit, it reports how far it got and what stopped it. If it stops for another reason, a finding gives only the share it reached, or says that the saved solution no longer solves. |
| Find closest solution | `enable_elastic_retry` | After a failed solve, retries with targets and equipment values free to move, then feeds if needed. Optimisation decisions stay fixed. It reports which values cannot be met and how close each can get. This adds solver runs to a failed solve. |

Both options need a steady-state solve of the full flowsheet with `solver_option`
exactly `ipopt`. Other IPOPT variants do not qualify. Start the solve with
`scenarioNumber` set to the scenario's `id`. A subset or dynamic solve ignores
them.

Ramp to new inputs has more conditions:

- `initialization_strategy` is `initialize_new` or `skip`. New scenarios use
  `initialize_all`, so ask permission for the strategy change and the option
  together.
- A single steady-state solve of the full flowsheet has succeeded, and every
  unit still holds the values it saved. A new unit, restore, or import can break
  this.
- The flowsheet has no active integrated model, because that model's unit never
  holds saved values.

A multi-steady-state solve does not update the saved solution. Each of its rows
ramps from the last single steady-state solve, so that solve must exist.

If a condition is not met, the solve takes its normal path and reports no error.
To confirm that an option ran, look in `get_solve_run` progress for
`solver_continuation` or `solver_elastic_retry`. For a multi-steady-state solve,
call `list_solve_run_children` and read these fields on each child run, because
the parent run does not hold them. `solver_elastic_retry` appears only after a
failed solve. Do not report that an option was used without this evidence.

### When to suggest each option

| Situation | Suggest |
| --- | --- |
| A single steady-state solve succeeded, the user changed inputs such as a feed or set point, and the solve now fails | Ramp to new inputs |
| The user wants to know how far an input can move before something limits it | Ramp to new inputs alone. With both options on, the closest-solution report replaces the ramp's limit report. |
| A solve fails and a target may be out of reach or may conflict with another value | Find closest solution |
| The user asks which specified value to change to make the flowsheet solve | Find closest solution |
| A changed flowsheet fails and the cause is unclear | Both. The ramp runs first, and the closest-solution retry runs if the ramp fails. |

Do not suggest them in these cases:

- **Neither option:** the flowsheet has readiness or degrees-of-freedom blockers,
  because those need a model fix first. The same applies to dynamic and subset
  solves, and to a solver other than `ipopt`.
- **Not Ramp to new inputs:** this is the first solve, only multi-steady-state
  solves have run, units were added, the flowsheet was restored or imported since
  its last full solve, or it has an active integrated model. There is no matching
  solution to start from.
- **Not Find closest solution:** the user only wants a faster solve. It does
  nothing unless a solve fails, and then it adds solver runs.

To read what these options find, follow "Specified values that cannot be met"
in the [debug-flowsheets skill](../debug-flowsheets/SKILL.md).

## Result reads

`get_solve_run_results` returns at most 25 property rows and 5,000 stored
values in one response page. For a dense indexed result, set `pageSize: 1` and
filter by `propertyValueId` when the property is known. Reduce the page size or
add a property or owner filter when the API reports that a page contains too
many values.

An execution-run scalar has `axes: []`. A one-axis result includes its exact
time, space, or stage coordinates. Treat those coordinates as part of the
result identity; do not compare or combine vectors that use different axes.

An aggregate multi-steady-state result has `axes: null` and parallel
`solve_indexes` and `child_solve_run_ids`. Aggregate reads contain scalar child
results only. To inspect an indexed result from a multi-steady-state solve,
call `list_solve_run_children`, then call `get_solve_run_results` for each
required child Solve Run.

## Dynamic run results

A dynamic run is one execution Solve Run with no children. Each result is a
time series on one `time` axis.

- **Long series.** A page of 25 long series holds far more values than one
  read returns (the `get_solve_run_results` description states the limit).
  Pass `maxPoints` (4–2,000), and optionally `timeStart` and `timeEnd` in
  seconds, to `get_solve_run_results`. Each series is cut to the window and
  thinned, and the limit drops further so the page's time series fit that
  one value limit; `point_limit` gives the cap each series got. Every group
  keeps its first, lowest, highest and last point, so peaks and steps
  survive. `downsampled` is true when thinning dropped points inside the
  window, and `stored_point_count` is how many points the run stored.
- **Steady-state results.** A multi-steady-state run has no time axis, so its
  results refuse `maxPoints`, `timeStart` and `timeEnd`.
- **Quote the summary.** Each time series has a `summary` taken from every
  stored point in the window: start and end values, minimum and maximum with
  their times, and the time-weighted mean. Use it for numbers you report, not
  the thinned points.
- **Failed or stopped runs.** Pass `includePartial: true` to read the values a
  run kept up to where it stopped. Call `get_solve_run_failure_report` for the
  stage and simulated time of the failure and the input changes near it.
- **Optimisation.** `get_solve_run_optimization` returns the chosen control
  schedules and event settings, each objective term with its replayed value,
  and how each limit ended. Write the schedules back with an
  `apply_optimised_inputs` edit.
- **Parameter fit.** For a scenario with goal `match_data`, start a check
  with `start_solve` and `fitCheckOnly` true, then the fit without it, and
  read both with `get_solve_run_fit`; follow "Fit a model to measured data"
  in the analyse-scenarios skill.
- **Events and inputs.** `get_solve_run_event_history` lists the events that
  fired. `get_solve_run_input_timeline` shows the inputs the solver received.
- **Out-of-date results.** `get_solve_run_staleness` says whether edits made
  since the run started make its results out of date.
- **Comparing dynamic runs.** `compare_solve_runs` gives point-by-point
  differences only when both runs have the same time grid, and it refuses
  comparisons above 5,000 values. For long runs, or runs with different grids,
  read both with the same window and compare the summaries.
