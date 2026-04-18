# Parameters

Intent parameters are declared with `@Parameter` and are automatically resolved by the system - the user is prompted, taps a picker, or chains a value from another action in Shortcuts.

## Declaring a parameter

```swift
struct AppendToNoteIntent: AppIntent {
    static let title: LocalizedStringResource = "Append to note"

    @Parameter(title: "Text")
    var newText: String

    func perform() async throws -> some IntentResult & ProvidesDialog {
        // newText is guaranteed non-nil here
        ...
    }
}
```

By the time `perform()` runs, every non-optional `@Parameter` is filled. If the user didn't provide one, the system asked them - via Siri voice, a text field, or a picker - before calling `perform()`.

## Supported parameter types

Primitive:

- `String`, `Int`, `Double`, `Bool`, `Date`, `URL`, `Measurement`
- `IntentFile`, `IntentItem`, `IntentEnum`
- `Decimal`, `Data`
- Optional or array of any of the above

Domain types:

- Any `AppEntity` (single or `[MyEntity]`)
- Any `AppEnum`

## Parameter options

`@Parameter` accepts options that shape the user-facing prompt:

```swift
@Parameter(
    title: "Tag",
    description: "Tag applied to the bookmark.",
    requestValueDialog: "Which tag should I use?",
    default: "reading"
)
var tag: String

@Parameter(
    title: "Priority",
    default: .normal,
    requestDisambiguationDialog: "Which priority?"
)
var priority: PriorityLevel  // an AppEnum

@Parameter(title: "Attachment", supportedTypeIdentifiers: ["public.image", "public.pdf"])
var attachment: IntentFile
```

## Entity parameters

Any `AppEntity` can be a parameter. The system uses the entity's `defaultQuery` to populate a picker and to resolve user speech to a specific entity:

```swift
struct OpenBookmarkIntent: AppIntent {
    static let title: LocalizedStringResource = "Open bookmark"

    @Parameter(title: "Bookmark")
    var bookmark: BookmarkEntity

    func perform() async throws -> some IntentResult {
        ...
    }
}
```

Given `BookmarkEntity.defaultQuery`, Shortcuts will show a "Bookmark" field with a picker populated from the query. In Siri, the user says "open bookmark Weather" and the system resolves "Weather" against the query's `EntityStringQuery` (if conformant) or falls back to a disambiguation dialog.

## `@AppEnum`

For fixed-set parameters, declare an `AppEnum`:

```swift
enum PriorityLevel: String, AppEnum {
    case low, normal, high

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Priority"
    static let caseDisplayRepresentations: [PriorityLevel: DisplayRepresentation] = [
        .low: "Low",
        .normal: "Normal",
        .high: "High"
    ]
}
```

Enums show as nice pickers in Shortcuts, are speakable by Siri, and are chainable in automation.

## Parameter summaries

Give Shortcuts a one-line summary by implementing `parameterSummary`:

```swift
struct MoveArticleIntent: AppIntent {
    static let title: LocalizedStringResource = "Move article"

    @Parameter(title: "Article") var article: ArticleEntity
    @Parameter(title: "Folder") var folder: FolderEntity

    static var parameterSummary: some ParameterSummary {
        Summary("Move \(\.$article) to \(\.$folder)")
    }

    func perform() async throws -> some IntentResult { .result() }
}
```

This renders as "Move [Article] to [Folder]" in the Shortcuts editor.

For intents with many parameters, `When`/`Switch` conditions tailor the summary to the inputs:

```swift
static var parameterSummary: some ParameterSummary {
    Switch(\.$action) {
        Case(.archive) { Summary("Archive \(\.$article)") }
        Case(.delete)  { Summary("Delete \(\.$article)") }
    }
}
```

## Requesting a value mid-perform

If a parameter is optional and you need it to continue, throw a needs-value error:

```swift
@Parameter(title: "Folder") var folder: FolderEntity?

func perform() async throws -> some IntentResult {
    guard let folder else {
        throw $folder.needsValueError("Which folder should the article go in?")
    }
    ...
}
```

The system re-prompts the user, then re-invokes `perform()` with the value filled.

## Disambiguation

When a parameter has multiple plausible matches (e.g., two entities with similar names), present a disambiguation:

```swift
throw $folder.needsDisambiguationError(among: candidates, dialog: "Which folder?")
```

## `@Parameter` vs ordinary property

Only `@Parameter`-annotated properties are exposed to the system. Ordinary properties are fine for local caching inside `perform()` but are invisible to Shortcuts, Siri, and the parameter-prompt system.

## Entity-to-entity relationships

Parameters can themselves be `AppEntity` arrays for bulk operations:

```swift
@Parameter(title: "Articles") var articles: [ArticleEntity]
```

Shortcuts renders this as a multi-select list; Siri asks "which articles?" and accepts multiple names.

## Keep perform() typed

Always type the perform result to match exactly what you return - `some IntentResult & ProvidesDialog & ReturnsValue<Int>` must match the `.result(...)` call or the runtime will crash. When in doubt, return less (`some IntentResult`) and add capabilities as you wire them up.
