# Scenario workflow

## Choose the mode

- Use `steady state` for one operating point.
- Use `mss` for ordered independent operating rows. A row's persisted `index` becomes the child SolveRun `solve_index`.
- Enable MSS time series only when those rows form a chronological sequence with one fixed real-world interval. Row time is `row index × interval`; index zero is elapsed time zero. Keep it disabled for unordered operating cases, design alternatives, sensitivity studies, and parameter sweeps.
- Use `dynamic` for time-dependent execution. Dynamic rows represent time steps and cannot be applied back through the MSS row-application operation.
- `enable_optimization` adds an objective and degrees of freedom to a compatible solve. Parameter-sweep generation is unavailable while optimisation is enabled.

## Inspect before changing

Call `get_scenario` and retain its `content_revision`. It returns the complete supported setup:

- scenario mode and solver settings;
- optimisation objective and direction;
- the complete degrees-of-freedom set and bounds;
- saved parameter-sweep definition;
- input column, row, and cell counts;
- selected immutable result root, signed canvas scope, and row coverage.

Use `get_scenario_input_table` for bounded row data. Use only current property, object, scenario, and row IDs from these reads or a current catalog.

## Mutate through one boundary

All scenario mutations use `plan_flowsheet_changes`, followed by `apply_flowsheet_changes` for that exact preview:

- `create_scenario` and `update_scenario` change basic metadata;
- `delete_scenario` removes the scenario and its owned input and selected-result links;
- `configure_scenario_optimization` replaces the objective, direction, and complete degrees-of-freedom set;
- `replace_scenario_input_table` atomically replaces all mapped columns and rows;
- `generate_scenario_parameter_sweep` stores the sweep definition and replaces the input table;
- `apply_scenario_row` applies eligible MSS input values, with optional results, to Current.

Re-read after apply. On a revision conflict, discard the preview, inspect the new state, and decide whether the intended change still applies.

Input-table replacement and parameter-sweep generation invalidate older selected scenario results and cached summaries. The new configuration needs a new solve.

CSV export is not supported in this version.

## Import a local CSV

The hosted plugin cannot send local file bytes with the MCP server's identity. Upload the CSV in the Ahuora web application, then re-read the scenario and continue from the current content revision.

## Parameter sweeps

1. Call `get_scenario_parameter_sweep_targets`.
2. Select exact returned property-value IDs and their units.
3. Call `preview_scenario_parameter_sweep`.
4. Inspect row count, warning threshold, hard limit, generated seed, units, and per-parameter value count.
5. Add the same deterministic specification to a `generate_scenario_parameter_sweep` edit operation. Monte Carlo requires an explicit seed.
6. Set `confirm_replace` only after accepting replacement of existing input data. Set `confirm_large_generation` only after accepting a preview above the warning threshold.

Grid sweeps require a step for every parameter. Monte Carlo, Hammersley, and Halton sweeps require a sample count and do not use steps.

## Optimisation

`configure_scenario_optimization` is a complete replacement, not a patch. Supply:

- the objective PropertyInfo ID, or null to clear it;
- `minimize` as true for minimisation or false for maximisation;
- every intended degree of freedom as a current property-value ID with optional lower and upper bounds.

Do not retain an omitted degree of freedom. Verify the complete returned configuration before solving.

## Solve correctly

Call `check_flowsheet_readiness` after configuration.

For a normal full scenario solve, do not call `preview_solve_plan`. Call `start_solve` without `selection`, `selectionStrategy`, or `executionPlanSignature`.

Use `preview_solve_plan` only when the intended solve contains fewer than all active units. Inspect the exact unit scope and blockers, then start that exact subset plan.

Retain the returned SolveRun ID. For MSS:

- the root is the aggregate identity;
- `list_solve_run_children` returns row runs;
- match children by scenario and `solve_index`, never by task creation or completion order;
- enable convergence streaming only for a running child that needs detailed iteration analysis.

Report partial failure explicitly. A successful child does not make a root with failed or cancelled siblings wholly successful.

## Analyse immutable results

Call `get_scenario_result_scope` before object tables, graphs, or summaries. Report:

- root SolveRun ID and child SolveRun ID when relevant;
- scenario ID and `solve_index`;
- solve, result, and projection status separately;
- expected, succeeded, failed, cancelled, with-values, and without-values row counts;
- source state and revision;
- units and unknown-unit flags.

Use `get_scenario_result_table` for row-level object values and availability. Page by simulation object and keep each row's `row_id`, `solve_index`, child SolveRun ID, solve status, inputs, outputs, and missing-value identity together. This is the supported MSS value-table path; CSV export is not needed. A missing result is not zero and is not the same as a legitimate nullable value.

Use `get_scenario_result_graph` for ordered series. Use `get_scenario_result_summaries` for statistics and histograms. These tools can return `pending`, `calculating`, `partial`, or `failed`; report that state and its errors instead of treating it as a transport failure.

For an MSS time series, label and interpret the graph x-axis as elapsed time in the configured unit. For other MSS scenarios, treat the x-axis as solve index. Do not describe solve index as time when time-series mode is disabled. A timing-setting change keeps immutable solve values attached, but it changes their time interpretation, invalidates cached summaries, and makes linked economics studies stale.

## Use MSS as an economics production schedule

An economics production schedule must be an MSS scenario with time series enabled, at least one input row, and a selected solve generation in which every input-row index has a successful child result. Set the interval to the duration represented by each operating row before solving and before selecting the schedule.

The schedule length is `row count × interval`. Economics derives annual operating cycles from annual operating hours divided by that schedule length. Use `query_economics_catalog` with `schedule_sources` to select a returned compatible scenario; do not guess or force a scenario ID. If any row is missing, failed, or cancelled, solve the incomplete rows or select another schedule.

Use SolveRun result and comparison tools for raw immutable values and cross-run deltas. The first comparison run is the baseline, and values are comparable only when units and vector lengths match.

## Apply one MSS row

Read the row through `get_scenario_input_table`. Use:

- `inputs` only when `can_apply_inputs` is true;
- `inputs_and_results` only when `can_apply_inputs_and_results` is true.

Apply through a previewed edit. Row application changes Current and advances the content revision. Verify the resulting snapshot before more work.
