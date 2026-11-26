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

## Fit a model to measured data

A fit changes chosen constant values (parameters), such as a heat transfer
coefficient, until a dynamic run matches plant readings. Follow these steps in
order. Make each change with `plan_flowsheet_changes` and
`apply_flowsheet_changes`.

1. **Map the measured values.** Call `get_scenario_measurements` to see the
   free data columns. Check each property with
   `get_scenario_property_candidates` and `purpose` `"measurement"`, then add a
   `set_scenario_measurement` operation for each column. State
   `measurement_error` when you know the instrument accuracy. A column whose
   values do not change needs one.
2. **Compare first.** Run a plain dynamic simulation and read
   `get_solve_run_fit`. Each measured value shows its typical error, bias,
   quality and status. Fix setup problems, such as a time shift or a sample
   window, before you fit.
3. **Check against another day.** One day of data can be matched by wrong
   values. Add a second day of data as a data set: upload the CSV as for a
   scenario CSV import, and when the upload is complete send
   `add_scenario_data_set` with its `upload_session_id`. Do not start an
   import for it. A file that is already stored can be added by its project
   `file_id` (`get_scenario_data_sets` lists the `file_id` of each data set
   that exists). Give one data set the role `"check"` with
   `update_scenario_data_set`, so the fit leaves it out and reports how the
   fitted values do on it. Read `get_scenario_data_sets` and fix every
   problem before you fit. An input that a data set has no column for needs
   a choice: send `set_scenario_data_set_choice` with the `slot` (the
   scenario data column name) and the file `column` to use. A measured value
   that a file does not record can take `leave_out` true. A fit reads every
   data set, so do not show one first; `show_scenario_data_set` is only for
   a plain run on another day.
   `add_scenario_data_set`, `remove_scenario_data_set` and
   `set_scenario_data_set_choice` edit the scenario and can import CSV rows
   into it, so they need both the flowsheet edit permission and the
   scenario CSV import permission. Send them in a plan of their own, apart
   from other edits.
4. **Set the goal and the parameters.** Turn on `enable_optimization` with
   `update_scenario`. Send `configure_scenario_optimization` with `goal`
   `"match_data"`, no objective, and one row for each parameter with both
   bounds (1 to 10 rows). Check each parameter with
   `get_scenario_property_candidates` and `purpose` `"parameter"`. Continue
   only when `fit_problems` from `get_scenario_measurements` is empty.
   `fit_notices` name saved rows the fit leaves out; they do not block it.
5. **Check the parameters.** Call `start_solve` with `fitCheckOnly` true. Read
   `get_solve_run_fit`. A parameter with no effect, or one tied to another,
   cannot be found from this data. Set `held_fixed` on its row before the fit.
6. **Fit.** Call `start_solve` without `fitCheckOnly`. If the user wants the
   fit to end early, call `stop_solve_run`: it keeps the best trial so far.
7. **Read before you apply.** Read `get_solve_run_fit`. For each parameter,
   read `status_label`, `status_note` and the range. Read the scaled error
   before and after, `outcome_label` and `starts_label`. Tell the user about
   each parameter that is poorly defined, tied or at a range edge, and each
   measured value with poor quality, before you apply. With several data
   sets, read each entry of `fit.data_sets` for its scaled error before and
   after. A `"check"` data set that gets worse means the fit matched noise
   or the model misses a behaviour: say so, and do not apply without the
   user's agreement.
8. **Fit again to leave a value out.** Set `held_fixed` on its row (and
   `initial_guess` for a different value), then fit again. Do not apply part
   of a fit: the fitted values belong together.
9. **Apply all values in one edit.** First read `get_solve_run_staleness`
   for the fit run. If the run is out of date, tell the user what changed
   since the fit started: the fit did not see those changes. Apply only if
   the user agrees, and then set `confirm_out_of_date` to true. Without it,
   an out-of-date apply is refused. Send one `apply_fitted_parameters`
   operation with the fit run. It writes every fitted value or none. Then
   re-read the scenario.
