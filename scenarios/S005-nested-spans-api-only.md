# Scenario S005: Nested spans (depth 3), API-only

> [!NOTE]
> This document is a draft.

## Motivation

Real requests produce chains of spans (e.g. handler, service, data access).
Starting a child requires looking up and propagating the current context, and
implementations differ in how much that costs when no SDK is present - for
example an implicit context lookup versus a null result with no listener.
This scenario measures a three-level span chain.

## Definition

- API-only: as in S003 - trace API package only, no SDK, no-op tracer.
- Workload: the measured operation starts and ends three nested spans:
  `house.request` (outer), `house.thermostat.set_temperature` (middle) and
  `house.device.write` (inner). The middle span is started while the outer is
  the active span, and the inner while the middle is active; they end in
  reverse order (inner, middle, outer).
- Each span carries the same three string attributes inline at start, as in S003
  (deliberately identical on all three spans, so per-span cost is directly
  comparable with S003):
  - `house.room` = `"living_room"`
  - `house.device` = `"thermostat"`
  - `house.action` = `"set_temperature"`
- Only the outer span is started as an explicit root, as in S003, including
  the same assertion that no ambient span is current before measuring. The
  middle and inner spans are not roots: they take their parent from the
  ambient context, which holds the span started just before them.
- Parent relationship: each child is started while its parent is the current
  span in the ambient context (for example `makeCurrent()` in Java, `attach`
  in Rust, `Activity.Current` in .NET), not via an explicitly passed parent.
  Making a span current and restoring the previous context on end is part of
  the measured region. .NET has no ambient context without a listener (see
  below).
- Depth is fixed at 3. Deeper chains are out of scope.
- Threading: single-threaded.
- Measurement: the harness uses its language's benchmarking framework to drive
  warmup and repetition, fixes no iteration count or duration, and must not
  hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as one complete chain of three nested spans (the unit for
`ns/op` and `allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is the trace API package named below
(`System.Diagnostics.DiagnosticSource`, `opentelemetry` or `opentelemetry-api`).

## Per-language interpretation

Same packages, and recorded versions, as
[S003](./S003-span-start-end-api-only.md).

- .NET - `ActivitySource.StartActivity` with no listener; nesting follows
  `Activity.Current`. With no listener every call returns `null` and
  `Activity.Current` is never set, so no context is made current or restored:
  the .NET result measures three null-returning calls, not context
  propagation. The context propagation part of this scenario is therefore not applicable
  in .NET. This is the real no-SDK behaviour; harnesses must not register a listener to force nesting. Cross-language
  results are therefore not a like-for-like comparison.
- Rust - `opentelemetry` crate only; start each span from
  `Context::current()` (the outer from an empty `Context`), wrap it with
  `Context::current_with_span`, and `attach` it. The `ContextGuard`s are
  created and dropped inside the measured region, dropped in reverse order
  (inner, middle, outer). `tracer.in_span` is not used.
- Java - `opentelemetry-api` only; `span.makeCurrent()` scopes closed in
  reverse order, children created via `spanBuilder(...)` against the current
  context.
