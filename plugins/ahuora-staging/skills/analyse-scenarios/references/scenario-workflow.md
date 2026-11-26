# Scenario workflow

## Choose the mode

- Use `steady state` for one operating point.
- Use `mss` for ordered independent operating rows. A row's persisted `index` becomes the child SolveRun `solve_index`.
- Enable MSS time series only when those rows form a chronological sequence with one fixed real-world interval. Row time is `row index × interval`; index zero is elapsed time zero. Keep it disabled for unordered operating cases, design alternatives, sensitivity studies, and parameter sweeps.
- Use `dynamic` for time-dependent execution (see "Dynamic scenarios"). Dynamic rows represent time steps and cannot be applied back through the MSS row-application operation.
- `enable_optimization` adds an objective and degrees of freedom to a compatible solve. Parameter-sweep generation is unavailable while optimisation is enabled.

## Inspect before changing

Call `get_scenario` and retain its `content_revision`. It returns the complete supported setup:

- scenario mode and solver settings;
- optimisation objective and direction;
- the complete degrees-of-freedom set and bounds;
- saved parameter-sweep definition;
- input column, row, and cell counts;
- dynamic settings with their effective options, and counts of profiles, events, initial-value overrides and active data columns;
- selected immutable result root, signed canvas scope, and row coverage.

Use `get_scenario_input_table` for bounded row data. Use only current property, object, scenario, and row IDs from these reads or a current catalog.

## Mutate through one boundary

All scenario mutations use `plan_flowsheet_changes`, followed by `apply_flowsheet_changes` for that exact preview:

- `create_scenario` and `update_scenario` change basic metadata and solve settings, including the experimental options in the run-solves skill;
- `delete_scenario` removes the scenario and its owned input and selected-result links;
- `configure_scenario_optimization` replaces the objective, direction, and complete degrees-of-freedom set;
- `replace_scenario_input_table` atomically replaces all mapped columns and rows;
- `generate_scenario_parameter_sweep` stores the sweep definition and replaces the input table;
- `apply_scenario_row` applies eligible MSS input values, with optional results, to Current;
- `set_scenario_input_source`, `remove_scenario_input_profile`, `set_scenario_event`, `remove_scenario_event` and `set_scenario_initial_values` change a dynamic scenario's inputs and events (see below);
- `apply_optimised_inputs` writes a dynamic optimisation run's control schedules back as profiles;
- `set_scenario_measurement` and `remove_scenario_measurement` change the measured values a dynamic run compares with its results, and `apply_fitted_parameters` writes every fitted value of a fit run into the flowsheet (see "Fit a model to measured data" in the skill; an out-of-date run needs the user's agreement and `confirm_out_of_date`);
- `add_scenario_data_set`, `remove_scenario_data_set`, `update_scenario_data_set` and `set_scenario_data_set_choice` change a dynamic scenario's data sets (see "Data sets" below).

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

## Dynamic scenarios

A dynamic scenario integrates the flowsheet over time. Set it up in this order:

1. **Run length and start.** Use `update_scenario`. `simulation_length` and `simulation_length_units` set the run length, and `num_time_steps` sets how many reported intervals the run has. With `time_horizon_source` `from_data`, the run lasts as long as the data file plus `data_hold_seconds`. `initial_state_source` `previous_run` starts from the final state of `initial_state_run_id`, a successful dynamic run of the same scenario. `dynamics_options`, `dynamic_input_options` and `dynamic_optimization_options` each replace the stored object; fields left out take defaults. `get_scenario` reports the effective values.
2. **Inputs.** Each time-varying input has exactly one source: the unit value, a data-file column, a profile, or an event action. `get_scenario_profiles` shows each input's current `source` and whether it can take a profile. It returns one page of inputs; pass `equipmentId` to list one unit's inputs, and read `target_page` for the page count. `get_scenario_events` pages its saved events (`event_page`), watchable properties and settable inputs the same way. Use a profile (`step`, `ramp`, `segments` or `function`) for a planned change, a data-file column for measured data, and an event for a change that depends on the process state. `set_scenario_input_source` gives an input its one source (`unit`, `profile` or `column`) in one edit and retires its other sources: other profiles are switched off and its data columns are unmapped.
3. **Events.** An event has a time or property trigger, a firing policy and ordered actions (`set_input`, `start_profile`, `marker`, `stop`). Remove an event's optimisation rows before you remove the event.
4. **Initial values.** `get_scenario_initial_values` lists each initial condition with the unit value and the effective value. `set_scenario_initial_values` overrides it for this scenario only; a null value removes the override.
5. **Check.** Call `preview_scenario_profile` or `preview_scenario_event` for each draft before adding it to an edit. Then call `get_scenario_timeline` to see every input and event on one time axis, and fix any findings before solving. All series together return at most 5,000 points, so a long data-file column comes back thinned (`downsampled` is true and `stored_point_count` gives the full count). Pass `timeStart` and `timeEnd` in seconds to read part of the run in detail.

### Data sets

A dynamic scenario can hold up to `max_data_sets` data sets. Each data set is one stored CSV file of plant data with a role in a fit: `"fit"` (the fit matches it) or `"check"` (the fit leaves it out and then reports how the fitted values do on it). Inputs, profiles, events, initial values, solver settings and time settings stay shared across data sets.

- `get_scenario_data_sets` lists each data set with its role, `shown` flag, rows, row spacing, span and `problems`, and each mapping slot (a scenario data column name that drives an input, holds a measured value or gives the time) with the column each data set uses and how: `same_name`, `chosen`, `left_out` or `not_found`. A fit refuses to start while any data set lists a problem.
- `add_scenario_data_set` takes the `upload_session_id` of a completed scenario CSV upload, or the project `file_id` of a stored CSV file, and matches its columns by name. The first data set of a scenario is also shown. The result gives `data_set_id`.
- `update_scenario_data_set` changes the role or the name. A scenario with goal `match_data` keeps at least one `"fit"` data set.
- `set_scenario_data_set_choice` names the `slot` and the file `column` to use when a file names a column differently. A null `column` goes back to the column of the slot's name. `leave_out` true leaves a measured value out of that data set; an input and the time column cannot be left out.
- `remove_scenario_data_set` removes one. Removing the shown data set shows the next; removing the last clears the scenario data.
- `show_scenario_data_set` (a tool, not an edit operation) queues the import of one data set's file into the scenario data table, so a plain dynamic run uses it. Undo does not reverse it; show the earlier data set again to go back. Its `import_task_id` is not readable with `get_scenario_csv_import`: read `get_scenario_data_sets` until that data set has `shown` true, then re-read the scenario. A fit reads every data set, so it needs no show.
- With more than one data set, `replace_scenario_input_table` is refused for that scenario; use the data set operations.

`get_solve_run_fit` gives one measurement row per data set (`data_set_id`, `data_set_name`) and, for a fit, `fit.data_sets` with each data set's scaled error before and after.

PID controllers read and drive properties through links. Set `changes.linked_property` on the controller's `measured_var` or `manipulated_var` with `set_property_value`. The flowsheet snapshot shows `linked_property`, `driven_by_id` and `measured_by`.

After a dynamic optimisation, `apply_optimised_inputs` writes the chosen control schedules back as profiles. Use `destination` `new_scenario` to keep the source scenario unchanged.

## Optimisation

`configure_scenario_optimization` is the one operation for optimisation in every scenario kind. It is a complete replacement, not a patch. Supply:

- `objective_property_id`, the steady-state objective PropertyInfo ID, or null to clear it;
- `minimize` as true for minimisation or false for maximisation;
- `dynamic_objectives`, for a dynamic scenario only: the weighted objective terms. A property term (`{"kind": "property", "property_id": ...}`) uses `integral`, `terminal`, `average` or `peak`. An event term (`{"kind": "event_firing_time", "event_id": ...}`) uses `event`. If a term's property was deleted, `get_scenario` returns it with `"property_id": null`; give it another property or leave it out. A dynamic solve ignores `objective_property_id`;
- `degrees_of_freedom`: every decision and limit to keep. Each row names a `target`, either `{"kind": "property", "property_value_id": ...}` or `{"kind": "event", "event_id": ..., "field": ...}`, with optional bounds and unit. `get_scenario` returns each row's stored `unit`, where empty means the target's own unit, and a read-only `display_unit` that resolves it. Send `unit` back as read and leave `display_unit` out.

The backend decides whether a row is a decision or a limit. A free input is a decision. A calculated result, or an input that a profile, data column or event already drives, is a limit. In a dynamic scenario:

- a decision can use `decision_type` `piecewise_constant` or `piecewise_linear` with `control_times` in seconds. The last control time must equal the run length. `max_move` and `max_rate` limit changes between intervals;
- a limit can use `constraint_timing` `path`, `window` (with `window_start` and `window_end` in seconds) or `terminal`;
- only a limit can use `firmness` other than `must_hold`, and `breach_scale`.

Rows that you leave out are removed, including event rows. To change one row, call `get_scenario`, change that row, and send back every row: `get_scenario` returns each row's `target` and settings in the shape this operation accepts. Leave out the read-only `id`, `role`, `label` and `simulation_object_id` of each row, and the read-only `label` of each dynamic objective term. Verify the complete returned configuration before solving.

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
