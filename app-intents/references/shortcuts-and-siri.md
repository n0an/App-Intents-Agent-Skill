# Shortcuts and Siri

Writing an `AppIntent` type is not enough. The system discovers intents through an `AppShortcutsProvider`. Anything not listed there is invisible to Shortcuts, Siri suggestions, the action button picker, and most automation surfaces.

## `AppShortcutsProvider`

```swift
import AppIntents

struct ReaderShortcuts: AppShortcutsProvider {
    static let shortcutTileColor: ShortcutTileColor = .blue

    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: RefreshFeedIntent(),
            phrases: [
                "Refresh my feed in \(.applicationName)",
                "Get new articles in \(.applicationName)"
            ],
            shortTitle: "Refresh Feed",
            systemImageName: "arrow.clockwise"
        )

        AppShortcut(
            intent: OpenArticleIntent(),
            phrases: [
                "Open an article in \(.applicationName)"
            ],
            shortTitle: "Open Article",
            systemImageName: "doc.text"
        )
    }
}
```

Exactly one `AppShortcutsProvider` per app. It's a static declaration and gets scanned at build time.

## The `\(.applicationName)` rule

Every phrase must include `\(.applicationName)` somewhere:

```swift
// WRONG - build error: "Every app shortcut phrase needs to contain the applicationName"
phrases: ["Refresh my feed"]

// CORRECT
phrases: ["Refresh my feed in \(.applicationName)"]
```

This is enforced by the macro at compile time. The reason: without the app name, phrases collide with other apps' commands. "Set a timer for 5 minutes" belongs to the Clock app; your app can't hijack it.

There's no way around this - never hard-code the name string. The interpolation expands to the bundle's display name and keeps working after app renames.

## Phrase coverage

Phrases are exact-match. Siri does not paraphrase or interpret loosely, so provide multiple wordings:

```swift
AppShortcut(
    intent: AppendNoteIntent(),
    phrases: [
        "Append to my latest note in \(.applicationName)",
        "Add to my most recent note in \(.applicationName)",
        "Save this to \(.applicationName)"
    ],
    shortTitle: "Append to Latest Note",
    systemImageName: "plus"
)
```

Short, natural forms beat long, formal ones. Think of what a user would actually say aloud.

## Titles vs short titles

`AppIntent.title` and `AppShortcut.shortTitle` are different and both shown:

- `title` appears in the Shortcuts action list when a user builds a multi-action shortcut (e.g., "Refresh feed").
- `shortTitle` appears as the action button tile in Shortcuts home, in the app's Shortcut gallery, and in Siri's "what can I do here" sheet.

Give them different-enough wording during development ("Count Recent Dreams" vs "Recent Dream Count") to see which surface is which; settle on consistent phrasing before shipping.

## Parameterising an `AppShortcut`

`AppShortcut` can be configured by the intent's `@Parameter`s - this lets one intent surface several ready-made phrases:

```swift
AppShortcut(
    intent: SearchArticlesIntent(),
    phrases: [
        "Search \(\.$query) in \(.applicationName)",
        "Find \(\.$query) in \(.applicationName)"
    ],
    shortTitle: "Search",
    systemImageName: "magnifyingglass"
)
```

`\(\.$query)` is a parameter key path - the user's utterance fills it in.

### Refreshing parameter-driven phrases: `updateAppShortcutParameters()`

When a phrase uses a key path to an entity parameter (e.g., `\(\.$folder)`), the system caches the list of candidate values it will show. When your underlying entity data changes - a new folder is added, a bookmark is renamed - call `updateAppShortcutParameters()` to invalidate the cache:

```swift
@main
struct ReaderApp: App {
    init() {
        ReaderShortcuts.updateAppShortcutParameters()

        let store = DataStore(...)
        self._store = .init(initialValue: store)
        AppDependencyManager.shared.add(dependency: store)
    }
    ...
}
```

Call it:

- Once in `App.init()` to make sure the current set is seeded.
- Again whenever an entity that appears in a shortcut phrase key path changes (creation, rename, deletion).

Without this, Siri-suggested phrases can point at stale entity names or offer deleted items.

## `shortcutTileColor`

```swift
static let shortcutTileColor: ShortcutTileColor = .navy
```

Options: `.grayBlue`, `.red`, `.orange`, `.yellow`, `.green`, `.teal`, `.blue`, `.indigo`, `.purple`, `.pink`, `.navy`, `.lightBlue`, `.gray`, `.lime`. The color is used by the Shortcuts app for the app's tiles.

## In-app discoverability

Two SwiftUI helpers nudge users toward shortcuts right inside your app.

### `ShortcutsLink`

Opens directly to the app's page in the Shortcuts app:

```swift
import AppIntents

var body: some View {
    Section {
        ...
    } footer: {
        ShortcutsLink()
    }
}
```

One line. No parameters. Fine to place in a Settings screen, an onboarding sheet, or a list footer.

### `SiriTipView`

Suggests a specific Siri phrase for one of your intents:

```swift
@AppStorage("suggest.refreshFeed") var showTip = true

SiriTipView(intent: RefreshFeedIntent(), isVisible: $showTip)
```

When `isVisible` is bound, the tip has an 'x' to dismiss it - persist the dismissal through `@AppStorage` so it doesn't re-appear. The displayed phrase is read from the intent's registered `AppShortcut` phrases.

### "What can I do here?"

On a real device, saying "what can I do here?" to Siri asks the OS to scan the current app's registered intents and show them. This works automatically once your `AppShortcutsProvider` is registered - no extra code needed. It's a powerful discoverability lever for users who already know Siri exists.

## Presenting intent parameters

`AppShortcut` takes an optional `parameterPresentation` to change how Shortcuts renders parameter pickers for that specific phrase. Use it to pre-fill parameter labels or example values. It's sparsely documented; reach for it only when default rendering is insufficient.

## What NOT to register

- Do not register intents you only want to use as widget configuration. Widget-configuration intents (`WidgetConfigurationIntent`) are resolved through widget kit, not through `AppShortcutsProvider`.
- Do not register intents meant solely as building blocks for other intents - keep them internal.
- If an intent should only run inside your own app code (e.g., from a button), you don't have to register it at all. Registration is the publication step to the rest of the system.
