# Editing contract

- Every write names a flowsheet and expected content revision. Django derives the
  user-client permission and observation session from the agent JWT.
- `requestId` identifies one exact preview document at one base revision. Reusing it for different input is an error.
- Batch references exist only within one ordered preview document.
- Preview validates the complete batch without changing the flowsheet.
- Apply consumes the persisted normalized document, not caller-resubmitted operations.
- Apply is idempotent. A repeated application returns the previously applied preview.
- When an exact action needs first-party approval, use native URL elicitation when available. Otherwise show the review link returned by the tool, wait for the user, and retry the same exact call. Never approve it through a direct API or browser automation.
- Use `undo_flowsheet_change` only for the same client's latest applied operation
  and the current revision.
- A committed apply or undo prompts open Ahuora views on that flowsheet to refresh canvas objects and property data. A preview does not prompt a refresh.
- `auto_layout_flowsheet` directly commits a revision-safe move for one group through Ahuora's existing auto-layout service. Use the latest revision after topology changes, then retain the revision that the tool returns.
- Normal simulation-object creation creates streams for ports whose catalog `creates_stream` value is true. Never use `add_unit` for a catalog entry whose `is_stream` value is true. The MCP server rejects that operation. Inspect the preview result and reuse or merge generated streams when you connect topology.
- A recycle object is platform-owned topology data. Never create it with `add_unit` or wire it by hand. Close a material loop with `connect_port`; the normal stream merge detects the directed loop, keeps the correct stream, creates the recycle object, and selects its tear stream. Confirm that effect in the preview and read it back from the next snapshot before configuration.
- Named flowsheet revisions are immutable experiment checkpoints. Restore clones a selected revision into a new Current state and advances the content revision.
- Object creation and updates follow the same rules as edits made in Ahuora.

Use these operations for intent-level changes:

- `add_unit` creates a non-stream process unit and its normal port streams. It must not create a recycle object. For a new downstream unit, connect its generated inlet to the current upstream stream and let the preview merge the generated inlet stream. Keep generated outlet streams as boundary streams until another unit connection merges them.
- `insert_unit` places a catalog unit with exactly one inlet and one outlet into an already connected material stream. It splits the stream in series and does not create a branch decision node or standalone streams.
- `insert_unit_port_pair` places one named inlet/outlet side of an existing multi-sided unit into an already connected material stream. Use it once for each heat-exchanger side. It consumes that inlet's factory stream, keeps that outlet's factory stream, and does not create a decision-node branch.
- `connect_port` follows normal canvas merge semantics. Use it to close a material loop by connecting the return stream to the receiving unit's generated inlet. That merge creates the recycle object automatically. Joining a generated boundary stream to an intermediate stream creates a branch. Use `insert_unit` or `insert_unit_port_pair` for series insertion instead.
- `delete_objects` bins up to 100 current objects as one previewed change. Read
  the normalized cascade before apply. `restore_objects` restores exact binned
  object IDs. `restore_connections` restores exact port-to-stream pairs and can
  set a port back to no stream with a null stream ID.
- `duplicate_objects` copies up to 100 selected objects around the supplied
  canvas point. Read the preview's new IDs and batch references before another
  operation targets a copy.
- `disconnect_stream` removes one stream's current port links without deleting
  the stream. `merge_streams` joins two current stream IDs through normal merge
  rules. Inspect all object deletion, port reassignment, recycle, and branch
  effects in the preview.
- `add_port` adds one named port to a current object and can attach it to one
  current stream. Use catalog and snapshot port keys; do not invent them.
- `convert_stream_to_decision_node` turns one eligible branch stream into a
  decision node. `merge_decision_nodes` keeps `active_node_id` and merges
  `merged_node_id` into it. Verify the retained node and all rewired ports.
- `insert_translator` inserts Ahuora's translator on one eligible stream when a
  package boundary needs it. Read back both new stream segments and packages.
- `set_property` selects one value by object, property key, property index, and named indexes. Use it to change an allowed value, formula, display value, or tag; never use it to change `enabled`.
- `add_expression` adds the standard expression data to one current object.
  `add_property_template` applies one catalog `template_key` to one current
  object. Read the catalog or snapshot before choosing either target.
- `make_group` groups up to 100 current object IDs. `ungroup` removes one current
  group while retaining its members. Read back parent groups and graphic IDs.
- `set_graphic_geometry` changes exact graphic-object IDs. Set x and y together,
  set width and height together, and supply only the geometry fields that must
  change. Use this for manual layout; use `auto_layout_flowsheet` for a full
  group layout.
- `switch_unit_variant` changes one current unit to a catalog
  `target_object_type`. Inspect port, property, stream, and solve-invalidation
  effects before apply because variants can change the model contract.
- `configure_stream` sets a property package, compounds, compound mode, and stream specifications on one source stream. The standard compound-propagation path carries its package and compounds to connected downstream streams. Do not repeat the same package and compound change on each downstream stream.
- `set_split_fractions` sets explicit outlet fractions; their total must not exceed one.
- `configure_recycle` selects the tear stream and the state variables fixed at the tear on an existing auto-created recycle. It never creates the recycle object.
- `replace_control` exchanges one current fixed property for one structurally valid free property. Use the fixed property as `manipulated_id`, the free property as `set_point_id`, and keep `structural_validation_bypassed` false.
- Use `set_property_value` only when the exact property-value ID came from the current snapshot, and never use it to change `enabled`.

## Structural specifications

- Manual `enabled` or disabled changes are not permitted. Never include `enabled` in a `set_property`, `set_property_value`, or `configure_stream` change, even when the tool schema accepts the field.
- If the desired property is already a fixed structural input, change only its value.
- If the desired property is free, call `analyze_flowsheet_structure` at the current revision. Select the current fixed property that the user wants to replace and a free property listed as its valid match.
- Preview `replace_control` before setting the new input value. Do not bypass structural validation. If validation rejects the exchange, choose another reported match or correct the model structure; do not toggle enablement.
- Apply the replacement, read the new revision, and verify the relationship endpoints. Then set the replacement-owned input value in a new preview.
- Re-run readiness and structural analysis. Require equal free-variable, constraint, and matching counts before solve work.

Never invent unit types, port keys, property keys, index keys, package names, or compound names. Read them from the catalog or current snapshot.

Property-package limits are not all hard-coded in the unit catalog. Read `ahuora://guidance/property-packages/v1`. In particular, use Helmholtz for one supported pure component and use this platform's Peng-Robinson adapter for mixtures with at least two compounds, not one-component PR.
