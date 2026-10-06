# Scenario S006: Log record emit, API-only

> [!NOTE]
> This document is a draft.

## Motivation

The [Logs
API](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/logs/api.md)
is also required to be a no-op without an SDK. Log calls are among the most
frequent telemetry calls in an application, so the cost of emitting a record
that goes nowhere matters. This scenario measures a single emit.

## Definition

- API-only: depends only on the OTel logs API package, or the language's
  standard logging abstraction where that is the API (see below); no
  OpenTelemetry SDK is referenced or loaded. Logger acquisition resolves to a
  no-op.
- Workload: the measured operation is a single emit of one log record with:
  - severity `INFO` (severity number 9)
  - body `"Thermostat setpoint changed"`
  - the same three string attributes, constructed inline per call:
    - `house.room` = `"living_room"`
    - `house.device` = `"thermostat"`
    - `house.action` = `"set_temperature"`
- Logger acquisition happens once, outside the measured region.
- No trace context is active.
- Threading: single-threaded.
- Measurement: the harness uses its language's benchmarking framework to drive
  warmup and repetition, fixes no iteration count or duration, and must not
  hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as one log record emit (the unit for `ns/op` and
`allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is that of the logging API package named below.

## Per-language interpretation

- .NET - There is no separate OTel logs API package; logging is
  `Microsoft.Extensions.Logging`. "API-only" means depending on
  `Microsoft.Extensions.Logging` (which includes `LoggerFactory`, and
  transitively `.Abstractions`) with no OpenTelemetry logging provider
  registered. The logger is an `ILogger` created from an `ILoggerFactory` with
  no providers, built with `LoggerFactory.Create(b => { })` (not `NullLogger`,
  which short-circuits and is not representative of application code). Emit via
  `logger.Log(LogLevel.Information, eventId, state, null, formatter)` where
  `state` is a list of the three key/value attributes constructed inline per
  call (not the `LogInformation(msg, args)` extensions, which allocate
  differently). With no providers, `Log` has nothing to write and may not invoke the
  formatter, so the measured cost is expected to be dominated by per-call state
  construction rather than formatting; this depends on the library version, and
  the harness records the version for that reason. The recorded version
  is the `Microsoft.Extensions.Logging` package version. Unlike Rust and Java,
  this measures an empty provider list in `Microsoft.Extensions.Logging`, not a
  no-op OpenTelemetry logger, so cross-language results are not a like-for-like
  comparison.
- Rust - `opentelemetry` crate only (with its `logs` feature); the logger comes
  from the API's no-op provider (`opentelemetry::logs::NoopLoggerProvider`)
  and is obtained once, outside the measured region. Build and emit a
  `LogRecord` via its `Logger`. The recorded version is the `opentelemetry`
  crate version.
- Java - `opentelemetry-api` only; obtain the named logger once, outside the
  measured region, with `GlobalOpenTelemetry.get().getLogsBridge().get("house")`,
  then `logger.logRecordBuilder()...emit()`. The recorded version is the
  `opentelemetry-api` artifact version.
