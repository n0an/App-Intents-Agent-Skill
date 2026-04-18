# Assistant schemas (Apple Intelligence)

Regular `AppEntity` / `AppIntent` let Siri understand *your* concepts. **Assistant schemas** let Siri understand your concepts *in terms of shared cross-app categories* - journal entries, mail messages, browser bookmarks, photo assets, spreadsheet cells. This is the layer that lets Apple Intelligence compose across apps ("take the last three photos I shared in Messages and draft a journal entry about them").

Requires iOS 18.2+ for most schemas; some journaling, mail, and spreadsheet schemas require 18.4+. Availability of the *features that consume* these schemas is rolling out slowly - adopting the schema makes your data eligible, it does not guarantee it will be used.

## Adopting a schema

Use `@AssistantEntity(schema:)` on an entity type:

```swift
import AppIntents
import CoreLocation

@AssistantEntity(schema: .journal.entry)
struct JournalEntryEntity: IndexedEntity {
    var id: UUID

    // These names and types are prescribed by the schema - you cannot rename them
    var title: String?
    var message: AttributedString?
    var mediaItems: [IntentFile]
    var entryDate: Date?
    var location: CLPlacemark?

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Journal entry"
    static let defaultQuery = JournalEntryEntityQuery()

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(title ?? "Untitled")")
    }
}
```

Key consequences:

- **Prescribed property names and types.** `message` must be `AttributedString?`, not `String`. `entryDate` is `Date?`, optional. `location` is `CLPlacemark?`. The macro validates these and emits opaque compiler errors if you deviate.
- **More optionality than you'd expect.** Many schema-required fields are optional even when your real data always has them. Provide sensible fallbacks in initializers or display representations.
- **Import `CoreLocation`** for any schema that carries location.

### A convenience initializer

Because the macro rewrites stored properties to private `_entityProperty`-wrapped versions, the synthesized init is ugly. Provide your own `init` for internal use:

```swift
init(id: UUID, title: String?, body: String, createdAt: Date?, location: CLPlacemark? = nil) {
    self.id = id
    self.title = title
    self.message = AttributedString(body)
    self.mediaItems = []
    self.entryDate = createdAt
    self.location = location
}
```

## Adopting intent schemas

Symmetric to entities. Use `@AssistantIntent(schema:)`:

```swift
@AssistantIntent(schema: .journal.createEntry)
struct CreateJournalEntryIntent: AppIntent {
    static let title: LocalizedStringResource = "Create journal entry"

    @Parameter(title: "Title") var title: String?
    @Parameter(title: "Body")  var body: String
    @Parameter(title: "Date")  var date: Date?

    @Dependency var store: DataStore

    @MainActor
    func perform() async throws -> some IntentResult & ReturnsValue<JournalEntryEntity> {
        let entry = try store.createJournalEntry(title: title, body: body, createdAt: date)
        return .result(value: entry.entity)
    }
}
```

The schema dictates:

- Which parameters are required/optional and what their names must be.
- What the intent must return (often an entity of the matching schema type, e.g., a created journal entry).
- Any domain-specific behavior (search, open, append, etc.).

## Available schemas (representative)

Categories (not exhaustive):

- `.books.*` - book, audiobook, library
- `.browser.*` - bookmark, window, tab
- `.files.*` - file
- `.journal.*` - entry, entryLocation, createEntry, updateEntry, searchEntries
- `.mail.*` - message, account, composition
- `.photos.*` - asset, album, person
- `.presentations.*` - slide deck, slide
- `.spreadsheets.*` - sheet, cell, range, template
- `.systemSearch.*` - search query, search suggestion

Xcode 16+ ships code snippets. Type `journal` or `mail` into the editor and the completion menu offers skeletons for each schema's intents and entities; this is by far the fastest way to adopt a schema correctly given how strict the macro is.

## What adoption gets you

In principle:

- Your data participates in system searches ("find the note where I mentioned Berlin").
- Cross-app composition ("take this journal entry and share it in Mail").
- Siri understands domain verbs ("append this to my last journal entry") without you wiring each phrase.

In practice, as of early 2026 the feature surfaces consuming assistant schemas are rolling out unevenly. Adopt where the schema closely matches your domain, but treat actual behavior as a moving target - verify on a current iOS release rather than on the WWDC24 announcement.

## When NOT to adopt

- Your data is genuinely novel and doesn't fit any schema. Don't bend your model to match; stick with plain `AppEntity`.
- The schema forces losing fidelity (e.g., you have rich markdown and the schema insists on `AttributedString` which loses your extensions). Weigh the integration benefit against the modeling cost.
- You're still on iOS 17 or earlier - assistant schemas require 18.2+.

## Versioning caveat

Xcode embeds the schema version ("journal entry 1.0.0") in the macro-expanded code. Schemas are expected to evolve - Apple chose macros precisely so they can add fields without breaking existing adopters. Treat schema adoption as you would any platform API: rev your minimum deployment target when you need newer schema features, and expect some churn at first.

## Co-existence

You can have both a plain `AppEntity` and a schema-adopted entity in the same app. Some flows want the raw type; some want the schema-shaped one. Map between them where needed rather than trying to make one entity wear both hats.
