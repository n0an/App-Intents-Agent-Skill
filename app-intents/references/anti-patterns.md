# Anti-patterns: common App Intents mistakes

App Intents has evolved quickly across iOS 16, 17, and 18. Most LLM training data predates the modern shape of the framework, so models often generate code from older patterns (SiriKit, pre-Swift-6 SwiftData, NSUserActivity). These are the mistakes to catch.

## SwiftData `@Model` conforming to `AppEntity`

This is the single most common mistake.

```swift
// WRONG - @Model is not Sendable, AppEntity requires Sendable
@Model
final class Article { ... }

extension Article: AppEntity {
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    ...
}
// Error: "Conformance of 'Article' to 'Sendable' unavailable"
```

Fix: create a separate **shadow struct** that conforms to `AppEntity`, and map model → entity at the query boundary.

```swift
// CORRECT
struct ArticleEntity: AppEntity {
    var id: UUID
    var title: String
    // ... only the fields you want to expose
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    static let defaultQuery = ArticleEntityQuery()
    var displayRepresentation: DisplayRepresentation { DisplayRepresentation(title: "\(title)") }
}

@Model
final class Article {
    var entity: ArticleEntity {
        ArticleEntity(id: id, title: title)
    }
}
```

## Using SwiftUI `@Query` inside an intent

`@Query` is a property wrapper that only works inside `View`. It silently does nothing from an intent.

```swift
// WRONG - @Query has no effect here
struct CountArticlesIntent: AppIntent {
    @Query var articles: [Article]   // not populated

    func perform() async throws -> some IntentResult {
        let count = articles.count   // always 0
        ...
    }
}

// CORRECT - use FetchDescriptor or a centralised data controller
struct CountArticlesIntent: AppIntent {
    @Dependency var store: DataStore

    @MainActor
    func perform() async throws -> some IntentResult & ReturnsValue<Int> {
        let count = try store.articleCount()
        return .result(value: count)
    }
}
```

## Siri phrase without `\(.applicationName)`

```swift
// WRONG - build error: "Every app shortcut phrase needs to contain the applicationName"
AppShortcut(
    intent: RefreshFeedIntent(),
    phrases: ["Refresh my feed"],
    shortTitle: "Refresh Feed",
    systemImageName: "arrow.clockwise"
)

// CORRECT
AppShortcut(
    intent: RefreshFeedIntent(),
    phrases: ["Refresh my feed in \(.applicationName)"],
    shortTitle: "Refresh Feed",
    systemImageName: "arrow.clockwise"
)
```

Enforced at compile time via macro. Never hardcode the app's name as a string - `\(.applicationName)` keeps working after a rename.

## Intent defined but never registered

Writing the `AppIntent` type is step one. Without an `AppShortcutsProvider` listing it, the intent is invisible to Shortcuts, Siri suggestions, the action button picker, and focus filters.

```swift
// Intent exists but nobody registers it - users will never see it
struct RefreshFeedIntent: AppIntent { ... }

// The missing piece
struct ReaderShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: RefreshFeedIntent(),
            phrases: ["Refresh my feed in \(.applicationName)"],
            shortTitle: "Refresh Feed",
            systemImageName: "arrow.clockwise"
        )
    }
}
```

There is exactly one `AppShortcutsProvider` per app. Intents meant purely for widgets or internal use don't need to be registered.

## Creating `ModelContainer` / `ModelContext` inside `perform()`

Works, but wasteful and leaks SwiftData concerns into every intent.

```swift
// WRONG (duplicative; hard to maintain)
func perform() async throws -> some IntentResult & ReturnsValue<Int> {
    let container = try ModelContainer(for: Article.self)
    let context = ModelContext(container)
    let descriptor = FetchDescriptor<Article>()
    let count = try context.fetchCount(descriptor)
    return .result(value: count)
}

// CORRECT - inject once in App.init(), use everywhere
struct CountArticlesIntent: AppIntent {
    @Dependency var store: DataStore

    @MainActor
    func perform() async throws -> some IntentResult & ReturnsValue<Int> {
        let count = try store.articleCount()
        return .result(value: count)
    }
}
```

## Passing `ModelContext` across actors

Only `ModelContainer` is `Sendable`. `ModelContext` is not.

```swift
// WRONG - will fail with Swift 6 strict concurrency
actor Indexer {
    func index(context: ModelContext) async { ... }
}

// CORRECT - pass the container, make a local context inside the actor
actor Indexer {
    let container: ModelContainer

    func index() async {
        let context = ModelContext(container)
        ...
    }
}
```

## Returning the wrong `IntentResult` shape

The declared return type must match what `.result(...)` actually returns. Mismatches crash at runtime, not at compile time.

```swift
// WRONG - declares ProvidesDialog but returns an empty result
func perform() async throws -> some IntentResult & ProvidesDialog {
    return .result()   // runtime: "missing dialog"
}

// WRONG - declares ReturnsValue<Int> but returns a value: String
func perform() async throws -> some IntentResult & ReturnsValue<Int> {
    return .result(value: "five")   // runtime: type mismatch
}

// CORRECT
func perform() async throws -> some IntentResult & ProvidesDialog & ReturnsValue<Int> {
    return .result(value: 5, dialog: "You have 5 items.")
}
```

Keep the type narrow until the call site forces you to widen it.

## `String(format:)` or manual plural logic for dialog

```swift
// WRONG
let s = count == 1 ? "item" : "items"
return .result(dialog: "You have \(count) \(s).")

// WRONG - misses locale
return .result(dialog: String(format: "%d items", count))

// CORRECT - Foundation grammar agreement
let message = AttributedString(localized: "You have ^[\(count) item](inflect: true).")
return .result(dialog: "\(message)")
```

Grammar agreement works for English, French, German, Italian, Spanish, and Portuguese (both variants). Other locales fall back to the base form.

## Omitting `entities(for:)` on a query

`entities(for identifiers:)` is mandatory. Without it, parameter resolution silently breaks - pickers show results, but Shortcuts fails to reload them on re-evaluation.

```swift
// WRONG - only half of EnumerableEntityQuery
struct FolderEntityQuery: EnumerableEntityQuery {
    func allEntities() async throws -> [FolderEntity] { ... }
    // missing entities(for:) - does not conform
}

// CORRECT
struct FolderEntityQuery: EnumerableEntityQuery {
    func allEntities() async throws -> [FolderEntity] { ... }

    func entities(for identifiers: [FolderEntity.ID]) async throws -> [FolderEntity] {
        try await store.folderEntities(matching: #Predicate { identifiers.contains($0.id) })
    }
}
```

## Setting up dependencies in a view modifier

`App.init()` runs for intents; `.onAppear` and `.task` do not (no UI is created when an intent fires).

```swift
// WRONG - dependency not registered when intent runs without UI
var body: some Scene {
    WindowGroup {
        ContentView()
            .onAppear {
                AppDependencyManager.shared.add(dependency: store)
            }
    }
}

// CORRECT - register in init, before any intent can run
init() {
    let store = DataStore(...)
    self._store = .init(initialValue: store)
    AppDependencyManager.shared.add(dependency: store)
}
```

## Using `NSUserActivity` or legacy `SiriKit` for new functionality

Old patterns: donating `NSUserActivity`, using `INIntent` and `INExtension` targets, subclassing `INExtension` for Siri responses. These still work in some cases but aren't the modern path.

Use App Intents for anything new. `NSUserActivity` is still the way to handle handoff; you can associate `AppEntity` with an existing `NSUserActivity`, but don't build new Siri integrations on top of `SiriKit`.

## Over-authenticating in `perform()`

```swift
// WRONG - blocks a Siri response on an auth sheet that can't display there
func perform() async throws -> some IntentResult {
    let token = try await showSignInSheet()
    ...
}

// CORRECT - bail out with a friendly dialog; user signs in next time they open the app
func perform() async throws -> some IntentResult & ProvidesDialog {
    guard store.isAuthenticated else {
        return .result(dialog: "Please sign in first.")
    }
    ...
}
```

## Missing `@MainActor` for SwiftData mutation

Mutating a SwiftData `@Model` object from inside `perform()` without main-actor guarantees produces sendability warnings (Swift 6) or data corruption (earlier modes).

```swift
// CORRECT when the intent mutates model objects
@MainActor
func perform() async throws -> some IntentResult & ProvidesDialog {
    let first = try store.articles(limit: 1).first
    first?.lastOpened = .now
    try first?.modelContext?.save()
    return .result(dialog: "Done.")
}
```

Alternatively, perform writes in a custom `ModelActor` and call it from the intent.

## Using `#Predicate` with entity property paths

SwiftData's `#Predicate` macro doesn't reach through entity property paths; copy the id to a local first.

```swift
// WRONG - macro error or wrong results
try store.articles(matching: #Predicate { $0.id == entity.id })

// CORRECT
let id = entity.id
try store.articles(matching: #Predicate { $0.id == id })
```

## Interactive SwiftUI controls inside a snippet view

Snippet views are rendered like widgets. Anything that needs a live `UIViewController` (scroll views, lists, text fields, maps in many cases) either doesn't render or renders incorrectly.

```swift
// WRONG - ScrollView is a platform view inside a snippet
return .result(dialog: "\(entity.title)") {
    ScrollView {
        Text(entity.longSummary)
    }
}

// CORRECT - static layout only
return .result(dialog: "\(entity.title)") {
    VStack(alignment: .leading) {
        Text(entity.title).font(.headline)
        Text(entity.summary).font(.body)
    }
    .padding()
}
```

If you need interaction, use `OpenIntent` and open the app instead.

## `target` parameter renamed on an `OpenIntent`

`OpenIntent` matches `target` by exact name.

```swift
// WRONG - the protocol's default matching looks for `target`
struct OpenArticleIntent: OpenIntent {
    @Parameter var article: ArticleEntity   // wrong property name
    ...
}

// CORRECT
struct OpenArticleIntent: OpenIntent {
    @Parameter var target: ArticleEntity
    ...
}
```

## Spotlight: mutating `CSSearchableItemAttributeSet` from scratch

Start from `defaultAttributeSet` - the system fills in type identifiers, display metadata, and a bunch of defaults you'd otherwise forget.

```swift
// WRONG - loses system defaults
var attributeSet: CSSearchableItemAttributeSet {
    let set = CSSearchableItemAttributeSet(contentType: .item)
    set.contentDescription = summary
    return set
}

// CORRECT
var attributeSet: CSSearchableItemAttributeSet {
    let set = defaultAttributeSet
    set.contentDescription = summary
    set.addedDate = publishedAt
    return set
}
```
