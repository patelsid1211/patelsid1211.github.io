---
layout: project
title: "OpenTelemetry Swift: metadata providers were silently ignored"
kind: Open source · open-telemetry/opentelemetry-swift
summary: Swift apps that attached request IDs to logs through a swift-log MetadataProvider lost that context on the way to OpenTelemetry. I traced why, fixed it, and covered it with tests.
status: "PR #1262, in review"
stack: [Swift, swift-log, OpenTelemetry, XCTest]
links:
  - label: Pull request #1262
    url: https://github.com/open-telemetry/opentelemetry-swift/pull/1262
  - label: Issue #1235
    url: https://github.com/open-telemetry/opentelemetry-swift/issues/1235
description: How I fixed OpenTelemetry Swift's log bridge dropping swift-log MetadataProvider context, with tests and trade-offs.
---

## The project

OpenTelemetry is the CNCF's open standard for observability: traces, metrics and logs from running apps,
sent to tools like Datadog or Grafana. OpenTelemetry Swift is its SDK for iOS, macOS and server-side Swift.
It includes `OTelLogHandler`, a bridge that turns Apple's `swift-log` calls into OpenTelemetry log records.

## The issue

`swift-log` lets an app attach a `Logger.MetadataProvider`, a closure that adds context such as a request ID
or session ID to every log line automatically. With `OTelLogHandler`, that context never arrived.

```swift
var handler = OTelLogHandler(loggerProvider: provider)
handler.metadataProvider = Logger.MetadataProvider { ["k": "v"] }
// Expected: every log record has attribute k = v
// Actual: k is missing
```

## Root cause

In `swift-log`, every `LogHandler` gets a default `metadataProvider` from a protocol extension. That default
returns `nil` and throws away anything assigned to it, with only a one-time warning in debug builds.
`OTelLogHandler` never declared its own property, so it fell back to the default. Assignments compiled
and ran without errors, and the data quietly disappeared. In release builds there was no signal at all.

## The fix

Two small changes in `LogHandler.swift`:

```swift
public var metadataProvider: Logging.Logger.MetadataProvider?

public func log(event: Logging.LogEvent) {
  // ...source, file, function and line attributes...

  if let providedMetadata = metadataProvider?.get() {
    otelattributes.merge(convertMetadata(providedMetadata)) { _, new in new }
  }

  // then per-call metadata, then the handler's own metadata
}
```

I also added a README example showing the real use case: a `@TaskLocal` request ID that gets attached to
every log inside a request.

## Tests

Each test runs through the real `Logger` API and a real `LoggerProviderSdk` with an in-memory record
processor, so the tests check what an app would actually export, not a mock.

| Test | What it proves |
|---|---|
| `testLogHandlerAppliesMetadataProviderAsAttributes` | Provider metadata shows up as attributes on the emitted record. This is the bug from the issue. |
| `testLoggerExposesHandlerMetadataProvider` | `Logger.metadataProvider` returns the handler's provider instead of `nil`, so swift-log's own APIs see it. |
| `testLogHandlerMethodMetadataOverridesMetadataProvider` | When the provider and the log call set the same key, the log call wins. |

To confirm the tests guard the fix, I restored the old `LogHandler.swift` while keeping the new tests.
Two of them failed, which reproduces the bug. With the fix back, all 13 tests in the suite passed,
and the full `swift test` run passed with 0 failures.

## Trade-offs and decisions

- **Lowest precedence for provider metadata.** Provider values are merged first, so anything more specific
  overrides them. This matches swift-log's built-in `StreamLogHandler`, so apps get the same behavior
  whichever handler they use.
- **Kept scope tight.** A separate issue (#1233) questions whether handler metadata should override per-call
  metadata. Changing that would alter behavior for existing users, so I left it to that issue and said so in
  the PR. A small, focused change is easier to review and safer to merge.
- **Cost per log call.** The provider closure runs on every log, which is how swift-log designs providers.
  When no provider is set, the property is `nil` and the cost is a single nil check.
- **Real objects over mocks in tests.** It takes a little more setup, but the tests catch integration bugs
  that a mocked handler would hide.

## What I learned

My first version of one test didn't compile. I had assumed `Logger` had an initializer taking both a
metadata provider and a custom handler. Reading swift-log's source showed it doesn't, so I rewrote the test
to check the same behavior through `logger.metadataProvider`. It was a good reminder to verify an API
against its source rather than memory.
