---
layout: post
title: "SwiftUI view identity, and why your @State keeps resetting"
description: "SwiftUI stores @State with a view's identity, so an if/else in body can wipe it. This post explains how identity works and how to keep it stable."
published: true
---

SwiftUI does not track your views by name. It tracks them by identity, and most of the time that identity comes from where a view sits in your `body`.

Once that idea clicks, a whole family of strange bugs starts to make sense. A counter goes back to zero, a text field loses what the user typed, or a scroll view jumps to the top, all because a Boolean flipped somewhere above it.

In this post I want to walk through how identity works, why it controls the lifetime of `@State`, and the small habits I use to keep it stable.

## The hotel analogy

Think of a hotel. The front desk keeps your luggage in room 204. If you walk into room 310, the room is empty, even though you are the same guest with the same name.

SwiftUI treats `@State` like that luggage. It does not attach state to the struct you wrote, because that struct is a short lived value that SwiftUI recreates all the time. It attaches state to the view's identity, which is the room number.

As long as the view keeps the same identity, its state survives every update. When the identity changes, SwiftUI treats it as a brand new view, creates fresh state for it, and throws the old state away.

## Two kinds of identity

Apple describes two ways a view gets its identity.

**Structural identity** comes from the view's type and its position in the view hierarchy. You get it for free whenever you write views in `body` without any explicit IDs. SwiftUI knows that "the second child of this `VStack`" is the same view from one update to the next.

**Explicit identity** comes from an identifier you provide. `ForEach` uses the `id` of each element, and the `.id(_:)` modifier lets you set one on any view.

Both kinds follow the same rule. Same identity means same view and kept state. New identity means new view and fresh state.

## How an if/else changes identity

This is where most surprises come from. When you write an `if/else` inside a view builder, SwiftUI does not see one view whose style changes. It sees two separate branches, and each branch has its own structural identity.

When the condition flips, SwiftUI removes the view in one branch and inserts the view in the other. Any `@State` inside the old branch is gone, and the new branch starts from its initial values. You also get an insertion and removal transition instead of a smooth change of a property, which is often the first visual hint that identity changed.

This small example shows it. `CounterView` owns a piece of state, and `PlanCard` decides how it looks.

```swift
import SwiftUI

struct CounterView: View {
    @State private var count = 0
    var body: some View { Button("Taps: \(count)") { count += 1 } }
}

struct PlanCard: View {
    let isPro: Bool
    var body: some View {
        // Before: two branches, two identities. Flipping isPro resets count to 0.
        // if isPro { CounterView().tint(.orange) } else { CounterView().tint(.gray) }

        // After: one view, one identity. Only the tint changes, so count survives.
        CounterView()
            .tint(isPro ? .orange : .gray)
    }
}
```

With the commented out version, tapping the button five times and then toggling `isPro` sets the counter back to zero. With the second version, there is only ever one `CounterView` in that position, so SwiftUI keeps its state and simply updates the tint.

## Common mistakes

**Branching on style.** If the only difference between two branches is a color, a font or an opacity, move the condition into the modifier's value. Keep `if/else` for cases where the content really is different, such as a loading screen versus a list.

**Conditional modifier helpers.** A popular extension adds an `.if(condition) { $0.someModifier() }` helper to `View`. It reads nicely, but it hides an `if/else` inside, so it changes identity every time the condition flips. I avoid it for anything that holds state.

**A fresh ID on every update.** Passing a new value to `.id(_:)` on each render gives the view a new identity each time. SwiftUI then rebuilds it from scratch, which resets state and costs performance.

```swift
// Resets state on every update, because the identity is new each time
DetailView(item: item)
    .id(UUID())

// Resets state only when the selected item changes, which is usually the goal
DetailView(item: item)
    .id(item.id)
```

The second form is a useful tool. When you do want a clean slate, such as resetting a form when the user picks a different record, a stable ID tied to that record is the clearest way to say so.

**Unstable IDs in ForEach.** If the identifiers in a `ForEach` change between updates, every row loses its state and SwiftUI cannot animate inserts and removals properly. Use a stable property from your model, not an array index that shifts when items move.

## When to use each tool

I use structural identity by default and keep `body` shaped the same across updates. I reach for `if/else` when the two states are truly different screens, and I reach for `.id(_:)` only when I want SwiftUI to throw state away on purpose.

My opinion, which not everyone shares, is that every `if/else` inside a SwiftUI body deserves a second look in code review. Many of them are fine, but the ones that only switch a style are the source of a lot of state bugs.

## Takeaway

`@State` lives with a view's identity, not with the struct you wrote. Keep identity stable by putting changing values inside modifiers, use `if/else` for genuinely different content, and give `.id(_:)` a stable value tied to your data. Do that, and your state will stay exactly where you left it.
