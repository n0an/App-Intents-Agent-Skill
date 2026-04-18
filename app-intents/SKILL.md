---
name: app-intents
description: Writes and reviews Swift App Intents code that exposes app actions and data to Siri, Shortcuts, Spotlight, widgets, Control Center, and Apple Intelligence. Use when adding AppIntent, AppEntity, OpenIntent, AppShortcutsProvider, EntityQuery, Focus Filters, AssistantEntity/AssistantIntent schemas, or when wiring SwiftData/networked data into intents.
license: MIT
metadata:
  author: Anton Novoselov
  version: "1.0"
---

Write and review Swift code that exposes app functionality through the App Intents framework, ensuring correct protocol conformance, safe data flow, and idiomatic discoverability wiring.

Review process:

1. Check intent fundamentals (protocol, `perform()`, return types, dialog) using `references/fundamentals.md`.
1. Validate parameters and parameter options using `references/parameters.md`.
1. Validate entity types, display representations, and queries using `references/entities.md`.
1. Check the `AppShortcutsProvider` registration and discoverability UI using `references/shortcuts-and-siri.md`.
1. Validate `OpenIntent` navigation and snippet view return types using `references/open-and-snippet-intents.md`.
1. Check dependency injection and the data-controller pattern using `references/dependencies.md`.
1. Validate Spotlight indexing via `IndexedEntity` and attribute sets using `references/spotlight.md`.
1. Check `@AssistantEntity` / `@AssistantIntent` schema adoption using `references/assistant-schemas.md`.
1. Catch common mistakes using `references/anti-patterns.md`.

If doing partial work, load only the relevant reference files.


## Core Instructions

- Target iOS 16+ / macOS 13+ minimum for App Intents. `IndexedEntity`, `OpenIntent`, focus filters, and control widgets require iOS 16+; `@AssistantEntity` / `@AssistantIntent` schemas require iOS 18.2+; anchored relative date styles and many assistant schemas require iOS 18.4+.
- **Never** make a SwiftData `@Model` class or other reference-type data model conform to `AppEntity`. `AppEntity` requires `Sendable`; `@Model` classes are not sendable. Create a separate `struct` entity that shadows the fields you want to expose.
- **Never** pass `ModelContext` across actor boundaries. `ModelContext` is not sendable. Pass `ModelContainer` (which is sendable) and create a local context inside the actor that needs one.
- **Never** expose an intent to Siri/Spotlight only by writing its type, always register it through an `AppShortcutsProvider`. Types not registered there will not appear in Shortcuts, Siri suggestions, or the action button picker.
- **Never** write a Siri activation phrase without interpolating `\(.applicationName)`. The App Intents macro rejects phrases without it at compile time, because phrases without the app name would collide with other apps' commands.
- **Never** reach back into a SwiftUI `@Query` from inside an intent. `@Query` only works inside a `View`. Run a one-shot `FetchDescriptor` through a `ModelContext` instead, or route through a centralized data controller.
- **Never** instantiate services, data stores, or authentication managers inside `perform()`. Inject them through `@Dependency` and register them once in `App.init()` with `AppDependencyManager.shared.add(dependency:)`.
- **Never** use `String(format:)` or manual concatenation for localized intent dialog. Use `LocalizedStringResource`, and use Foundation's grammar-agreement markdown (`^[\(count) item](inflect: true)`) inside an `AttributedString` for pluralization.
- Prefer `OpenIntent` for "take me to this thing" actions, `AppIntent & ShowsSnippetView` for self-contained one-shot summaries, and `AppIntent & ShowsSnippetIntent` + a paired `SnippetIntent` when the snippet contains `Button(intent:)` and needs to re-render after buttons fire.
- Always set `static let isDiscoverable: Bool = false` on helper intents that only back a widget button, snippet button, or other intent - otherwise they pollute the user's Shortcuts library.
- When an intent mutates data that widgets or control widgets display, call `WidgetCenter.shared.reloadAllTimelines()` inside `perform()` before returning.
- When `Button(intent:)` lives inside a widget view, share state between the intent (runs in the app process) and the widget's timeline provider (runs in the extension process) via App Group `UserDefaults(suiteName:)` or a shared `ModelContainer` URL - never `UserDefaults.standard` or in-memory `@Dependency` state.
- When entity data that appears in a shortcut phrase's key path changes (creation, rename, deletion), call `YourShortcutsProvider.updateAppShortcutParameters()` to invalidate the cached candidate list.
- Prefer `EnumerableEntityQuery` when the whole set is small and cheap to load; implement `EntityQuery` + `EntityStringQuery` when the dataset is large or searchable. `entities(for identifiers:)` is mandatory on every query; without it, parameter resolution breaks.
- The app's `App` struct initializer is executed when an intent runs, even if the UI never appears. Do all intent-relevant setup (`ModelContainer` creation, `AppDependencyManager.shared.add(...)`, log plumbing) inside `init()`, not inside view modifiers like `.task` or `.onAppear`.
- `LocalizedStringResource` is the standard string type everywhere in App Intents (titles, dialog, parameter prompts). It shares string catalogs with SwiftUI, so localization works out of the box.
- Grammar agreement (`inflect: true`) works in English, French, German, Italian, Spanish, and Portuguese (both variants). For other locales it falls back to the unmodified form.


## Output Format

If the user asks for a review, organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the anti-pattern being replaced.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

If the user asks you to write or fix intent code, make the changes directly instead of returning a findings report.

Example output:

### RecentItemsIntent.swift

**Line 18: SwiftData `@Model` cannot conform to `AppEntity` (not `Sendable`).**

```swift
// Before
extension Article: AppEntity { ... }

// After
struct ArticleEntity: AppEntity {
    var id: UUID
    var title: String
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    static let defaultQuery = ArticleEntityQuery()
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)")
    }
}
```

**Line 42: Siri phrase missing `\(.applicationName)` - will fail to build.**

```swift
// Before
AppShortcut(intent: OpenArticleIntent(), phrases: ["Open an article"], ...)

// After
AppShortcut(intent: OpenArticleIntent(), phrases: ["Open an article in \(.applicationName)"], ...)
```

### Summary

1. **Sendability (high):** `Article` as `AppEntity` will fail to compile on Swift 6; create a shadow struct.
2. **Siri phrases (high):** `\(.applicationName)` is required in every phrase.

End of example.


## References

- `references/fundamentals.md` - `AppIntent` protocol, `perform()`, return types (`IntentResult`, `ProvidesDialog`, `ReturnsValue`, `ShowsSnippetView`, `OpensIntent`), intent dialog, grammar agreement.
- `references/parameters.md` - `@Parameter`, primitive vs entity parameters, `@AppEnum`, dialog options, parameter prompts.
- `references/entities.md` - `AppEntity`, `IndexedEntity`, shadow-struct pattern, display representations, entity queries (`EnumerableEntityQuery`, `EntityQuery`, `EntityStringQuery`, `UniqueIDEntityQuery`).
- `references/shortcuts-and-siri.md` - `AppShortcutsProvider`, phrases (`\(.applicationName)` rule), `shortcutTileColor`, `SiriTipView`, `ShortcutsLink`, parameter presentation.
- `references/open-and-snippet-intents.md` - `OpenIntent`, snippet views (`ShowsSnippetView`), navigation via data controller, when to use which.
- `references/dependencies.md` - `@Dependency`, `AppDependencyManager`, data-controller pattern, `ModelContainer` vs `ModelContext` sendability, main-actor vs local-context tradeoff.
- `references/spotlight.md` - `IndexedEntity`, `CSSearchableIndex`, `attributeSet`, index-on-launch vs index-on-change, debounced reindexing.
- `references/assistant-schemas.md` - `@AssistantEntity`, `@AssistantIntent`, schema adoption, Xcode code snippets, caveats.
- `references/anti-patterns.md` - common mistakes LLMs make when generating App Intents code.
