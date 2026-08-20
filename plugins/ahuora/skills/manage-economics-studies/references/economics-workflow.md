# Economics workflow contract

## Identity and revisions

- A flowsheet ID names the stable workspace.
- `content_revision` protects create, duplicate, preview, apply, and undo work.
- `configuration_revision` protects one study's inputs and calculation basis.
- An economics result-run ID names one durable result record. The current service can refresh presentation rows on a matching idempotent reuse, so the record is not an immutable historical snapshot.
- A request ID makes create, duplicate, and calculate retries safe. Reuse it only for the exact same request.

## Input workflow

Inspect the current study and live catalogs first. Build one small typed change batch. Preview it, check the summary, and apply only that preview. Read the new study revision before the next batch.

The typed batch supports study settings, assumptions, manual baseline values, composite schedule rules, costing enablement, cost drivers, equipment mappings, capital lines, operating lines, cost groups, and selected row removal.

## Calculation workflow

Readiness is an advisory check, not a calculation. Review all blockers and warnings for missing rates, empty cost scope, schedule faults, stale process data, and line-level issues. Resolve blockers that make the result invalid for the task. If calculation continues, report the unresolved blockers returned in the calculation response. Calculate only the exact current study revision.

## Scenario production schedules

Use `schedule_mode: scenario` only with a scenario returned by the `schedule_sources` catalog. A compatible source is an MSS scenario with time series enabled, at least one row, and successful selected results for every row index. The rows must represent chronological, equal-duration operating states rather than unordered cases or a parameter sweep.

The configured interval is the duration of each row. The production schedule length is `row count × interval`; annual operating cycles are annual operating hours divided by this duration. Check this basis before calculation because it affects time-weighted operating costs. A change to source inputs, selected solve results, time-series enablement, interval, or unit makes the economics result stale. Re-read the schedule preview and recalculate; do not treat the prior result run as current.

## Evaluation workflow

Use exact result-run IDs. Record the result currency, basis date, classification, warning set, calculation configuration revision, input fingerprint, and source study for each run. Request paged dependency and formula audit data when the decision needs provenance. Compare like with like. State when schedule scope, cost scope, units, or input basis differ.

For investment advice, separate calculated outputs from judgement. State the battery limits, CAPEX and OPEX scope, operating basis, exclusions, and uncertainty before you recommend an option.
