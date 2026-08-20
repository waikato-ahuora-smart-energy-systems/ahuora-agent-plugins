# Comparison contract

- Use immutable SolveRun-owned solutions, never the flowsheet's current mutable property values.
- Join values by semantic meaning and captured owner identity. Supply each run's
  own captured source property ID; IDs are not stable across flowsheets.
- Confirm units before calculating deltas.
- Distinguish absent, unavailable, and zero values.
- State when full and subset plans differ, because their output domains may not be comparable.
- For aggregate runs, compare corresponding child rows or aggregate summaries consistently.
- `compare_solve_runs` is bounded to two through five target pairs and 100
  properties. Each property must name one property ID for every target run. The
  first target supplies the baseline for numeric deltas.
- Django authorizes each target flowsheet and performs the bounded bulk query.
  Do not loop over `get_solve_run_results` to build a comparison.
- A null delta means the value is absent or its unit or vector shape is incompatible; it does not mean zero change.
