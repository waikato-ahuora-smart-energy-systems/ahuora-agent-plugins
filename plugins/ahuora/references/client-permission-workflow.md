# Client Permission Workflow

Read this workflow before you request a client permission change.

1. Call `get_client_permissions` before you request a change. Use the active
   scope, `resource_permissions`, and policy as the source of truth. When the
   user gives names instead of IDs, use `list_projects`, `get_project`,
   `list_project_folders`, or `get_flowsheet_identity` to resolve the visible
   resource first. An `in_client_scope: false` identity can be named in a
   permission request, but it cannot be used for flowsheet content until the
   user approves that request. `list_projects` defaults to all projects visible
   to the user; use its `owned` or `shared` visibility filter when the task needs
   one exact set. If workspace discovery is denied, request
   `read_workspace: yes` before resolving names.
2. Request only the resources and capabilities that the task needs:
   - For `workspace`, send no project or flowsheet IDs.
   - For `project`, send one or more IDs in `projectIds` and no flowsheet IDs.
   - For `flowsheets`, send one or more IDs in `flowsheetIds` and no project
     IDs. The flowsheets can belong to different projects.
3. Use a fresh UUID in `requestId`. In `requestedPolicy`, send only the
   capability decisions that the task needs to change. Ahuora keeps the user's
   saved defaults for omitted decisions and applies the stated decisions to each
   selected resource row. Do not create, group, or send resource permission
   records. The user can change each record during review.
4. A request grants no access. Let native URL elicitation pause for first-party
   Ahuora review. Otherwise show the returned review link and wait. Never use a
   direct call, browser automation, a pasted token, or another client to bypass
   review.
5. Do not request a sandbox as scope or as a permission. A sandbox is a
   first-party review choice that Ahuora can show only when the request names
   exactly one flowsheet.
6. For link-based review, poll `get_client_permission_change_request` with the
   exact request ID. Continue only when native resumption or that read returns
   `status: active`.
7. Read `get_client_permissions` again after approval. Confirm that the active
   scope contains every target and that `resource_permissions` has one
   single-resource record for each selected project or flowsheet. Call
   `get_capabilities` for every target flowsheet and require the decisions that
   the next operation needs.
