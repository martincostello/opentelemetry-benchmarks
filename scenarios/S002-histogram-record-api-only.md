# Scenario S002: Histogram record, API-only

> [!NOTE]
> This document is a draft.

## Motivation

The Histogram is the most commonly used instrument after the Counter, and its
record path differs from the Counter's add path in most implementations
(different no-op instrument type, value type handling, and in some languages
different generic instantiations). S002 measures what a single histogram
record costs when no SDK is configured.

## Definition

- API-only: depends only on the OTel metrics API package; no SDK package is
  referenced or loaded. The instrument resolves to the API's built-in no-op
  implementation.
- Workload: the measured operation is a single record of the value `1.5`
  (a double-precision floating-point value) read from a non-constant source
  (for example a field or benchmark parameter the compiler/JIT cannot
  constant-fold) on a `Histogram` instrument named
  `house.action.duration` with unit `s`.
- Instrument creation happens once, outside the measured region.
- Attributes: each call constructs and passes the same three string
  attributes inline, exactly as in S001:
  - `house.room` = `"living_room"`
  - `house.device` = `"thermostat"`
  - `house.action` = `"set_temperature"`
- Threading: single-threaded. Multi-threaded variants are out of scope.
- Measurement: the harness uses its language's benchmarking framework to drive
  warmup and repetition, fixes no iteration count or duration, and must not
  hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as a single histogram record (the unit for `ns/op` and
`allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is that of the metrics API package named below.

## Per-language interpretation

- .NET - `System.Diagnostics.Metrics.Histogram<double>` from
  `System.Diagnostics.DiagnosticSource`, with no listener or SDK registered.
- Rust - `opentelemetry` crate only;
  `global::meter(...).f64_histogram(...).build()`.
- Java - `opentelemetry-api` only; a `DoubleHistogram` built from
  `GlobalOpenTelemetry.getMeter(...).histogramBuilder(...)` (the default,
  double-valued builder), not a `LongHistogram`.

The instrument must be a double-valued histogram in every language, so that
the recorded `1.5` is not truncated or converted.

Languages not listed here follow the same rule: depend only on that language's
OTel metrics API package, with no SDK registered, and record that package's
version against each data point. The recorded versions are the same as S001
(`System.Diagnostics.DiagnosticSource`, `opentelemetry`, `opentelemetry-api`).
