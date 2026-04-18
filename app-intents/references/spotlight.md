# Spotlight indexing

Spotlight surfaces app entities in the system search UI and routes taps back to an `OpenIntent`. Setup is three small steps.

## 1. Conform to `IndexedEntity`

`IndexedEntity` is a subprotocol of `AppEntity` that adds Spotlight behavior. No extra required members - the entity's `displayRepresentation` is used to build the Spotlight card.

```swift
import AppIntents
import CoreSpotlight

struct ArticleEntity: IndexedEntity {
    var id: UUID
    var title: String
    var summary: String
    var publishedAt: Date
    var thumbnailURL: URL?

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    static let defaultQuery = ArticleEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)", image: .init(systemName: "doc.text"))
    }
}
```

## 2. Send entities to `CSSearchableIndex`

Indexing is async and happens off the UI. Hand your entities to `CSSearchableIndex.default().indexAppEntities(_:)`.

```swift
import CoreSpotlight

try await CSSearchableIndex.default().indexAppEntities(entities)
```

That's the whole API. You pass `IndexedEntity` instances; the system extracts the display representation, attribute set, and id.

## 3. Decide when to index

There's no one right answer - choose based on how changeable your content is.

### Full reindex on view appearance

Simple, and fine for small datasets (a few hundred entries):

```swift
struct ArticleList: View {
    @Query private var articles: [Article]

    var body: some View {
        List(articles) { article in
            NavigationLink(article.title, value: article)
        }
        .task(indexAll)
    }

    @Sendable func indexAll() async {
        try? await CSSearchableIndex.default().indexAppEntities(articles.map(\.entity))
    }
}
```

Pros: correctness-first, no change tracking.
Cons: wasted work for large datasets or rarely-changing content.

### Per-entity reindex on change

Index one item when a specific field changes. Use with a debounced task so you don't reindex on every keystroke:

```swift
struct ArticleEditor: View {
    @Bindable var article: Article
    @State private var indexingTask: Task<Void, Error>?

    var body: some View {
        Form {
            TextField("Title", text: $article.title)
            TextField("Body", text: $article.body, axis: .vertical)
        }
        .onChange(of: article.title,  scheduleIndex)
        .onChange(of: article.body,   scheduleIndex)
    }

    func scheduleIndex() {
        indexingTask?.cancel()
        indexingTask = Task {
            try await Task.sleep(for: .seconds(1))
            try await CSSearchableIndex.default().indexAppEntities([article.entity])
        }
    }
}
```

Cancelling the previous task before sleeping means typing fast produces one index call, not twenty.

### Bulk index at startup

If your content is effectively static (curated catalogue, preset library), index everything once in `App.init()` or on first launch and don't bother with per-change tracking.

## Enriching the Spotlight card with `attributeSet`

Override `var attributeSet: CSSearchableItemAttributeSet` on your `IndexedEntity` to add content beyond the default display representation. Start from `defaultAttributeSet`, don't build one from scratch:

```swift
import CoreSpotlight

struct ArticleEntity: IndexedEntity {
    var id: UUID
    var title: String
    var summary: String
    var publishedAt: Date
    var thumbnailURL: URL?

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Article"
    static let defaultQuery = ArticleEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title)", image: .init(systemName: "doc.text"))
    }

    var attributeSet: CSSearchableItemAttributeSet {
        let set = defaultAttributeSet
        set.contentDescription = summary
        set.addedDate = publishedAt
        set.thumbnailURL = thumbnailURL
        set.keywords = ["article", "reading"]
        return set
    }
}
```

Useful `CSSearchableItemAttributeSet` fields (there are many more - autocomplete is your friend):

- `contentDescription` - body text Spotlight searches over.
- `addedDate`, `contentCreationDate`, `contentModificationDate`, `dueDate`, `startDate`, `completionDate`.
- `thumbnailURL`, `thumbnailData`.
- `keywords` - array of strings weighted higher than body text.
- `authors`, `contributors` (`CSPerson` array).
- `artist`, `album` (for audio).
- `latitude`, `longitude`, `namedLocation` (for geotagged content).

The system's ranking algorithm is opaque; more signal generally helps. Don't abuse semantically-loaded fields (`dueDate`, `startDate`) for unrelated data - Siri may interpret them literally ("what's due soon?" surfacing unrelated items).

## Tap-to-open

Tapping a Spotlight result lands in your app via a matching `OpenIntent` (see `open-and-snippet-intents.md`). The match is: result entity type → `OpenIntent` whose `target` parameter is that entity type. No extra registration.

## Cleaning up

On app uninstall the system removes your index automatically. If an item is deleted in-app:

```swift
try await CSSearchableIndex.default().deleteAppEntities(
    identifiedBy: [deletedID],
    ofType: ArticleEntity.self
)
```

## Debugging

In Simulator's Developer menu there are two toggles worth knowing:

- **Display Recent Shortcuts** - shows cached shortcut suggestions, confirms they are reaching the system.
- **Display Donations on Lock Screen** - shows intent donations, helpful when wiring up predictive shortcuts.

Reliability of Spotlight indexing in Simulator is noticeably worse than on device; if search "doesn't find anything", try the device before assuming a code bug.
