# Anti-patterns: common App Intents mistakes

App Intents has evolved quickly across iOS 16, 17, 18, 26, and 27. Most LLM training data predates the modern shape of the framework, so models often generate code from older patterns (SiriKit, pre-Swift-6 SwiftData, NSUserActivity). These are the mistakes to catch. The Siri AI Developer Preview (27.3) entries are grouped at the end.

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

## Helper intents pollute the Shortcuts library (missing `isDiscoverable = false`)

Intents that only exist to back a widget button, snippet button, or another intent should not show up in the user's Shortcuts library.

```swift
// WRONG - shows up in Shortcuts even though it's an implementation detail
struct LogAmountIntent: AppIntent {
    static let title: LocalizedStringResource = "Log caffeine amount"
    @Parameter var amount: Int
    ...
}

// CORRECT
struct LogAmountIntent: AppIntent {
    static let title: LocalizedStringResource = "Log caffeine amount"
    static let isDiscoverable: Bool = false
    @Parameter var amount: Int
    ...
}
```

Same rule for `SnippetIntent` types used indirectly via `ShowsSnippetIntent` - they're always internal and must be `isDiscoverable = false`.

## `Button(intent:)` without a matching init

`Button(intent:)` takes an intent *instance*. If the intent has parameters, you need a convenience init that accepts them:

```swift
struct LogAmountIntent: AppIntent {
    static let title: LocalizedStringResource = "Log amount"
    static let isDiscoverable: Bool = false
    @Parameter var amount: Int
    ...
}

// WRONG - can't construct LogAmountIntent(amount:) without the extra init
Button(intent: LogAmountIntent(amount: 64)) { Text("Single") }   // won't compile

// Fix: add the init
extension LogAmountIntent {
    init(amount: Int) { self.amount = amount }
}
```

Don't try to work around it by assigning to `@Parameter`-wrapped properties after construction from outside; use the custom init.

## Using `ShowsSnippetView` where `ShowsSnippetIntent` is cleaner

Inline `ShowsSnippetView` works, but it binds the snippet's UI to the business intent's `perform()`. If the snippet contains interactive buttons (`Button(intent:)`) that need to refresh the view after they fire, prefer the two-intent pattern:

```swift
// OK, but rigid - fine for static summaries
func perform() async throws -> some IntentResult & ShowsSnippetView {
    .result { SummaryView(data: ...) }
}

// BETTER when the snippet has Button(intent:) inside it
func perform() async throws -> some IntentResult & ReturnsValue<Int> & ShowsSnippetIntent {
    .result(value: count, snippetIntent: DashboardSnippet())
}
```

The two-intent form lets the snippet re-render in place when its buttons fire; the inline form can't.

## Forgetting `updateAppShortcutParameters()` after entity changes

Shortcut phrases that reference entity parameters (e.g., `"Open \(\.$folder) in \(.applicationName)"`) cache the candidate list. When a folder is renamed or a new one is added, Siri-suggested phrases can show stale or missing values.

```swift
// Add after any change that affects the parameter's candidate list
func createFolder(_ name: String) {
    store.insert(Folder(name: name))
    try? modelContext.save()
    ReaderShortcuts.updateAppShortcutParameters()   // refresh phrase cache
}
```

Also call it once in `App.init()` to seed the cache on first launch.

## Widget-fired intent writes to unshared storage

`Button(intent:)` inside a widget fires an intent that runs in the *app's* process. The widget view lives in the *widget extension's* process. They share neither memory nor standard `UserDefaults`.

```swift
// WRONG - widget never sees the new value; TimelineProvider reads old data
struct IncrementIntent: AppIntent {
    static let title: LocalizedStringResource = "Increment"
    static let isDiscoverable: Bool = false

    func perform() async throws -> some IntentResult {
        let count = UserDefaults.standard.integer(forKey: "count")
        UserDefaults.standard.set(count + 1, forKey: "count")
        WidgetCenter.shared.reloadAllTimelines()
        return .result()
    }
}

// CORRECT - both processes read/write the same suite
enum SharedCounter {
    private static let defaults = UserDefaults(suiteName: "group.com.example.myapp")!

    static var current: Int { defaults.integer(forKey: "count") }
    static func increment() { defaults.set(current + 1, forKey: "count") }
}

struct IncrementIntent: AppIntent {
    static let title: LocalizedStringResource = "Increment"
    static let isDiscoverable: Bool = false

    func perform() async throws -> some IntentResult {
        SharedCounter.increment()
        WidgetCenter.shared.reloadAllTimelines()
        return .result()
    }
}
```

Requires adding the same App Group capability to both the main app target and the widget extension target. See `open-and-snippet-intents.md` for the full pattern, including shared SwiftData stores.

## `@Dependency` used in a widget-fired intent

`@Dependency` reads from `AppDependencyManager`, which is populated in `App.init()`. When a widget fires an intent, the app's `init()` has run (the OS launches the app process), so `@Dependency` works - but only for app-process state. Anything stored in-memory-only (a `@State` on a view, a `@Published` on a view-model) is not shared with the widget's timeline provider running in the extension.

If the widget needs to display state the intent mutated, the state must be persisted somewhere both processes can see - App Group `UserDefaults`, a shared file, or a `ModelContainer` whose URL is inside the App Group's shared container. Don't try to "inject" app-only state into the widget side through `@Dependency`.

## Missing `WidgetCenter.reloadAllTimelines()` after intent writes

Widgets keep showing old data until the next scheduled refresh if the intent that wrote the data doesn't ask for a reload.

```swift
// WRONG - widget is stale until the next scheduled refresh
@MainActor
func perform() async throws -> some IntentResult {
    try store.log(amount)
    return .result()
}

// CORRECT
@MainActor
func perform() async throws -> some IntentResult {
    try store.log(amount)
    WidgetCenter.shared.reloadAllTimelines()
    return .result()
}
```

For apps with many widget kinds, use `reloadTimelines(ofKind:)` to avoid unnecessary work.

## Entity fields without `@Property` aren't queryable

Plain stored or computed properties on an `AppEntity` are visible to your code but invisible to Shortcuts, Find intents, and parameter summaries. If a user would reasonably want to filter or sort by the field, mark it `@Property` (or `@ComputedProperty` for derived values):

```swift
// WRONG - `trailLength` can never be used in Shortcuts, Find intents, or parameter summaries
struct TrailEntity: AppEntity {
    @Property var name: String
    var trailLength: Measurement<UnitLength>   // plain - invisible to the system
    ...
}

// CORRECT
struct TrailEntity: AppEntity {
    @Property var name: String
    @Property var trailLength: Measurement<UnitLength>
    ...
}
```

Ask: "Could a user want to build a shortcut that filters or sorts by this field?" If yes, wrap it.

## `EntityQuery` alone when `EntityPropertyQuery` would auto-generate a Find intent

When an entity has several `@Property` fields users might filter on, conforming the query to `EntityPropertyQuery` automatically gives the user a "Find [Entity]" action in Shortcuts with full predicate building and sorting. You don't have to write that UI.

```swift
// OK but leaves features on the table
struct TrailEntityQuery: EntityQuery {
    func entities(for identifiers: [TrailEntity.ID]) async throws -> [TrailEntity] { ... }
    func suggestedEntities() async throws -> [TrailEntity] { ... }
}

// BETTER - user gets a "Find Trail" action automatically
extension TrailEntityQuery: EntityPropertyQuery {
    static let properties = QueryProperties {
        Property(\TrailEntity.$name) {
            ContainsComparator { ... }
            EqualToComparator { ... }
        }
        Property(\TrailEntity.$trailLength) {
            LessThanOrEqualToComparator { ... }
        }
    }
    static let sortingOptions = SortingOptions {
        SortableBy(\TrailEntity.$name)
        SortableBy(\TrailEntity.$trailLength)
    }
    func entities(matching: [Predicate<TrailEntity>], mode: ComparatorMode,
                  sortedBy: [EntityQuerySort<TrailEntity>], limit: Int?) async throws -> [TrailEntity] { ... }
}
```

Skip `EntityPropertyQuery` only if the dataset is genuinely enumerable (small, fixed) - in which case prefer `EnumerableEntityQuery`.

## Implementing `perform()` on a `URLRepresentableIntent`

If the entity conforms to `URLRepresentableEntity` and the intent conforms to both `OpenIntent` and `URLRepresentableIntent`, the system opens the URL via the app's universal-link handler automatically. Writing a `perform()` body *replaces* that automation - you lose the URL routing path.

```swift
// WRONG - perform() runs instead of URL routing; duplicates universal-link logic
struct OpenTrail: OpenIntent, URLRepresentableIntent {
    @Parameter var target: TrailEntity

    @MainActor
    func perform() async throws -> some IntentResult {
        navigator.route = .trail(target.id)   // duplicate of what .onOpenURL already does
        return .result()
    }
}

// CORRECT - no perform(), system uses urlRepresentation
struct OpenTrail: OpenIntent, URLRepresentableIntent {
    @Parameter var target: TrailEntity
    // no perform()
}
```

Pick one routing mechanism. If the app has mature universal-link handling, use the URL path. If it doesn't, implement `perform()` without `URLRepresentableIntent`.

## Returning computed / aggregated data as `AppEntity` instead of `TransientAppEntity`

`AppEntity` requires a persistent identifier and a working `EntityQuery`. Computed summaries (total steps today, current weather, aggregated workout stats) have no id and can't be looked up by one - they're recomputed each request. Using `AppEntity` for them forces an awkward `EntityQuery` that can only return one "current" value.

```swift
// WRONG - no real id, EntityQuery makes no sense
struct WorkoutSummary: AppEntity {
    var id: UUID   // always .init(), meaningless
    @Property var totalSteps: Int
    static let defaultQuery = DummyQuery()   // awkward
    ...
}

// CORRECT
struct WorkoutSummary: TransientAppEntity {
    static let typeDisplayRepresentation = TypeDisplayRepresentation(name: "Workout Summary")
    @Property var totalSteps: Int
    init() { totalSteps = 0 }
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "Workout Summary", subtitle: "\(totalSteps) steps")
    }
}
```

Rule of thumb: if you can't answer "what does `entities(for: [id])` do here?", you want `TransientAppEntity`.

## Missing `Transferable` conformance on entities used onscreen

For entities surfaced via `userActivity(_:element:)`, Siri / Apple Intelligence can *identify* the entity but can't do anything useful with its *content* unless the entity conforms to `Transferable`. Without it, "what can I do with this?" returns empty.

```swift
// Viewable by Siri but opaque
struct PhotoEntity: AppEntity { ... }

// Usable by Siri (forwardable as image/PDF/text)
struct PhotoEntity: AppEntity { ... }

extension PhotoEntity: Transferable {
    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(exportedContentType: .png) { entity in try await entity.pngData() }
        DataRepresentation(exportedContentType: .plainText) { entity in entity.caption.data(using: .utf8)! }
    }
}
```

For schema-adopted entities (e.g., `.photos.asset`), `Transferable` conformance is practically required - several consuming features assume it.

## Mutating app state inside `SnippetIntent.perform()`

iOS 26's snippet refresh cycle calls the snippet intent's `perform()` multiple times per user interaction - on first show, after each button tap, on dark-mode toggle, on `SnippetIntent.reload()`. If `perform()` has side effects, they run repeatedly.

```swift
// WRONG - logs an analytics event every time the snippet re-renders
struct DashboardSnippetIntent: SnippetIntent {
    @Dependency var analytics: Analytics
    func perform() async throws -> some IntentResult & ShowsSnippetView {
        analytics.log("dashboard_viewed")   // fires on every refresh, not just opens
        return .result(view: DashboardView(data: store.current))
    }
}

// CORRECT - pure view construction; side effects belong in the button's intent
struct DashboardSnippetIntent: SnippetIntent {
    func perform() async throws -> some IntentResult & ShowsSnippetView {
        .result(view: DashboardView(data: store.current))
    }
}
```

Snippet-intent `perform()` should read state, assemble the view, and return. Writes and side effects live in the *button's* intent.

## Expensive work inside `SnippetIntent.perform()`

For the same reason, slow operations make the snippet feel unresponsive - users see a spinner instead of content. Defer network calls, model inference, and heavy database aggregates; surface cached values and trigger refresh off-path.

```swift
// WRONG - network call on every refresh stalls the overlay
func perform() async throws -> some IntentResult & ShowsSnippetView {
    let latest = try await api.fetchDashboard()
    return .result(view: DashboardView(data: latest))
}

// CORRECT - read cache, kick off refresh in background
func perform() async throws -> some IntentResult & ShowsSnippetView {
    Task.detached { try await api.refreshDashboardCache() }
    return .result(view: DashboardView(data: store.cachedDashboard))
}
```

## Intent-per-variant instead of a flexible intent

When actions differ only in a parameter value, don't split them into separate intents:

```swift
// WRONG
struct CreateWorkReminderIntent: AppIntent { ... }
struct CreatePersonalReminderIntent: AppIntent { ... }
struct CreateShoppingReminderIntent: AppIntent { ... }

// CORRECT
struct CreateReminderIntent: AppIntent {
    @Parameter var list: ReminderListEntity
    @Parameter var text: String
    @Parameter var dueDate: Date?
    ...
}
```

One flexible intent composes better in Shortcuts and covers more Siri phrasings. Reserve separate intents for genuinely distinct actions.

## Exposing UI-level actions as intents

Intents should represent tasks the user cares about, not UI buttons they might tap:

```swift
// WRONG - tied to a specific UI layout
struct TapCancelButtonIntent: AppIntent { ... }

// CORRECT - represents the real task
struct DiscardDraftIntent: AppIntent { ... }
```

UI-element intents break the moment you redesign the screen; task-level intents survive refactors and translate cleanly into Siri phrases and Shortcuts actions.

## Missing `perform()` breaks Spotlight on Mac

On macOS, an intent with no `perform()` (e.g., `URLRepresentableIntent` that relies purely on URL routing) does not surface in Spotlight search results - the Mac's search indexer skips intents that can't be directly invoked.

If macOS Spotlight reachability matters, implement a `perform()` that calls the same navigator your universal-link handler does. On iOS, leaving `perform()` absent is fine.

## App-launching intents hidden from Spotlight

Only intents that can complete *without* launching your app are eligible to appear in Spotlight's shortcut suggestions. Intents with `openAppWhenRun = true` (or an equivalent foreground continuation) won't show there.

Design a mix: lightweight read-only intents (get count, show status) that Spotlight can surface; deeper mutation intents that open the app for full interaction. Don't expect Spotlight to be the primary discovery path for app-opening actions.

## Parameterized shortcut phrases shown before first launch

Shortcut phrases that interpolate entity parameters (`"Open \(\.$folder) in \(.applicationName)"`) don't appear in Spotlight or the Shortcuts home until the app has launched at least once and `updateAppShortcutParameters()` has populated the entity list.

Include at least one non-parameterized phrase for every App Shortcut so users see and can invoke it immediately on install:

```swift
AppShortcut(
    intent: OpenFolderIntent(),
    phrases: [
        "Open folder in \(.applicationName)",                  // works on first launch
        "Open \(\.$target) in \(.applicationName)"             // available after populate
    ],
    shortTitle: "Open Folder",
    systemImageName: "folder"
)
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

## Resolving thousands of entities when you only need their ids

A `[MyEntity]` parameter forces the system to fully resolve every entity (run the query, populate all properties) before `perform()` runs. For a parameter holding hundreds or thousands of entities where your code only needs the **ids**, that's a large, pointless cost. Use `EntityCollection` (iOS 27+).

```swift
// WRONG - the system resolves 1,000 PhotoEntity instances before perform() even starts
struct TagPhotosIntent: AppIntent {
    @Parameter var photos: [PhotoEntity]
    func perform() async throws -> some IntentResult {
        try await store.addTag(tag, toPhotosWith: photos.map(\.id))   // only needed the ids
        return .result()
    }
}

// CORRECT - identifiers only; no resolution during parameter handling
struct TagPhotosIntent: AppIntent {
    @Parameter var photos: EntityCollection<PhotoEntity>
    func perform() async throws -> some IntentResult {
        try await store.addTag(tag, toPhotosWith: photos.identifiers)
        return .result()
    }
}
```

Call `await photos.resolvedEntities()` when you genuinely need the full entities. Keep `[Entity]` for small selections where you read properties.

## `LongRunningIntent` that goes quiet (no progress)

`LongRunningIntent` keeps its extended-runtime grant only while you **report progress**. If you do the work without updating `progress`, the system assumes the task stalled and cancels the extension - the intent dies as if it never adopted the protocol.

```swift
// WRONG - no progress updates; the runtime extension gets revoked mid-upload
func perform() async throws -> some IntentResult {
    try await performBackgroundTask {
        for chunk in chunks { await upload(chunk) }   // silent - system thinks it stalled
    }
    return .result()
}

// CORRECT - report progress as the heartbeat that keeps the extension alive
func perform() async throws -> some IntentResult {
    try await performBackgroundTask {
        progress.totalUnitCount = Int64(chunks.count)
        for (i, chunk) in chunks.enumerated() {
            try Task.checkCancellation()
            await upload(chunk)
            progress.completedUnitCount = Int64(i + 1)
        }
    }
    return .result()
}
```

Relatedly: don't try to beat the 30-second limit by spawning a detached `Task` - unmanaged tasks aren't covered by the background-execution extension and get suspended when the app backgrounds. Adopt `LongRunningIntent`.

## Plain `nil` check instead of `valueState` in update intents

In an update intent, an optional parameter that's `nil` is ambiguous: "leave this field alone" or "clear it"? A `nil` check collapses both into one behavior - usually a bug where the user can never clear a field by voice.

```swift
// WRONG - can't distinguish "don't touch recurrence" from "remove recurrence"
if let recurrence { store.setRecurrence(recurrence, on: event.id) }
// else: do nothing - so "make this not repeat" silently does nothing

// CORRECT - valueState distinguishes the three cases
switch $recurrence.valueState {
case .set(let rule?): store.setRecurrence(rule, on: event.id)   // new value
case .set(nil):       store.clearRecurrence(on: event.id)        // explicitly cleared
case .unset:          break                                      // not in the request
}
```

## Two processes writing the same store (missing `allowedExecutionTargets`)

When intents live in a shared package linked by both the app and a widget extension, the system may run a write-intent in *either* process. If only one process is supposed to own writes (e.g. the widget has read-only access to the store), letting the intent run in the widget extension causes conflicts.

```swift
// WRONG - "favorite" can run in the widget extension and write a store it shouldn't
struct FavoritePhotoIntent: AppIntent {
    @Parameter var photo: PhotoEntity
    func perform() async throws -> some IntentResult { try await library.toggleFavorite(photo.id); return .result() }
}

// CORRECT - pin write-intents to the process that owns the store (iOS 27+)
struct FavoritePhotoIntent: AppIntent {
    static var allowedExecutionTargets: IntentExecutionTargets { [.app] }
    @Parameter var photo: PhotoEntity
    func perform() async throws -> some IntentResult { try await library.toggleFavorite(photo.id); return .result() }
}
```

## `TransientAppEntity` where a persistent identifier is required

Several iOS 27 integrations key off `EntityIdentifier`: on-screen view annotations and entity annotations on User Notifications, Now Playing, and AlarmKit. `TransientAppEntity` has **no** persistent identifier, so it can't be used in any of them.

```swift
// WRONG - transient entities have no EntityIdentifier to annotate with
notificationContent.appEntityIdentifier = EntityIdentifier(for: summaryTransientEntity)   // no id

// CORRECT - annotate with a real AppEntity / IndexedEntity
notificationContent.appEntityIdentifier = EntityIdentifier(for: messageEntity)
```

Use `TransientAppEntity` for computed/returned values only. Anything the system needs to *refer back to* (annotations, syncable identity, ownership) must be a real `AppEntity`.

## Over-donating interactions

Interaction donations (`IntentDonationManager`) should reflect **real, completed** user actions, and only ones taken through your app's UI (the system already donates Siri/Shortcuts runs). Donating on every render, every tap, or re-donating system-run intents floods the system, which may then **ignore your donations** entirely.

```swift
// WRONG - donates on every view appearance, not on a completed action
.onAppear { IntentDonationManager.shared.donate(intent: SendMessageIntent()) }

// CORRECT - donate once, after the user actually sends, from the UI path only
func sendFromUI(_ body: String, to contact: ContactEntity) async throws {
    let message = try await messenger.send(body, to: contact)
    var intent = SendMessageIntent(); intent.recipient = contact; intent.content = body
    IntentDonationManager.shared.donate(intent: intent, result: .resolved(value: message.entity))
}
```

Delete stale donations with `deleteDonations(matching:)` when the data is removed or the action is undone.

## Expecting `IndexedEntity` to index itself

`IndexedEntity` only declares that the type *may* be indexed by Spotlight and Siri - it writes nothing. Without an explicit `indexAppEntities(_:)` call the entity never appears in Spotlight, Siri can't resolve it by name, and nothing errors.

```swift
// WRONG - conformance alone, nothing is ever written to the index
struct RouteEntity: IndexedEntity { ... }

// CORRECT - write the index where the data changes (after save / delete / sync)
func routesDidChange() async {
    let entities = RouteStore.load().compactMap(RouteEntity.init)   // failable init skips unusable rows
    let index = CSSearchableIndex.default()
    try? await index.deleteAppEntities(ofType: RouteEntity.self)   // full resync: drop stale items first
    guard !entities.isEmpty else { return }
    try? await index.indexAppEntities(entities)
}
```

`indexAppEntities` only adds / updates, so a full resync without the `deleteAppEntities(ofType:)` step leaves renamed or deleted items searchable. For large sets prefer per-entity updates and `deleteAppEntities(identifiedBy:ofType:)` (see `spotlight.md`); on iOS 27+ also adopt `IndexedEntityQuery` so the system can ask you to rebuild the index.

## Siri AI (27.3): writing a "search" tool

Siri finds entities on its own through Spotlight and entity queries, then passes them to tools. A search tool duplicates that and competes with it.

```swift
// WRONG - a tool that takes criteria and returns entities
@AppIntent(.tool)
struct SearchRecipes {
    @Parameter(title: "Query") var query: String
    func perform() async throws -> some ReturnsValue<[RecipeEntity]> { ... }
}

// CORRECT - make the entity findable, and write tools that act on it
struct RecipeEntity: AppEntity, IndexedEntity { ... }

@AppIntent(.tool)
struct AddRecipeIngredients {
    @Parameter(title: "Recipe", description: "The recipe whose ingredients to add")
    var recipe: RecipeEntity
    ...
}
```

The same goes for a generable "search criteria" entity. Read-only tools that return the app's data (a plan, a list, match history) are fine and useful.

## Siri AI (27.3): `.tool` with no usable metadata

Adding the trait without fixing the text makes the tool invisible to the model, or gets it called for the wrong requests.

```swift
// WRONG
@AppIntent(.tool)
struct Run {
    static let title: LocalizedStringResource = "Run"
    @Parameter(title: "Value") var value: String
    ...
}

// CORRECT
@AppIntent(.tool)
struct PlanMeal {
    static let title: LocalizedStringResource = "Plan Meal"
    static let description = IntentDescription(
        "Puts a recipe on the meal plan for breakfast, lunch, or dinner on a specific day.",
        searchKeywords: ["schedule", "menu", "cook"]
    )
    @Parameter(title: "Date", description: "The day to plan the meal for, in the person's local time zone")
    var date: Date
    ...
}
```

There is no model-only description or tool name to add. Do not invent a `toolDefinition`, `toolDescription`, or similar property.

## Siri AI (27.3): `.tool` on a primary-domain schema intent

```swift
// WRONG - build error: the schema is in a primary domain
@AppIntent(.tool, schema: .messages.sendMessage)
struct SendMessageIntent { ... }

// CORRECT - the schema already makes it available to Siri
@AppIntent(schema: .messages.sendMessage)
struct SendMessageIntent { ... }
```

`.tool` combines only with Shortcuts-only schema domains (`.journal.createEntry`, `.books.openBook`, ...). Writing `: ToolIntent` inline or in an extension produces the same error.

## Siri AI (27.3): a tool that depends on UI state

A tool cannot be enabled or disabled, and Siri can launch the process with no UI. A tool that reads the "currently open document" from a view model, or assumes a screen is showing, fails for most requests.

```swift
// WRONG - only works while the editor screen is open
func perform() async throws -> some IntentResult {
    guard let doc = EditorViewModel.shared.currentDocument else { return .result() }   // silently does nothing
    ...
}

// CORRECT - take the entity as a parameter, write to the persisted store
@Parameter(title: "Document", description: "The document to export")
var document: DocumentEntity
```

Onscreen context ("this document") still works: annotate the view and Siri fills the parameter (see `onscreen-awareness.md`). Changes must reach the persisted store, because the process can exit right after `perform()`.

## Siri AI (27.3): returning nothing and throwing generic errors

The model reads both. A bare `.result()` gives it nothing to describe or chain, and a plain `Error` carries no message.

```swift
// WRONG
func perform() async throws -> some IntentResult {
    guard result.scores.count > 1 else { throw NSError(domain: "app", code: 1) }
    ...
    return .result()
}

// CORRECT - an actionable message, and the updated entity as the value
struct GameError: Error, CustomLocalizedStringResourceConvertible {
    var localizedStringResource: LocalizedStringResource
}

func perform() async throws -> some ReturnsValue<GameEntity> {
    guard result.scores.count > 1 else {
        throw GameError(localizedStringResource: "A match needs at least two players. Ask who else played.")
    }
    ...
    return .result(value: GameEntity(game.model))   // rebuilt after the mutation, not the stale parameter
}
```

## Siri AI (27.3): trusting model-written values

For a generable parameter no query runs, and for any parameter the value came from a language model. Validate before writing.

```swift
// WRONG - saves whatever arrived
try store.createRecipe(name: draft.name, minutes: draft.minutes, ingredients: draft.ingredients)

// CORRECT
guard !draft.ingredients.isEmpty else { throw LarderError.emptyIngredients }
guard draft.minutes > 0 else { throw LarderError.invalidCookTime }
```

## Siri AI (27.3): making a stored entity generable

`.generable` means "the model invents this value". On an app-owned entity that produces objects that do not exist in the app. Putting an app-owned entity *inside* a generable entity is a build error, and while it stands no App Intents metadata is exported for the target at all.

```swift
// WRONG - 'course' is app-owned; the model cannot author it
@AppEntity(.generable)
struct StudyPlan {
    @Property(title: "Course") var course: CourseEntity
    @Property(title: "Sessions", description: "The sessions to do each day, in order") var sessions: [StudySession]
}

// CORRECT - the stored entity is its own intent parameter
@AppIntent(.tool)
struct SetStudyPlan {
    @Parameter(title: "Plan", description: "The new daily plan, as practice sessions") var plan: StudyPlan
    @Parameter(title: "Course", description: "The course whose plan to replace") var course: CourseEntity
    ...
}
```

Do not make `CourseEntity` generable to silence the error.

## Siri AI (27.3): `@available` on an intent that already ships

```swift
// WRONG - removes the intent from Shortcuts, Spotlight, and App Shortcuts below 27.3
@available(iOS 27.3, *)
@AppIntent(.tool)
struct AddToCart { ... }          // this intent shipped in iOS 17

// CORRECT - leave the type alone, gate only the conformance
struct AddToCart: AppIntent { ... }

@available(iOS 27.3, macOS 27.3, visionOS 27.3, *)
extension AddToCart: ToolIntent {}
```

Do not drop the feature for an older deployment target either. `ToolIntent` and `GenerableEntity` do not exist on watchOS or tvOS, so wrap the extension in `#if !os(watchOS) && !os(tvOS)` for targets that build there. The 27.3 property arguments (`description:`, `visibility:`) cannot be gated on a stored property: on an entity that ships earlier, see the fallback in `siri-ai-entities.md`.

## Siri AI (27.3): deleting App Shortcuts after adding tools

Adding any tool intent makes the app's App Shortcuts unavailable **to Siri**, which can look like they stopped being useful. They still drive Spotlight, the Action button, and the Shortcuts app. Keep the `AppShortcutsProvider`, and check the tools cover what the phrases did by voice.

## Siri AI (27.3): app skill mistakes

A `Skills.skillsbundle` fails quietly. The build checks format only.

```markdown
<!-- WRONG -->
---
name: Meal_Planning                         <!-- must be kebab-case and match the directory -->
description: Plan meals: breakfast or dinner   <!-- unquoted ": " is a YAML error -->
license: MIT                                <!-- unsupported field -->
---
Call PlanMealIntent with the recipe.        <!-- Swift type name -->
See references/tools.md for details.        <!-- other files do not ship -->

<!-- CORRECT -->
---
name: meal-planning
description: Put a recipe on the meal plan for a specific day and meal. Use when the user asks to plan, schedule, or put something on the menu. Not for adding single items to a shopping list.
---
Before planning, use the Get Meal Plan tool for that day. If a different recipe is
already planned for that meal, tell the user and ask before replacing it.
```

- **One error disables every skill in the app**, not only the broken one.
- A bundle missing from Copy Bundle Resources is not compiled and Xcode says nothing. Check for `Metadata.appintents/Skills.skillsc` in the built app.
- Xcode's Skills Bundle template leaves a `sample-skill` placeholder. Untouched, it builds and ships as a real skill. Renaming the folder without updating `name:` (or the reverse) is a build error.
- A skill that mostly explains one tool's arguments means the tool is badly shaped. Fix the tool.
- A skill is guidance, not enforcement. Validation and confirmation belong in `perform()`.
- Writing the app's `SKILL.md` as a coding-agent skill, or following its instructions yourself. Siri's model reads it at runtime.

## Siri AI (27.3): treating a clean analyzer run as proof

`sirimetadataanalyzer` checks the shape of built metadata. It reads build output, so after a failed build it reports the previous build's findings. It does not catch overlapping tools, vague skill descriptions, or whether Siri actually picks the tool. Follow it with `SiriEvaluations` scenarios and on-device requests, and run those more than once: Siri's answers vary between runs (see `siri-evaluations.md`).
