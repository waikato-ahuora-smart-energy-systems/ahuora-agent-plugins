---
name: analyse-scenarios
description: Configure and analyse Ahuora scenarios through revision-safe edits and SolveRun-owned results, including multi-steady-state child runs. Use when Codex must inspect, change, solve, or analyse an Ahuora scenario.
---

# Analyse Scenarios

For P-Graph alternative processes, use [analyse-pgraph-alternatives](../analyse-pgraph-alternatives/SKILL.md) for decision modes, generation, structure and row identities, and applying a solved alternative.

In user-facing text, use each returned name, label, display name, or run number
with clear process context. Use raw database IDs only for tool calls and exact
matching. Show an ID to the user only when they ask for it, it resolves real
ambiguity, or no useful name exists. Put it after the plain-language reference,
and never invent a name.

1. Call `set_agent_task_purpose` with a concise statement of the user's task,
   then call `get_client_permissions` and inspect capabilities for the target
   flowsheet. Django creates and renews the visible session from the JWT.
2. If the client permission lacks the target scope or a capability needed to
   inspect, edit, import, export, calculate, or restore scenario data, call
   `request_client_permission_change` as defined in the shared client
   permission workflow.
3. After approval, follow that workflow to verify the active resource records
   and capabilities for every target flowsheet.
4. Call `get_scenario`. Identify its mode, full configuration, objective, degrees of freedom, input-table state, selected result scope, and coverage. Do not invent fields or IDs.
5. Create or change scenario state only through `plan_flowsheet_changes` and `apply_flowsheet_changes` at the observed content revision.
6. For a parameter sweep, read eligible targets and preview its size before adding `generate_scenario_parameter_sweep` to an edit batch.
7. Enable MSS time series only when rows are chronological operating states with one fixed real-world interval. Keep it disabled for unordered cases, sensitivity studies, and parameter sweeps. Set the interval and unit before you interpret graphs or use the scenario as an economics schedule.
8. Local CSV upload is not available through the hosted plugin. Use the Ahuora web application to upload a CSV, then re-read the scenario before more work.
9. Verify the resulting scenario and revision, then call `check_flowsheet_readiness`.
10. For a normal full solve, call `start_solve` without selection fields. Use `preview_solve_plan` only for a true subset solve. Start with a unique request ID and retain the aggregate or execution SolveRun ID.
11. For multi-steady-state work, use `list_solve_run_children` and match each row by `scenario_id` and `solve_index`, not task order.
12. Read the selected result scope before tables, graphs, or summaries. Use `get_scenario_result_table` to read paged MSS row inputs, outputs, status, and missing-value identity; do not require CSV export for value analysis. Treat pending, partial, and failed summary-cache states as explicit domain outcomes.
13. Analyse immutable results and diagnostics, preserving row ID, root and child SolveRun IDs, scenario, solve index, units, availability, coverage, source state, and source revision.
14. Apply a scenario row only when `get_scenario_input_table` reports that the requested mode is eligible, then verify the new revision.

Read [the scenario workflow](references/scenario-workflow.md) for comparison rules.
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
