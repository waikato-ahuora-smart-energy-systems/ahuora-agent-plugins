---
name: analyse-pgraph-alternatives
description: Build, generate, solve, compare, and apply Ahuora P-Graph alternative processes, including mixed decision streams and multi-period structure runs.
---

# Analyse P-Graph Alternatives

Use the source flowsheet as the superstructure. Each generated alternative owns a retained state. Solving creates its SolveRun; it does not replace Current until explicitly applied.

## Set up the alternatives

1. Set the task purpose, resolve the flowsheet by name, and check its client permissions and capabilities. Inspect needs read access; topology changes need edit; generation and solves need calculate; applying a structure needs restore. Use [the shared permission workflow](../../references/client-permission-workflow.md) if access is missing.
2. Read the current snapshot and unit catalog. Retain the Current state ID, content revision, root grouping, stream IDs, and decision modes.
3. Use [build-flowsheets](../build-flowsheets/SKILL.md) for topology and values. Connect alternative routes through the normal stream merge path. `convert_stream_to_decision_stream` converts a connected material stream to the current `decision_stream` type. The retained stream ID is the decision identity; inspect its port links after apply.
4. Set `update_object.changes.decision_mode` to `exclusive` when each connected side must select one route, or `mixed` when combinations of routes may operate together. Mixed mode belongs only to a decision stream. Preview and apply at the exact revision, then read the snapshot to verify the mode and connections. Use `merge_streams` to join decision streams.
5. Select required products and disposable materials from current stream IDs using the rules below. These settings control different parts of the process and may include the same stream.

## Required products and disposable materials

**Required products define what every alternative must deliver.** P-Graph can remove process routes when it builds alternatives. Select the outputs that must remain so it cannot choose a smaller process that fails the task. This can include an intermediate stream when its production must remain part of every alternative. Requiring a stream retains a route that produces it; it does not require one specific producer when several routes can supply it. Set flow, temperature, and other required values through normal process specifications; the product selection alone does not set a production rate.

For example, compare heat recovery with boiler heating at a factory. Require both the factory's main product and the useful heat output. If only useful heat is required, a boiler-only alternative could omit the factory process and still meet that selected goal. Requiring the main product keeps production in the comparison while P-Graph varies how useful heat is supplied.

**Disposable materials allow a produced intermediate stream to leave without downstream use.** Every stream produced by retained equipment stays in the alternative. A disposal mark permits P-Graph to omit the equipment that would consume that stream. It does not delete the stream or its material balance. Use this for an intermediate that can be rejected instead of recovered. Source terminal outlets already leave the process without a disposal mark.

In the factory example, mark the waste heat stream feeding the recovery exchanger as disposable. P-Graph can then compare recovering that heat with leaving it unused and using the boiler. The waste heat stream remains when the hot process produces it, even when the recovery exchanger is omitted.

A stream can be **both required and disposable**: it must be produced in every alternative, but downstream use is optional. Use both when the study must retain a waste heat source while testing whether recovery is worthwhile. Disposal alone does not require the upstream process to operate; if another valid route omits that producer, its unproduced stream is absent for that reason, not because disposal removes produced material.

Before marking an ordinary intermediate stream disposable, use the typed conversion operation to make it a decision stream, as the UI does. Read back its identity and connections, then use the latest revision for generation. Resolve selections from the user's production goals and allowed rejection routes; do not mark every outlet required or treat every unselected stream as disposable.

## Generate and solve

Call `start_solve` with the current revision, a new request ID, root group, and `pgraphRequiredProductIds`. Set `pgraphDisposableMaterialIds` as needed and `pgraphMaxStructures` from 1 to 100 (default 25).

- To inspect alternatives before spending solve time, set `pgraphGenerateOnly: true`. Generation ignores scenario execution and leaves structures ready to solve. A completed generation does not prove feasibility.
- To solve alternatives, omit `pgraphGenerateOnly` or set it to false. Supply `scenarioNumber` for an existing steady-state, optimisation, or multi-period scenario. Ahuora may reuse a matching generated topology and creates the solve-owned run tree. Use a new request ID when changing intent; an identical retry uses the original ID.
- Superstructures do not support dynamic, heat-exchanger-network, selected-unit, or changed-unit solves. Omit subset selection and plan signatures. Do not run subset structural analysis on the whole superstructure to establish each alternative's feasibility.
- Keep deep diagnostics off for routine solves. Correct formula or configuration rejections at Current through typed edits, then use the new revision. Do not edit retained alternatives.

Generation-only requests return a task `id` and `status`, with `solve_run: null`. Keep Current unchanged and replay the same `start_solve` request ID and exact inputs to read that task's status without starting it again. Wait for completion before requesting a solve. Inspect generated previews in the P-Graph panel; they have no SolveRun or solved results to read through solve tools. A new generation request is blocked while this flowsheet is generating or solving alternatives.

For solve requests, poll `get_solve_run` on the returned `solve_run.id`. Its aggregate progress distinguishes generating from solving alternatives. Keep task status, solve status, and result status separate. Report `structures_truncated` with any ranking: a limited set does not prove a global best result.

## Read and compare results

Use `list_solve_run_children` on the P-Graph root to list alternatives. A steady-state structure is an execution run. A multi-period structure is an aggregate; call `list_solve_run_children` again to reach its row executions. Use `structure_index` for the alternative and `solve_index` for the row. Report user-facing structure numbers as index plus one, along with the returned scenario name and run number.

Read results and diagnostics from the execution or multi-period structure run with `get_solve_run_results` and `get_solve_run_diagnostics`. The outer P-Graph root is not a result vector. A failed or cancelled alternative can coexist with successful siblings. Generated-only structures have no solved values. For detailed iterations on a running multi-period row, use the convergence stream tool described in [run-solves](../run-solves/SKILL.md).

Compare two to five structure execution runs, or two to five multi-period structure aggregates, with `compare_solve_runs`. Do not compare outer P-Graph roots or mix aggregates with executions. Read each run's own property IDs first; cloned alternatives can use different IDs and omit equipment. Match by returned names, property labels, units, and physical meaning. Keep row indices and child-run arrays aligned. Use [compare-solve-runs](../compare-solve-runs/SKILL.md) for the comparison contract.

For economics, list result runs for the source study. Use `evaluation_id`, `evaluation_label`, and `source_solve_run_id` to select the alternative's exact result. Compare exact economics result IDs, not the study's latest result. Read warnings, configuration revision, currency, and operating basis with [manage-economics-studies](../manage-economics-studies/SKILL.md). Missing or blocked economics results are not zero cost.

## Apply one alternative

Apply only when the user asks to make a chosen solved alternative Current. Check `can_apply_structure`, then reread Current's state ID and revision. Call `apply_pgraph_structure` with `structureStateId`, the returned `structure_solve_run_id`, and those exact Current identities. For a multi-period structure, choose an explicit successful `rowSolveRunId` whose values will initialise Current.

The server checks ownership, successful complete results, active tasks, and the source revision. It saves an automatic recovery revision and replaces Current atomically. Report the returned backup revision and warnings. Reread the snapshot; all state-owned IDs can change. On rejection, refresh and resolve the stated cause rather than reusing stale IDs. Applying a structure does not mean the new Current has a fresh solve result.
