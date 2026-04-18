# Dependencies and data flow

Intents do not have a SwiftUI `@Environment`. They cannot use `@Query`. They get their collaborators via **`@Dependency`**, which reads from a global registry populated by the app.

## The pattern

1. Build a data controller (or service, navigator, network client) in `App.init()`.
2. Register it with `AppDependencyManager.shared.add(dependency:)`.
3. Declare `@Dependency var x: X` in any intent or entity-query that needs it.

```swift
import AppIntents
import SwiftData
import SwiftUI

@main
struct ReaderApp: App {
    @State private var store: DataStore
    @State private var modelContainer: ModelContainer

    init() {
        let modelContainer: ModelContainer

        do {
            modelContainer = try ModelContainer(for: Article.self)
        } catch {
            let config = ModelConfiguration(isStoredInMemoryOnly: true)
            modelContainer = try! ModelContainer(for: Article.self, configurations: config)
        }

        self._modelContainer = .init(initialValue: modelContainer)

        let store = DataStore(modelContainer: modelContainer)
        self._store = .init(initialValue: store)

        AppDependencyManager.shared.add(dependency: store)
    }

    var body: some Scene {
        WindowGroup {
            ContentView(store: store)
        }
        .modelContainer(modelContainer)
    }
}
```

```swift
struct RefreshFeedIntent: AppIntent {
    @Dependency var store: DataStore

    static let title: LocalizedStringResource = "Refresh feed"

    func perform() async throws -> some IntentResult {
        try await store.refresh()
        return .result()
    }
}
```

`@Dependency` looks up the first registered instance of its type. There's no hierarchy or scoping - it's a flat registry. Register once.

## `App.init()` runs for intents too

When a shortcut fires, the OS launches the app process and runs `App.init()` even if no UI is ever created. That's the window for:

- Creating `ModelContainer`s.
- Registering dependencies.
- Wiring any other setup intents will read.

What does **not** run: `.task`, `.onAppear`, `@StateObject` init closures inside views. If you rely on those for intent-needed setup, the intent will crash or see stale state.

## Data-controller skeleton

A data controller concentrates all SwiftData (or network) access in one place so intents don't re-invent queries:

```swift
import Foundation
import SwiftData

@Observable @MainActor
final class DataStore {
    var modelContext: ModelContext
    var path: [Article] = []
    var searchText = ""

    init(modelContainer: ModelContainer) {
        modelContext = ModelContext(modelContainer)
    }

    func articles(
        matching predicate: Predicate<Article> = #Predicate { _ in true },
        sortBy: [SortDescriptor<Article>] = [SortDescriptor(\.publishedAt, order: .reverse)],
        limit: Int? = nil
    ) throws -> [Article] {
        var descriptor = FetchDescriptor<Article>(predicate: predicate, sortBy: sortBy)
        descriptor.fetchLimit = limit
        return try modelContext.fetch(descriptor)
    }

    func articleEntities(
        matching predicate: Predicate<Article> = #Predicate { _ in true },
        sortBy: [SortDescriptor<Article>] = [SortDescriptor(\.publishedAt, order: .reverse)],
        limit: Int? = nil
    ) throws -> [ArticleEntity] {
        try articles(matching: predicate, sortBy: sortBy, limit: limit).map(\.entity)
    }

    func articleCount(
        matching predicate: Predicate<Article> = #Predicate { _ in true }
    ) throws -> Int {
        let descriptor = FetchDescriptor<Article>(predicate: predicate)
        return try modelContext.fetchCount(descriptor)
    }
}
```

Key conventions:

- **Main-actor bound.** The controller drives UI; pinning it keeps SwiftData access serialized and removes sendability noise.
- **Two return shapes** - one returning `[Article]` (for mutation and UI), one returning `[ArticleEntity]` (sendable; safe to hand across actors to intents).
- **Default-valued parameters** so callers can say `store.articles()` for the common case.

## `ModelContainer` vs `ModelContext` sendability

Only `ModelContainer` is `Sendable`. `ModelContext` is not.

- Pass `ModelContainer` across actors. Create a local `ModelContext(modelContainer)` inside each actor that needs one.
- If you pin the intent's `perform()` (or the data controller) to `@MainActor`, you can use the `modelContainer.mainContext` directly.

```swift
// OK - main-actor perform reads main context
@MainActor
func perform() async throws -> some IntentResult {
    let recent = try store.articles(limit: 5)
    ...
    return .result()
}

// OK - cross-actor access via fresh local context
func perform() async throws -> some IntentResult {
    let container = try ModelContainer(for: Article.self)
    let context = ModelContext(container)
    let descriptor = FetchDescriptor<Article>(predicate: #Predicate { _ in true })
    let count = try context.fetchCount(descriptor)
    ...
    return .result()
}
```

Avoid creating ad-hoc `ModelContainer`s from inside `perform()` when a shared one already exists on your `DataStore`. It works but it wastes the container setup and produces leaky code paths.

## Mutating model objects from an intent

To mutate an `@Model` object from an intent, do it on the main actor, on the same main context:

```swift
struct AppendNoteIntent: AppIntent {
    @Dependency var store: DataStore

    @Parameter(title: "Text")
    var newText: String

    static let title: LocalizedStringResource = "Append to latest note"

    @MainActor
    func perform() async throws -> some IntentResult & ProvidesDialog {
        let recent = try store.articles(limit: 1)

        guard let first = recent.first else {
            return .result(dialog: "You haven't saved anything yet.")
        }

        first.body.append(" \(newText)")
        try first.modelContext?.save()

        return .result(dialog: "Added.")
    }
}
```

Two things make this work:

1. Both the intent's `perform()` and the data controller are `@MainActor`. No sending of non-sendable data across actors.
2. `first` is the actual `Article` instance, not an `ArticleEntity` copy - so mutations persist.

Calling `try first.modelContext?.save()` explicitly is recommended. SwiftData's autosave is unreliable from intents because the app may be torn down before the next run loop.

## Don't authenticate inside `perform()`

If your app requires login, assume the user is already signed in. If they aren't, return early with a `ProvidesDialog`:

```swift
guard store.isAuthenticated else {
    return .result(dialog: "Sign in to the app first.")
}
```

Don't present an auth sheet from an intent - you'll strand the user in Siri or Shortcuts.

## Shared framework extraction (iOS 18+)

For larger apps with many intents, split intents into their own framework target. `@Dependency` resolution works across frameworks in the same app as long as the dependency is registered in `App.init()`.
