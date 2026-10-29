---
layout: post
title: "Why your swift-log metadata provider might be silently ignored"
description: "A LogHandler has to opt in to MetadataProvider. Here's how I found that OpenTelemetry's Swift bridge didn't, and the fix."
published: false
---

<!-- DRAFT: read and rewrite this in your own words, then change `published: false` to `true`. -->

swift-log lets you attach a `Logger.MetadataProvider` to a logger. It's a closure that adds
metadata, like a request ID or a trace ID, to every log line without passing it around by hand.

```swift
let logger = Logger(label: "api", metadataProvider: .init { ["request-id": "\(requestID)"] })
logger.info("Order placed")
```

There's a catch. The provider only works if the `LogHandler` behind the logger supports it.
`LogHandler` gives `metadataProvider` a default implementation that returns `nil` and ignores
anything you assign. If a handler never declares the property, your provider is dropped.
In debug builds you get a one-time warning. In release builds, nothing.

## Where I hit it

OpenTelemetry Swift ships `OTelLogHandler`, which turns swift-log calls into OpenTelemetry log
records. It didn't declare `metadataProvider`, so provider metadata never reached the records
([issue #1235](https://github.com/open-telemetry/opentelemetry-swift/issues/1235)).

## The fix

Two parts. Declare the property, then merge the provider's metadata when logging:

```swift
public var metadataProvider: Logger.MetadataProvider?

public func log(event: LogEvent) {
  if let provided = metadataProvider?.get() {
    attributes.merge(convertMetadata(provided)) { _, new in new }
  }
  // then per-call metadata, then the handler's own metadata
}
```

Order matters. The provider goes first so it has the lowest priority. Metadata you pass at the
call site should always win, which matches swift-log's own `StreamLogHandler`.

## What to check in your own code

If you've written a custom `LogHandler`, search it for `metadataProvider`. If it isn't there,
your providers aren't doing anything.
