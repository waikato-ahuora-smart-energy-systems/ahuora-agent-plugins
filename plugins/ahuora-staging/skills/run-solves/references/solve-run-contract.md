# SolveRun contract

- `SolveRun` is the durable Ahuora identity for inputs, plan, progress, outcome, diagnostics, and immutable results.
- A Solve Run is the stable identity for one solve and its results.
- MCP Tasks may later wrap a call but do not replace the SolveRun record.
- Keep `solve_status`, `result_status`, and `projection_status` distinct.
- A repeated `start_solve` with the same request ID and identical revision-bound payload recovers the same response without redispatch.
- Aggregate multi-steady-state runs own ordered child SolveRuns; inspect children rather than internal child tasks.
- Token/session expiry does not cancel a SolveRun. Cancellation is an explicit
  operation and remains limited to work owned by the same client permission.
- A full solve uses `start_solve` without subset fields. A subset solve uses a previewed selection and exact execution-plan signature only when fewer than all active units are intended.
- Full diagnostics are opt-in. Omit `performDiagnostics` or set it to `false` for normal and iterative solves. When enabled, the IDAES structural, infeasibility, and numerical checks share the solve task's time limit and can slow large or poorly initialized models significantly.
- Unit initialization status and run diagnostics belong to the captured SolveRun. Recycle-cut reports derive closure comparisons from that run's immutable evidence.
- Stored `progress` exposes process stages and detailed solver iterations. Use its ordered sequence, phase, unit status, elapsed time, objective, primal infeasibility, and convergence health to update users during long solves.
- `restoration_mode: true` or convergence health `restoring` means IPOPT entered its restoration phase to recover feasibility. This is usually an adverse sign, though it does not prove final failure. Report it and track whether infeasibility improves.
- Detailed convergence streaming is an opt-in request for a running multi-steady-state child. It adds frames to the same durable SolveRun progress; it does not create another solve identity.
