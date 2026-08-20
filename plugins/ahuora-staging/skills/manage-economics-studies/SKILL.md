---
name: manage-economics-studies
description: Build, inspect, calculate, and evaluate Ahuora economics studies through typed, revision-safe inputs and durable economics result runs.
---

# Manage Economics Studies

Use this skill for techno-economic study setup, cost inputs, schedules, calculation, and result comparison.

1. Call `set_agent_task_purpose`, then call `get_client_permissions` and confirm
   the read, edit, or calculate capabilities needed for the task. Django creates
   and renews the visible session from the JWT.
2. If the client permission lacks the target scope or a needed capability, call
   `request_client_permission_change` as defined in the shared client
   permission workflow.
3. After approval, follow that workflow to verify the active resource records
   and capabilities for every target flowsheet.
4. Call `list_economics_studies`, then read the target with `get_economics_study_snapshot`. Request only the sections and page needed.
5. Call `query_economics_catalog` before you select a profile, schedule source, cost curve, property, default rate, or cost index. Use returned IDs and units. Use the enum values published by the tool input schema; do not guess IDs, units, or enum values.
6. For a scenario production schedule, select only a returned solved MSS time-series source. Confirm that its rows are chronological equal-duration states and that the interval and unit match the real production cadence.
7. Create a study with `create_economics_study`, or copy inputs with `duplicate_economics_study`. A duplicate does not copy result runs.
8. For input edits, use one `edit_economics_study` operation with the exact `configuration_revision`. Call `plan_economics_study_changes`, inspect its summary, then call `apply_economics_study_changes` with that preview ID.
9. Re-read the study after apply. If either revision conflicts, discard the old plan and build a new plan from current data.
10. Call `check_economics_study_readiness`. Review blockers and warnings before calculation. Resolve them when they make the result invalid for the task; if you continue, state each unresolved blocker with the result. Recheck the schedule if its source scenario inputs, solve generation, time-series flag, interval, or unit changed.
11. Call `calculate_economics_study` with the exact current `configuration_revision` and a new request ID. Inspect the pre-calculation `readiness` returned with the calculation and keep the economics result-run ID.
12. Use `list_economics_result_runs` and `get_economics_result_run` for result evidence. Set `includeAudit: true` when the decision needs formula audit rows and paged dependency fingerprints. Record the calculation configuration revision and input fingerprint.
13. Use `compare_economics_result_runs` with exact result-run IDs. Results can
    come from any authorized flowsheets in the selected project; they cannot
    cross projects. Treat the first run as the baseline. Join comparison cells
    by `result_run_id`; use `study_id` only for the source study. Compare the
    operating basis, currency, basis date, warnings, formula audit, chart data,
    configuration revision, and input fingerprint before you accept a delta.
14. Use `undo_economics_study_change` only for this client's latest eligible
    input batch. Re-read the study after undo.

Do not treat mutable study inputs as calculated evidence. A result-run ID is durable, but the current service can refresh a matching run's presentation rows; do not call it an immutable historical snapshot. Do not compare “latest” results when the decision names exact alternatives. Do not hide missing rates, stale solve inputs, schedule faults, or cost-line warnings.

See [the economics workflow](references/economics-workflow.md).
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
