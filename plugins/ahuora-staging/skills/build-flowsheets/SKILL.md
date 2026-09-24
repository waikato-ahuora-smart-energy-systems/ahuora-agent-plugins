---
name: build-flowsheets
description: Build or modify Ahuora flowsheets through revision-safe typed edit previews and exact applications. Use for topology, unit metadata, property values, controls, and scenario configuration.
---

# Build Flowsheets

For P-Graph alternative processes, use [analyse-pgraph-alternatives](../analyse-pgraph-alternatives/SKILL.md) for decision modes, generation, structure and row identities, and applying a solved alternative.

In user-facing text, use each returned name, label, display name, or run number
with clear process context. Use raw database IDs only for tool calls and exact
matching. Show an ID to the user only when they ask for it, it resolves real
ambiguity, or no useful name exists. Put it after the plain-language reference,
and never invent a name.

1. Call `set_agent_task_purpose` with a concise statement of the user's task,
   then call `get_client_permissions`. Django creates and renews the visible
   session from the JWT. Use `list_projects`, `get_project`,
   `list_project_folders`, and `get_flowsheet_identity` to resolve user-facing
   names to stable IDs. If the active scope includes more than one flowsheet and
   the user did not name one, ask which flowsheet to use.
2. If the current client permission lacks the scope or capability needed for the
   task, call `request_client_permission_change` as defined in the shared client
   permission workflow. `write_workspace` is a general permission that is
   compatible with every resource scope. When the user asks for a new project,
   or implies one by asking to build a flowsheet without naming or scoping an
   existing target, request `write_workspace: yes`. Request workspace scope as
   well when the task must continue by editing the new project's default
   flowsheet because a narrower scope cannot include that future project.
3. After approval, follow the shared workflow to verify the active resource
   records and capabilities for every target flowsheet.
4. For an explicit or implied new project, verify `write_workspace: yes`, then
   call `create_project` with a fresh request ID. Project creation always creates
   one default flowsheet. Use that flowsheet ID and root grouping ID for the rest
   of the work; do not call `create_flowsheet` for the project's first flowsheet.
   To add another flowsheet to an existing approved project, verify project scope
   and `manage_project: yes`, then call `create_flowsheet`. To copy a flowsheet,
   call `duplicate_flowsheet` only when the same project is in scope. This tool
   cannot copy a flowsheet to another project.
5. Call `get_capabilities` for the selected or new flowsheet. Continue only when `decisions["edit.flowsheet"]` is `yes` or after Ahuora approves an `ask` decision.
6. Read `get_unit_catalog` before choosing unit types, ports, property keys, or index sets. Read `ahuora://guidance/property-packages/v1` before choosing a package; the catalog does not encode every package-use rule.
7. Read `get_flowsheet_snapshot` and retain its `content_revision`.
8. Use `get_compound_catalog` after a stream has an ID and before selecting compounds. Set the property package and compounds once on an upstream material stream or its owning unit through the standard edit path. Let compound propagation update connected downstream streams. Do not send one package or compound edit per downstream stream. Use another source edit only for a disconnected network or an explicit package boundary, then verify the propagated package and compounds in the snapshot.
9. Use `create_flowsheet_revision` for a named checkpoint before a broad edit or experiment, and at useful gradual milestones. Revisions preserve experiment states without creating a new flowsheet identity.
10. Never send `enabled` in `set_property`, `set_property_value`, or `configure_stream` changes. Agents must not toggle property enablement. To select a different structural input, call `analyze_flowsheet_structure`, choose one fixed property and one valid free match, and preview `replace_control` with the fixed property as `manipulated_id`, the free property as `set_point_id`, and structural validation enabled. Set the selected input's value only after the replacement exists.
11. Never use `add_unit` for a catalog entry whose `is_stream` value is true, or for a recycle object. Normal unit creation generates streams on ports whose `creates_stream` value is true. Add the owning unit and inspect its generated streams in the preview. Use `connect_port` only to join boundary feed/product streams or batch-created unit ports. To close a material recycle loop, connect the return stream to the receiving unit's generated inlet with `connect_port`. The normal merge path detects the loop, keeps the correct stream, creates the recycle object, and selects its tear stream. Never add or wire a recycle object yourself. After the loop exists, use `configure_recycle` to change that auto-created recycle's tear. Set `fixed_tear_variables` only for a fully closed material circuit that needs a physical specification, or for a short-lived diagnostic experiment. Each fixed tear property disables its matching recycle equality and does not converge to the upstream value. Do not fix a tear property on an open loop. For debugging, prefer a true subset solve with a previewed execution plan over changing the full flowsheet's recycle equations. `configure_recycle` does not create a recycle. To place a one-inlet, one-outlet unit in an already connected stream, use `insert_unit`. To place one named inlet/outlet side of an existing multi-sided unit, such as a heat exchanger, use `insert_unit_port_pair` once per side. Do not connect a generated port stream directly to an intermediate process stream because normal merge semantics create a branch decision stream.
12. Form a small, typed batch and call `plan_flowsheet_changes` with a fresh request ID and the observed revision.
    The editing contract lists all supported topology, restore, grouping,
    geometry, expression, template, and unit-variant operations.
13. Inspect the normalized operations, summary, warnings, default streams, merges, auto-created recycle effects, propagation effects, and solve invalidation. Apply only that preview with `apply_flowsheet_changes`.
14. Expect open Ahuora views for the same flowsheet to refresh canvas objects and property data after a committed apply or undo.
15. Read the snapshot again. Use `query_flowsheet_snapshot` for bounded follow-up checks on large flowsheets. Verify the new revision, persisted names, allocated ports and streams, properties, compounds, replacements, and recycle settings before continuing.
16. After topology additions or connections settle, call `auto_layout_flowsheet` with the latest revision and the group that contains the new objects. Inspect its moved-object list and resulting revision. Skip this step when the user asks to keep the manual layout.
17. Call `check_flowsheet_readiness` and `analyze_flowsheet_structure` before solve work. Resolve value blockers with small typed edits and specification defects with validated replacements. Treat an open-loop fixed-recycle diagnostic as a request to restore the tear equality and use a normal process specification.
18. Use `list_flowsheet_revisions` to find checkpoints. Before `restore_flowsheet_revision`, save the current experiment if it must remain recoverable, reread the latest revision, and restore only at that exact revision.
19. On a revision conflict, discard the stale plan, reread, and re-plan. Never retry an old preview without a new read.

Read [the editing contract](references/editing-contract.md) before constructing a batch.
Read [the client permission workflow](../../references/client-permission-workflow.md)
before you request access.
