# Scenario S004: Span with attribute set and event, API-only

> [!NOTE]
> This document is a draft.

## Motivation

Instrumentation rarely only starts and ends a span; it also records attributes
learned during the operation and adds events. In a no-op implementation these
calls should be nearly free, but argument construction (attribute containers,
timestamps) can allocate even when nothing is recorded.

## Definition

- API-only: as in S003 - trace API package only, no SDK, no-op tracer.
- The span is started as an explicit root, as in S003.
- Workload: the measured operation is one span lifecycle containing:
  1. Start a span named `house.thermostat.set_temperature` with no
     attributes at start.
  2. Set the three string attributes below on the span, one call each.
  3. Add one event named `house.thermostat.updated` with the two attributes
     below, using the implementation's default timestamp.
  4. End the span.
- Span attributes (set after start):
  - `house.room` = `"living_room"`
  - `house.device` = `"thermostat"`
  - `house.action` = `"set_temperature"`
- Event attributes (constructed inline per call):
  - `house.previous_state` = `"idle"`
  - `house.new_state` = `"heating"`
- Tracer acquisition happens once, outside the measured region.
- Threading: single-threaded.
- Measurement: the harness uses its language's benchmarking framework to
  drive warmup and repetition, fixes no iteration count or duration, and must
  not hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as one complete span lifecycle, including the attribute and
event calls (the unit for `ns/op` and `allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is the trace API package named below
(`System.Diagnostics.DiagnosticSource`, `opentelemetry` or `opentelemetry-api`).

## Per-language interpretation

Same packages, and recorded versions, as
[S003](./S003-span-start-end-api-only.md).

- .NET - `ActivitySource` / `Activity` with no listener. Because
  `StartActivity` returns `null` without a listener, steps 2-3 are expressed
  idiomatically with null-conditional calls (`activity?.SetTag(...)`,
  `activity?.AddEvent(new ActivityEvent(...))`); the `ActivityEvent` and its
  tags are written inside the `activity?.` call, as application code would, so
  they are not constructed when `activity` is `null`. .NET therefore does less
  work than Rust and Java here. This is a real property of the API, not an
  error, but cross-language results are not a like-for-like comparison of equal
  work.
- Rust - `opentelemetry` crate only; `Span::set_attribute` and
  `Span::add_event` on the no-op span.
- Java - `opentelemetry-api` only; `Span.setAttribute` and
  `Span.addEvent(name, attributes)`.
