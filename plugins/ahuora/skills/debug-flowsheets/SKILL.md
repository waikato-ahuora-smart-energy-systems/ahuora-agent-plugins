---
name: debug-flowsheets
description: Diagnose Ahuora flowsheet structure and solve failures using snapshots, SolveRun evidence, minimal revision-safe corrections, and verification.
---

# Debug Flowsheets

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
6. Call `check_flowsheet_readiness`, then `analyze_flowsheet_structure`. Keep each blocker, unmatched partition, and unsupported property ID as direct evidence.
7. Use `trace_flowsheet_dependencies` to explain why suspect units couple. Expansion follows streams, constraints, replacements, formulas, optimisation relations, and recycle coupling. Cut eligible recycle arcs when a loop would include more of the flowsheet than the intended test segment.
8. If a SolveRun is implicated, call `get_solve_run`, `get_solve_run_diagnostics`, and, when relevant, `get_solve_run_results` and `get_solve_run_recycle_cut_report`.
9. Separate topology or specification evidence from solver outcome evidence. Treat the SolveRun as the durable result identity.
10. Propose the smallest typed correction. Preview it against the observed revision and review the impact before applying.
11. Verify the resulting snapshot, readiness, and structural result. Start a new solve only when verification requires computation.
12. Use `compare_solve_runs` to compare the new SolveRun's provenance, status, diagnostics, and selected immutable values to the original evidence.
13. Keep the experiment as a new named revision or restore the earlier checkpoint at the exact current revision.

Use [the diagnostic workflow](references/diagnostic-workflow.md) to keep conclusions evidence-bound.
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
