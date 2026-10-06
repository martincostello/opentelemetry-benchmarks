# Benchmark Scenarios

This directory holds the language-agnostic scenario definitions that
language harnesses implement. Each scenario is the single source of truth for
what is measured and how results are reported, so that any language
implementation can produce a conforming harness from the document alone.

## Layout

- One Markdown file per scenario, named `S<nnn>-<short-slug>.md`
  (e.g. [`S001-counter-increment-api-only.md`](./S001-counter-increment-api-only.md)).
- Scenario IDs (`S001`, `S002`, ...) are stable and never reused.
- The matching implementations live under
  [`/harnesses/<language>/`](../harnesses/), one harness per language.

## Each scenario document should define

- Workload - exactly what operation is measured and how (instrument names,
  attribute keys/values, loop shape, threading model).
- Reported metrics - the values each harness must emit (e.g. `ns/op`,
  `allocations/op`) and their "smaller/larger is better" direction.
- Per-data-point metadata - the environment fields recorded with each
  result.
- Per-language interpretation - how the abstract scenario maps to each
  language's API/SDK packages.

## Scenarios

| ID | Scenario | Status |
|----|----------|--------|
| [S001](./S001-counter-increment-api-only.md) | Counter increment, API-only (no SDK configured) | Active |
| [S002](./S002-histogram-record-api-only.md) | Histogram record, API-only | Draft |
| [S003](./S003-span-start-end-api-only.md) | Span start/end, API-only | Draft |
| [S004](./S004-span-attribute-event-api-only.md) | Span with attribute set and event, API-only | Draft |
| [S005](./S005-nested-spans-api-only.md) | Nested spans (depth 3), API-only | Draft |
| [S006](./S006-log-record-emit-api-only.md) | Log record emit, API-only | Draft |
| [S007](./S007-log-enabled-check-api-only.md) | Log record with Enabled check, API-only | Draft |

Additional scenarios (SDK fast-paths, multi-threaded workloads)
are tracked as future work in
[OTEP 5109](https://github.com/open-telemetry/opentelemetry-specification/pull/5118).
