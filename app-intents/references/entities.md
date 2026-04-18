# App entities

`AppEntity` is how the system understands your domain objects. Entities let users pick articles, bookmarks, playlists, rooms, etc., as parameters - by voice, by tapping in Shortcuts, or by selecting in Spotlight.

## The sendability rule

`AppEntity` refines `AppValue` and requires `Sendable`. SwiftData `@Model` classes, Core Data `NSManagedObject`, and any other reference-type data model are **not** sendable. They cannot be `AppEntity`.

```swift
// WRONG - will not compile under Swift 6, produces sendability warnings earlier
@Model
class Article { ... }
extension Article: AppEntity { ... }   // conformance of 'Article' to 'Sendable' unavailable
```

The fix is to create a separate `struct` entity that **shadows** the fields you want to expose to the system, then convert between them at the query boundary.

```swift
struct ArticleEntity: AppEntity {
    var id: UUID
    var title: String
    var summary: String
    var publishedAt: Date
    var thumbnailURL: URL?

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    static let defaultQuery = ArticleEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(
            title: "\(title)",
            subtitle: "\(summary)",
            image: .init(systemName: "doc.text")
        )
    }
}
```

In the underlying model, add a computed property to produce the entity cheaply:

```swift
@Model
final class Article {
    var id = UUID()
    var title: String
    var summary: String
    var publishedAt: Date
    var thumbnailURL: URL?

    var entity: ArticleEntity {
        ArticleEntity(
            id: id,
            title: title,
            summary: summary,
            publishedAt: publishedAt,
            thumbnailURL: thumbnailURL
        )
    }

    init(...) { ... }
}
```

Loading entities now means loading models and mapping with `\.entity`:

```swift
try modelContext.fetch(descriptor).map(\.entity)
```

## Required members

An `AppEntity` must provide:

| Member | Purpose |
|---|---|
| `var id: some Hashable & Sendable` | Stable unique identifier. `UUID` works well. |
| `static let typeDisplayRepresentation: TypeDisplayRepresentation` | Type-level name ("Article"). Used in pickers and Siri. |
| `static let defaultQuery: some EntityQuery` | How the system loads entities when it needs to populate parameters. |
| `var displayRepresentation: DisplayRepresentation` | Per-instance label shown in pickers, notifications, Spotlight cards. |

`typeDisplayRepresentation` is shared across all entities of the type; `displayRepresentation` is per-instance. Do not mix them up.

## `DisplayRepresentation` anatomy

```swift
var displayRepresentation: DisplayRepresentation {
    DisplayRepresentation(
        title: "\(title)",
        subtitle: "\(summary)",
        image: .init(systemName: "doc.text")
    )
}
```

Variants:

```swift
// Minimal
DisplayRepresentation(title: "\(title)")

// Title + subtitle
DisplayRepresentation(title: "\(title)", subtitle: "\(summary)")

// With URL-backed image (thumbnail)
DisplayRepresentation(
    title: "\(title)",
    image: DisplayRepresentation.Image(url: thumbnailURL)
)

// With an SF Symbol
DisplayRepresentation(
    title: "\(title)",
    image: .init(systemName: "book.closed")
)

// With a tinted symbol
DisplayRepresentation(
    title: "\(title)",
    image: .init(systemName: "book.closed", tintColor: .systemBlue)
)
```

## Entity queries

The system cannot guess how to load your entities. You must provide a query.

### `EnumerableEntityQuery` (small, loadable sets)

Use when the whole set is small and cheap to enumerate - folders, tag lists, starter presets.

```swift
struct FolderEntityQuery: EnumerableEntityQuery {
    @Dependency var store: DataStore

    func allEntities() async throws -> [FolderEntity] {
        try await store.folderEntities()
    }

    func entities(for identifiers: [FolderEntity.ID]) async throws -> [FolderEntity] {
        try await store.folderEntities(matching: #Predicate {
            identifiers.contains($0.id)
        })
    }
}
```

`allEntities()` and `entities(for:)` are both required. `entities(for:)` is called when the system already knows the id and needs to resolve it back to an entity - common during Shortcuts re-evaluation.

### `EntityQuery` (large, searchable sets)

Use when there can be thousands of entries. Add `EntityStringQuery` to support search-by-string (Siri spoken lookup, Shortcuts "Find…").

```swift
struct ArticleEntityQuery: EntityQuery {
    @Dependency var store: DataStore

    func entities(for identifiers: [ArticleEntity.ID]) async throws -> [ArticleEntity] {
        try await store.articleEntities(matching: #Predicate {
            identifiers.contains($0.id)
        })
    }

    func suggestedEntities() async throws -> [ArticleEntity] {
        // Shown at the top of pickers; return recent or pinned items.
        try await store.articleEntities(sortBy: [.init(\.publishedAt, order: .reverse)], limit: 10)
    }
}

extension ArticleEntityQuery: EntityStringQuery {
    func entities(matching string: String) async throws -> [ArticleEntity] {
        try await store.articleEntities(matching: #Predicate { article in
            article.title.localizedStandardContains(string)
        })
    }
}
```

### `UniqueIDEntityQuery`

Convenience for the common case where you have exactly one identifier column and simple lookup by id.

## Predicate pitfall: local copy of id

When filtering by the entity's id inside `#Predicate`, copy the id to a local constant first. The macro does not reach through property paths on entity types:

```swift
// WRONG - macro cannot reach entity.id in this position
try store.articles(matching: #Predicate { $0.id == entity.id })

// CORRECT
let id = entity.id
try store.articles(matching: #Predicate { $0.id == id })
```

## Registering the default query

Entities reference their query through `defaultQuery`:

```swift
struct ArticleEntity: AppEntity {
    ...
    static let defaultQuery = ArticleEntityQuery()
    ...
}
```

Without `defaultQuery`, parameter pickers are empty and Siri cannot resolve named entities.

## Indexed entities (Spotlight)

`IndexedEntity` is a subprotocol of `AppEntity` that makes entities searchable through Spotlight. See `spotlight.md`.

```swift
struct ArticleEntity: IndexedEntity { ... }   // instead of AppEntity
```

No additional required members - the entity's `displayRepresentation` is used to build the Spotlight card automatically. Override `attributeSet` to enrich indexing.
