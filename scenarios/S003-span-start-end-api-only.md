# Scenario S003: Span start/end, API-only

> [!NOTE]
> This document is a draft.

## Motivation

Libraries and frameworks create spans on hot paths (every request, query or
message). When no SDK is configured the [Trace
API](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md)
must be a no-op, and library owners need to know what that costs. This
scenario measures the cost of starting and ending one span.

## Definition

- API-only: depends only on the OTel trace API package; no SDK package is
  referenced or loaded. The tracer resolves to the API's built-in no-op
  implementation.
- Workload: the measured operation is one span lifecycle - start a span named
  `house.thermostat.set_temperature` with span kind `Internal` (the default)
  and end it. No work happens between start and end.
- Tracer acquisition happens once, outside the measured region.
- Attributes: the same three string attributes are supplied inline at span
  start (for example via start options or a builder) on every call, with no
  hoisting or caching of an attribute container (no static or pre-built tag
  collection or attribute set reused across calls; it is constructed per call
  as application code with per-call values would):
  - `house.room` = `"living_room"`
  - `house.device` = `"thermostat"`
  - `house.action` = `"set_temperature"`
- No parent: the span is started as an explicit root, so a leaked ambient
  context from earlier work cannot change the result (for example
  `setNoParent()` in Java, an empty `Context` in Rust, a default
  `ActivityContext` parent in .NET). The harness asserts, outside the measured
  region, that no ambient span is current (for example
  `Activity.Current == null` in .NET).
- Threading: single-threaded. Multi-threaded variants are out of scope.
- Measurement: the harness uses its language's benchmarking framework to drive
  warmup and repetition, fixes no iteration count or duration, and must not
  hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as one span start plus end (the unit for `ns/op` and
`allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is the trace API package named below
(`System.Diagnostics.DiagnosticSource`, `opentelemetry` or `opentelemetry-api`).

## Per-language interpretation

- .NET - `System.Diagnostics.ActivitySource` from
  `System.Diagnostics.DiagnosticSource`, with no `ActivityListener` or SDK
  registered. `StartActivity` returns `null` in this state; the measured
  operation is `StartActivity(...)` followed by disposal/`Stop` when the result
  is non-null, as idiomatic `using` or null-conditional code does. The call
  is `StartActivity(name, ActivityKind.Internal, default(ActivityContext), tags)`
  (the explicit-parent overload, so the root rule holds), with the attributes
  passed as `tags`.
- Rust - `opentelemetry` crate only; `global::tracer(...)` returns the no-op
  tracer. Measure
  `span_builder(...).with_attributes(...).start_with_context(&tracer, &Context::new())`
  followed by `end()` (or drop). Plain `.start(&tracer)` parents to the current
  context and is not used.
- Java - `opentelemetry-api` only;
  `spanBuilder(...).setNoParent().setAttribute(...).startSpan()` followed by
  `end()`.
