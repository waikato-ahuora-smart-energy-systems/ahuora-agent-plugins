---
name: debug-flowsheets
description: Diagnose Ahuora flowsheet structure and solve failures using snapshots, SolveRun evidence, minimal revision-safe corrections, and verification.
---

# Debug Flowsheets

For P-Graph alternative processes, use [analyse-pgraph-alternatives](../analyse-pgraph-alternatives/SKILL.md) for decision modes, generation, structure and row identities, and applying a solved alternative.

In user-facing text, use each returned name, label, display name, or run number
with clear process context. Use raw database IDs only for tool calls and exact
matching. Show an ID to the user only when they ask for it, it resolves real
ambiguity, or no useful name exists. Put it after the plain-language reference,
and never invent a name.

1. Call `set_agent_task_purpose` with the diagnostic task and capture the
   reported symptom without assuming its cause. Django creates and renews the
   visible session from the JWT.
2. Call `get_client_permissions` and read target capabilities. If the client
   permission lacks the scope or capability needed to inspect, correct, solve,
   or restore the flowsheet, call `request_client_permission_change` with a
   fresh request ID as defined in the shared client permission workflow.
3. After approval, follow that workflow to verify the active resource records
   and capabilities for every target flowsheet.
4. Read the current flowsheet snapshot. Use `query_flowsheet_snapshot` for bounded follow-up reads.
5. Save a named revision before an experiment that may need rollback.
6. Call `check_flowsheet_readiness` (with `scenarioId` when a scenario solve fails to start), then `analyze_flowsheet_structure`. Keep each blocker, unmatched partition, and unsupported property ID as direct evidence.
7. Use `trace_flowsheet_dependencies` to explain why suspect units couple. Expansion follows streams, constraints, replacements, formulas, optimisation relations, and recycle coupling. Cut eligible recycle arcs when a loop would include more of the flowsheet than the intended test segment.
8. If a SolveRun is implicated, call `get_solve_run`, `get_solve_run_diagnostics`, and, when relevant, `get_solve_run_results` and `get_solve_run_recycle_cut_report`.
9. Separate topology or specification evidence from solver outcome evidence. Treat the SolveRun as the durable result identity.
10. Propose the smallest typed correction. Preview it against the observed revision and review the impact before applying.
11. Verify the resulting snapshot, readiness, and structural result. Start a new solve only when verification requires computation.
12. Use `compare_solve_runs` to compare the new SolveRun's provenance, status, diagnostics, and selected immutable values to the original evidence.
13. Keep the experiment as a new named revision or restore the earlier checkpoint at the exact current revision.

## Specified values that cannot be met

The experimental options Ramp to new inputs and Find closest solution report
what stops a steady-state solve. See "Experimental solve options" in the
[run-solves skill](../run-solves/SKILL.md) for when they apply. For a
multi-steady-state run, call `list_solve_run_children` first and read the evidence
on each child run, because the parent run does not hold it. Read the evidence in
this order:

1. `get_solve_run`: `progress.terminal_outcome.shortfall_summary` names the
   largest miss and the number of missed values. `progress.solver_continuation.reached`
   is the share of the requested change the ramp reached, from 0 to 1.
2. `get_solve_run_diagnostics`: quote each finding's `title` and `message`. Its
   `specificationShortfall` holds the numbers. A ramp that stopped without
   finding a limit has a finding with no `specificationShortfall`, so quote its
   message.

In `specificationShortfall`:

- `kind` is `summary`, `specification`, or `limit`.
- `source` is `elastic_retry` for the closest point to all values, or
  `continuation` for the furthest point on a straight path to the new inputs.
  A continuation point is not the closest possible solution. With both options
  on, a closest-solution report replaces the ramp's report.
- `specification.role` is `missed`, `involved` (met, but changing it would reduce
  the miss), or `limiting` (a held feed or decision that keeps the miss in place).
- `specification.alternative` says what moving this one value alone would
  achieve: its `value`, whether it `meets_all` targets, and the `gap_closed`.
- `limit` names the bound that holds the miss in place, and its `origin`:
  `property_range`, `physical`, `equipment`, or `model`.
- `limit_count: 0` means the report found no bound holding the point back.

Keep these rules when you report the findings:

- **Closest values are evidence, not a solution.** When values are missed, the
  run stays failed. Its stored values are not a solution, and they are not the
  closest values. After a stopped ramp, they mix the last converged point with
  the inputs at their requested values, so balances may not close. Do not report
  this as a model fault. Quote closest values only from the finding.
- **A closest value is on the same scale as the input, so it can pass 0 or 1.**
  Relative humidity above 100% means water above saturation (fog). A humid-air
  vapour fraction below 1 means fog, and above 1 means unsaturated air. A
  Helmholtz quality can read below 0 or above 1 outside the two-phase region.
- **A suggested change is a proposal.** Show the user the `alternative` or limit,
  apply a change only with their agreement, and confirm it with a new solve.

## Dynamic run failures

A dynamic run fails at start, at the starting state, or partway through.
Gather evidence in this order before you change anything:

1. `get_solve_run` gives `failure_code` and a plain `failure_summary`. Its
   progress shows how far the integration reached.
2. `get_solve_run_failure_report` gives the stage, the simulated time where
   the run stopped, the last time it passed, and the input changes near that
   time. A failed optimisation can also return an answer it found but could
   not confirm; that answer is evidence, never a result.
3. `get_solve_run_results` with `includePartial: true` and `maxPoints` shows
   the values up to the stop. Look for a level, flow or fraction that reaches
   zero, or a value that changes very fast near the stop.
4. `get_solve_run_input_timeline` shows what the solver received. Compare it
   with `get_scenario_timeline` to find a later edit. Both thin long series to
   one bounded read; pass the same `timeStart` and `timeEnd` to compare a
   window in detail.

Map the cause to the smallest correction:

- **Start failure** (`dynamic_start_failed`): a starting value conflicts with
  the inputs. Review `get_scenario_initial_values`.
- **Convergence or bound** (`dynamic_integrator_convergence`,
  `dynamic_integrator_bound`): a vessel runs empty or full, or a flow
  reverses. Change the input profile or event, or the initial values.
- **Accuracy or step limit** (`dynamic_integrator_accuracy`,
  `dynamic_integrator_step_limit`): a sudden input change or a controller
  tuned too hard. Ramp the change, or retune the PID gains.
- **Optimisation infeasible**: read `get_solve_run_optimization` for the limit
  report, and widen decision bounds or soften a limit's firmness.

Change tolerances in `dynamics_options` only after the model evidence rules
out these causes.

Use [the diagnostic workflow](references/diagnostic-workflow.md) to keep conclusions evidence-bound.
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
