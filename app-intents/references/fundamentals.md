# Fundamentals

The core of App Intents is the `AppIntent` protocol. Every exposed app action conforms to it or to one of its subprotocols (`OpenIntent`, `AudioPlaybackIntent`, `VideoCallIntent`, `ForegroundContinuableIntent`, ...).

## Minimal intent

```swift
import AppIntents

struct RefreshFeedIntent: AppIntent {
    static let title: LocalizedStringResource = "Refresh feed"

    func perform() async throws -> some IntentResult {
        .result()
    }
}
```

Three required pieces:

- `static let title: LocalizedStringResource` - human-readable title; displayed in Shortcuts, Siri, focus filter pickers.
- `func perform() async throws -> some IntentResult` - the work. Always `async throws`; runs off the main actor unless you opt in.
- `.result()` - an empty `IntentResult` meaning "done, nothing to return".

`LocalizedStringResource` integrates with string catalogs and with SwiftUI's localization pipeline, so localized strings work everywhere App Intents displays them.

## Optional static metadata

A few more statics fine-tune how the intent presents:

```swift
struct RefreshFeedIntent: AppIntent {
    static let title: LocalizedStringResource = "Refresh feed"
    static let description = IntentDescription(
        "Pulls the latest articles from all your subscribed sources.",
        categoryName: "Reading",
        searchKeywords: ["refresh", "sync", "fetch"],
        resultValueName: "Articles"
    )
    static let isDiscoverable: Bool = true   // default
    static let openAppWhenRun: Bool = false  // default

    func perform() async throws -> some IntentResult { .result() }
}
```

- `description: IntentDescription` - the long-form blurb shown in Shortcuts under the action title. Accepts optional `categoryName:` (Shortcuts library category), `searchKeywords:` (extra search tokens Shortcuts matches against), and `resultValueName:` (the label used when the output of this intent is bound into another action, e.g., "Articles" in "Use Articles from Refresh Feed"). Include these whenever the intent returns a chainable value.
- `isDiscoverable` - when `false`, the intent is invisible in the Shortcuts library and Siri suggestions. Use for helper intents that only exist to back a widget button, a snippet button, or another intent. Keeps the user-facing library clean.
- `openAppWhenRun` - opens the app after `perform()` finishes. Prefer `OpenIntent` for user-visible navigation; reach for this only when the opening is a side-effect of a larger action.

## Common intent subprotocols

`AppIntent` is the baseline. Several subprotocols specialize the behavior; pick the most specific one that fits:

| Protocol | Purpose |
|---|---|
| `AppIntent` | Generic action. |
| `OpenIntent` | Opens the app to a specific entity (`target: MyEntity` parameter). See `open-and-snippet-intents.md`. |
| `SnippetIntent` | Renders a snippet view only - no business logic. Paired with `ShowsSnippetIntent` results. |
| `ForegroundContinuableIntent` | Can bring the app to the foreground mid-perform via `needsToContinueInForegroundError(...)`. For flows that require UI (login, permissions). |
| `DeleteIntent` | Deletes one or more entities; system may prompt for confirmation automatically. |
| `ShowInAppSearchResultsIntent` | Routes a search query into the app's own search UI. |
| `AudioPlaybackIntent` / `AudioStartingIntent` | Plays audio; integrates with lock-screen, CarPlay. |
| `VideoCallIntent` | Starts a video call. |
| `CameraCaptureIntent` | Starts a camera capture flow. |
| `URLRepresentableIntent` | Lets the system open the app via a universal link URL without your `perform()` running. Pairs with `URLRepresentableEntity`. See `open-and-snippet-intents.md`. |
| `TargetContentProvidingIntent` | Marker protocol on iOS - tells the system this intent produces the app scene users navigated to. Needed for visual intelligence routing back into the app. |

## `ForegroundContinuableIntent`

Use this when an intent sometimes can complete in the background but sometimes needs the user to finish a flow in the foreground (sign-in, permissions, confirmation of a destructive action):

```swift
struct SuggestArticlesIntent: ForegroundContinuableIntent {
    static let title: LocalizedStringResource = "Suggest articles"
    @Dependency var account: AccountManager
    @Dependency var navigation: NavigationModel

    @Parameter var topic: String?

    func perform() async throws -> some IntentResult & ReturnsValue<[ArticleEntity]> {
        if !account.loggedIn {
            let dialog = IntentDialog("You aren't logged in. Tap Continue to sign in.")
            throw needsToContinueInForegroundError(dialog) {
                // Configure app UI when the user continues
                navigation.route = .signIn
            }
        }

        // Regular in-background path
        let articles = try await account.suggestions(for: topic)
        return .result(value: articles)
    }
}
```

`needsToContinueInForegroundError(_:)` throws an error, stops the intent, and - if the user taps Continue - launches the app and runs the closure so the app's UI is already positioned correctly. If the user dismisses, the intent is simply cancelled.

Prefer this over `openAppWhenRun = true` when the "open the app" path is conditional.

## Return-type composition

`some IntentResult` is the base. Compose additional capabilities with `&`:

| Conformance | Meaning |
|---|---|
| `IntentResult` | Baseline - intent completed. |
| `ProvidesDialog` | Attaches spoken/shown dialog to the result. |
| `ReturnsValue<T>` | Returns a typed value chainable in Shortcuts. |
| `ShowsSnippetView` | Attaches a SwiftUI snippet view (widget-style). |
| `ShowsSnippetImage` | Attaches a single image. |
| `OpensIntent` | Opens the app when the intent completes. |

Examples:

```swift
// Dialog only
func perform() async throws -> some IntentResult & ProvidesDialog {
    .result(dialog: "Feed refreshed.")
}

// Dialog + chainable value (Int, String, Bool, Double, Date, URL, AppEntity, or an array)
func perform() async throws -> some IntentResult & ProvidesDialog & ReturnsValue<Int> {
    let count = try await service.unreadCount()
    let message = AttributedString(localized: "You have ^[\(count) unread article](inflect: true).")
    return .result(value: count, dialog: "\(message)")
}

// Dialog + SwiftUI snippet
func perform() async throws -> some IntentResult & ProvidesDialog & ShowsSnippetView {
    return .result(dialog: "\(entity.title)") {
        VStack {
            Image(systemName: "doc.text")
                .font(.largeTitle)
            Text(entity.summary)
        }
        .padding()
    }
}
```

The runtime checks the return shape: if you declare `ProvidesDialog` but return `.result()` without a dialog, it crashes at the call site - not at compile time. Match the signature to the call exactly.

## Perform concurrency

`perform()` is `async throws` and not main-actor by default. Anything you touch must either be `Sendable` or hop to the correct actor:

```swift
// Option A: pin the whole perform to the main actor (good when you need UI or SwiftData main context)
@MainActor
func perform() async throws -> some IntentResult {
    let items = try service.recent()
    return .result()
}

// Option B: stay off the main actor; hop only when needed
func perform() async throws -> some IntentResult {
    let summary = try await service.fetchSummary()
    await MainActor.run { uiCoordinator.present(summary) }
    return .result()
}
```

`@MainActor` on `perform()` is the pragmatic choice when the intent reads/writes SwiftData or mutates UI state - it's what Apple's own samples do.

## Intent dialog

`IntentDialog` is constructed by string interpolation of anything `LocalizedStringResource` or `AttributedString`:

```swift
return .result(dialog: "Saved \(count) items to \(folder.name).")
```

For pluralization use Foundation's automatic grammar agreement (markdown syntax, `^[...](inflect: true)`):

```swift
let count = 5
let message = AttributedString(localized: "Added ^[\(count) bookmark](inflect: true).")
return .result(dialog: "\(message)")
```

Output: "Added 1 bookmark." / "Added 5 bookmarks." Automatic agreement works in English, French, German, Italian, Spanish, and Portuguese (both variants). For other locales the text renders as-is, so write the singular form as the base.

You can provide richer dialog variants:

```swift
let dialog: IntentDialog = IntentDialog(
    full: "You have \(count) unread articles in your saved feed.",
    supporting: "\(count) unread"
)
return .result(dialog: dialog)
```

Siri chooses which variant to use based on context (voice vs. screen, short vs. long).

## Refreshing widgets and controls after state changes

When `perform()` mutates data that widgets, control widgets, or live activities display, reload their timelines before returning:

```swift
import AppIntents
import WidgetKit

struct AddBookmarkIntent: AppIntent {
    static let title: LocalizedStringResource = "Add bookmark"

    @Parameter(title: "URL") var url: URL
    @Dependency var store: DataStore

    @MainActor
    func perform() async throws -> some IntentResult & ProvidesDialog {
        try store.addBookmark(url: url)
        WidgetCenter.shared.reloadAllTimelines()
        return .result(dialog: "Saved.")
    }
}
```

`WidgetCenter.shared.reloadAllTimelines()` tells the system every registered widget is stale. For fine-grained reloads use `reloadTimelines(ofKind:)` with the widget kind string. Do this inside `perform()`, before returning - otherwise the widget keeps showing pre-change state until the next refresh tick.

## Errors

Throw from `perform()` to signal failure. Any `Error` works, but App Intents understands these well:

- `NeedsValueError(...)` - request a parameter the user hasn't supplied.
- `RequestDisambiguationError(...)` - ask the user to pick between several options.
- `ConfirmationRequiredError` - ask the user to confirm a destructive action.

```swift
throw $folder.needsValueError("Which folder should this go in?")
```

For general failures, throw a plain `Error` conforming to `CustomLocalizedStringResourceConvertible` so the dialog is localizable.

## Scope: what an intent should do

Apple's guidance (from WWDC24 onwards): "Anything your app does should be an App Intent." The practical interpretation:

- Expose small, discrete actions: refresh, create, append, mark-as-read, open-X, summarize-X.
- Do not authenticate *inside* an intent; assume the user is signed in, and return a `ProvidesDialog` result explaining if they aren't.
- Do not start lengthy UI flows from an intent; either return a snippet, open the app at the right place (`OpenIntent`), or return a value.
- Keep `perform()` reasonably fast; Siri will not wait indefinitely.
