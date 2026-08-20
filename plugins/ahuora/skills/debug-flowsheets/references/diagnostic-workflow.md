# Diagnostic workflow

Check in this order:

1. Access and current revision.
2. Object and port topology from the snapshot.
3. Required specifications, controls, package choice, compound propagation, and scenario configuration.
4. Full structural analysis: degrees of freedom, matching, unsupported properties, and DM partitions.
5. Dependency trace, inclusion reasons, boundaries, and intentional recycle cuts.
6. SolveRun plan provenance and included units.
7. Separate solve, result, and projection statuses.
8. Run-attributed diagnostics, failure summary, unit initialization status, and bounded logs.
9. Immutable result values and recycle-cut mismatch report when the run is eligible.

Do not infer a mathematical result from the execution task status. Do not overwrite evidence with a new solve before recording the failing SolveRun ID and its diagnostics.

Use revisions as lab checkpoints: snapshot, make one change, solve, compare, then keep the new state or restore the checkpoint. A recycle cut is a logical solve boundary, not a deleted stream.
