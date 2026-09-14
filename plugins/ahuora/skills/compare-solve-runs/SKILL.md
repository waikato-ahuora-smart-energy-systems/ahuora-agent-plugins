---
name: compare-solve-runs
description: Compare immutable results, provenance, statuses, and diagnostics from two or more Ahuora SolveRuns without using mutable live values.
---

# Compare Solve Runs

In user-facing text, use each returned name, label, display name, or run number
with clear process context. Use raw database IDs only for tool calls and exact
matching. Show an ID to the user only when they ask for it, it resolves real
ambiguity, or no useful name exists. Put it after the plain-language reference,
and never invent a name.

1. Call `set_agent_task_purpose`, then call `get_client_permissions` and confirm
   read capability for each flowsheet. Django creates and renews the visible
   session from the JWT.
2. If the client permission lacks a target scope or read capability, call
   `request_client_permission_change` as defined in the shared client
   permission workflow.
3. After approval, follow that workflow to verify the active resource records
   and capabilities for every flowsheet in the comparison.
4. Call `compare_solve_runs` with two to five `{flowsheetId, solveRunId}` targets.
   For each property, supply one `{solveRunId, propertyValueId}` selection per
   target. Property IDs can differ between flowsheets. Treat the first target as
   the baseline.
5. Verify each run's source revision, scenario, run kind, plan, unit scope, and separate status fields before accepting value deltas.
6. Use `get_solve_run`, `get_solve_run_results`, or `get_solve_run_diagnostics` for detail that the bounded comparison does not include.
7. Compare only compatible values and units. Report missing properties, unknown units, different vector lengths, or different execution scopes explicitly.
8. For recycle experiments, compare each run's `get_solve_run_recycle_cut_report` as well as its process outputs.
9. Include diagnostics and provenance in the conclusion; do not present a value delta without its run context.

See [the comparison contract](references/comparison-contract.md).
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
