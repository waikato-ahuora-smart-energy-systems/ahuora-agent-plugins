---
name: run-solves
description: Preview, start, monitor, inspect, and cancel Ahuora solves using durable SolveRun identity and explicit revision checks.
---

# Run Solves

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
10. Use `list_solve_run_children` for aggregate multi-steady-state rows. For a running child whose detailed iterations are needed, call `enable_solve_run_convergence_stream`, then inspect its updated `get_solve_run` progress.
11. If an iteration has `restoration_mode: true`, or convergence health is `restoring`, tell the user at once. IPOPT entered recovery to seek feasibility; this usually indicates that the solve is in trouble. It can recover, so keep the wording diagnostic, but watch whether primal infeasibility improves and whether health becomes stalled or diverging.
12. Read immutable outputs with `get_solve_run_results` and findings with `get_solve_run_diagnostics`.
13. For a solve with recycle cuts, call `get_solve_run_recycle_cut_report` and inspect calculated-versus-captured closure errors.
14. If cancellation is requested and permitted, call `cancel_solve_run` with the SolveRun ID.

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
