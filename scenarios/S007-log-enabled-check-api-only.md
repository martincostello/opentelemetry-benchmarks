# Scenario S007: Log record with Enabled check, API-only

> [!NOTE]
> This document is a draft.

## Motivation

The Logs API defines an `Enabled` operation so callers can skip building a
record that nothing will consume. Callers guard emits with it precisely to
avoid argument construction cost, and the most common case is verbose/debug
logging: diagnostic records that are frequent, expensive to build, and usually
disabled in production. Both the guard's own cost and the saving it provides
are what library owners need to see. This scenario measures both, using
severity `DEBUG`.

## Definition

- API-only: as in S006, except that `Enabled` is exposed only through unstable
  or incubating APIs in some languages (see below). The extra artifact or
  feature is allowed, and its version is recorded as well.
- Workload: two variants, each reported as its own benchmark:
  - **S007a - Enabled check only:** call `Enabled` on the logger with severity
    `DEBUG`. The arguments are fixed per language: severity only in .NET and
    Java, and severity plus a fixed target and no name in Rust. The calls are
    therefore not identical across languages, which is a property of each API,
    but each is the minimal call the language allows. The result is consumed so it cannot be
    eliminated.
  - **S007b - Guarded emit:** call `Enabled` as above and, if it returns
    `true`, emit the record as in S006 (same body and three inline attributes),
    except with severity `DEBUG` (severity number 5) instead of `INFO`. Because
    the API is a no-op, `Enabled` is expected to return `false` and the emit
    branch is not taken; attribute and record construction must sit **inside**
    the guarded branch.
  - Both variants assert, outside the measured region, that `Enabled` returns
    `false` for the logger under test. A harness whose check returned `true`
    would otherwise benchmark a full emit under the S007b name.
- Logger acquisition happens once, outside the measured region.
- Threading: single-threaded.
- Measurement: the harness uses its language's benchmarking framework to drive
  warmup and repetition, fixes no iteration count or duration, and must not
  hand-roll a counting loop that the compiler/JIT could hoist or eliminate.

## Reported metrics

See [S001](./S001-counter-increment-api-only.md#reported-metrics), with one
operation defined as one invocation of the variant being measured (the `Enabled`
check alone for S007a, the check plus guarded emit for S007b); each variant is
reported as its own result (the unit for `ns/op` and `allocations/op`).

## Per-data-point metadata

See [S001](./S001-counter-increment-api-only.md#per-data-point-metadata). The
recorded version is that of the logging API package named for S006. Where
`Enabled` needs an extra package (Java: `opentelemetry-api-incubator`), the
package name and version are added to the data point's `extra` field as
`extra_package=<name> <version>`. Rust has no extra package (the unstable
`spec_unstable_logs_enabled` feature is part of the `opentelemetry` crate), so
the feature name is recorded instead as `extra_feature=<name>`.

## Per-language interpretation

Same packages as S006, plus the unstable or incubating pieces noted below.

- .NET - `ILogger.IsEnabled(LogLevel.Debug)` on a logger from a factory
  built with `LoggerFactory.Create(b => { })` (no providers), which must
  return `false` (asserted, as above); S007b guards the S006 emit with it.
- Rust - `Logger::event_enabled(Severity::Debug, "house.thermostat", None)` on
  the no-op logger (fixed target, no name), guarding record construction and
  `emit`. This API may be gated behind an unstable crate feature
  (`spec_unstable_logs_enabled`), which the harness enables and documents.
- Java - `Logger.isEnabled(Severity.DEBUG)`, which may only be available on the
  incubating extended logger API; the harness documents the artifact used (for
  example `opentelemetry-api-incubator`) and records its version in addition to
  `opentelemetry-api`. The guarded emit is the S006 emit.

Where a language's stable API lacks an `Enabled` operation, the incubating or
unstable one is used if it exists (as above). If neither exists, or the no-op
`Enabled` returns `true` (failing the assertion), the scenario is marked not
applicable for that language rather than approximated.
