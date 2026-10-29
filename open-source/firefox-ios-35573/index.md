---
layout: project
title: "Firefox for iOS: white flash when swiping back in dark mode"
kind: Open source · mozilla-mobile/firefox-ios
summary: With Website Dark Mode on, swiping back could flash a bright white screen. I traced the cause into WebKit's own source and wrote a fix in Firefox's web view.
status: Proposed fix, verifying on device
stack: [Swift, UIKit, WebKit, XCTest]
links:
  - label: Issue #35573
    url: https://github.com/mozilla-mobile/firefox-ios/issues/35573
description: Root cause analysis and proposed fix for a dark mode white flash in Firefox for iOS swipe-back navigation.
---

## The issue

Firefox for iOS has a **Website Dark Mode** that darkens any web page. With it on, swiping back from the left
edge sometimes showed a bright white screen for a moment. That's jarring at night and defeats the point of
dark mode. It only happened sometimes, and the back button never caused it.

## Root cause

The "sometimes" was the key clue. I followed the swipe gesture into WebKit's source code
(`ViewGestureControllerIOS.mm`). When a back swipe starts, WebKit shows a placeholder view of the previous
page while the real page loads:

- The placeholder (layer name `SwipeSnapshot`) starts with a **white** background.
- It only uses a different color if WebKit still has a saved snapshot of that page.
- When the snapshot has been evicted from memory, the placeholder stays white.

Firefox's dark mode works by injecting a script into the page, so WebKit has no idea the page is dark.
Firefox already sets the web view's under-page color to black in dark mode, but WebKit's swipe
placeholder is a separate view that ignores it.

## Proposed fix

In `TabWebView`, Firefox's `WKWebView` subclass, I watch for WebKit adding its swipe transition container.
When dark mode is on, I give any empty placeholder a black background before it's visible:

```swift
override func didAddSubview(_ subview: UIView) {
  super.didAddSubview(subview)
  guard isNightModeEnabled,
        subview.layer.name == SwipeTransitionLayerName.container else { return }
  DispatchQueue.main.async { [weak subview] in
    guard let subview else { return }
    TabWebView.darkenEmptySwipeSnapshots(in: subview)
  }
}

static func darkenEmptySwipeSnapshots(in view: UIView, color: UIColor = .black) {
  if view.layer.name == SwipeTransitionLayerName.snapshot, view.layer.contents == nil {
    view.backgroundColor = color
  }
  view.subviews.forEach { darkenEmptySwipeSnapshots(in: $0, color: color) }
}
```

Placeholders that do have a real snapshot image are left alone, so normal swipes look exactly as before.

## Tests

The darkening logic is a static function, so it can be tested without driving a real swipe gesture:

| Test | What it proves |
|---|---|
| `testDarkenEmptySwipeSnapshots_setsColorOnEmptySnapshot` | An empty placeholder nested inside the container turns black. |
| `testDarkenEmptySwipeSnapshots_keepsSnapshotWithImage` | A placeholder that has a real snapshot is not touched. |
| `testDarkenEmptySwipeSnapshots_ignoresOtherViews` | Unrelated views are never changed. |

## Trade-offs and decisions

- **Relies on WebKit's internal layer names.** `SwipeTransitionContainer` and `SwipeSnapshot` aren't public
  API and could change in a future iOS. I chose this because it **fails safe**: if Apple renames them, the
  code simply does nothing and the old behavior returns. Nothing crashes. I'll flag this openly in the PR so
  maintainers can weigh it.
- **Alternatives I ruled out:**
  - *Changing `underPageBackgroundColor`*: Firefox already does this, and the placeholder ignores it.
  - *Turning off swipe-back in dark mode*: removes a core navigation gesture to hide a visual glitch.
  - *Keeping Firefox's own page snapshots*: costs memory on every page to fix a rare case.
- **Scoped to dark mode.** The hook returns immediately when dark mode is off, so light-mode users see no
  change and pay no cost.
- **Testable by design.** Keeping the logic in a small static function let me unit test it without
  simulating gestures.

## Status

The issue is open to contributors. Next, I'll confirm the view hierarchy on a device with Xcode's
view debugger, record before and after videos, and open the pull request.
