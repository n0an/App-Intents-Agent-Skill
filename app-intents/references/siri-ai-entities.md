# Siri AI: entities the model reads and writes (27.3)

Three entity features arrived with the Siri AI Developer Preview:

- **Property descriptions, visibility, and authentication** - what the model can read on any entity.
- **Generable entities** - plain values the model writes itself from the conversation.
- **Card snippets** - a custom SwiftUI card the system shows for an entity in Siri responses.

Tool intents and the preview's limits are in `siri-ai-tools.md`.

## Describing properties for the model

This applies to **every entity Siri's model sees**, not only generable ones: entities passed as tool parameters, entities returned from tools, and entities found in search. For each one the model reads:

- its `displayRepresentation`, to tell results apart, pick one, or ask which one the person meant
- each `@Property`, `@ComputedProperty`, and `@DeferredProperty`, with its title and `description`

An entity with no exposed properties gives the model only its display representation.

```swift
struct StoreEntity: AppEntity, IndexedEntity {
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Store"
    static let defaultQuery = StoreQuery()

    var id: String

    @Property(title: "Name", description: "The store's name, such as Market Street",
              indexingKey: \.displayName)
    var name: String

    @Property(title: "City", description: "The city the store is in")
    var city: String

    @Property(title: "Closing Time", description: "When the store closes today, in its local time zone")
    var closesAt: Date

    @Property(title: "Tax ID", visibility: .internal)
    var taxID: String

    @ComputedProperty(title: "Open Now", description: "Whether the store is open at this moment")
    var isOpen: Bool { hours.contains(.now) }

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)", subtitle: "\(city)")
    }
}
```

### Arguments

`@Property`, `@ComputedProperty`, and `@DeferredProperty` take the same optional labeled arguments, in this order:

| Argument | Use |
|---|---|
| `title:` | A word or short phrase naming the property. Generated from the property name when omitted. |
| `description:` | What the property holds and how to interpret it. The model reads it. |
| `authenticationPolicy:` | The authentication required before the system reads the property. `nil` applies the entity's policy. |
| `visibility:` | `.default` or `.internal` (`IntentVisibility`). |
| `indexingKey:` / `customIndexingKey:` | The Spotlight attribute the property maps to (iOS, macOS, visionOS only). |

`description:`, `authenticationPolicy:`, and `visibility:` are 27.3 on every platform.

### What to expose and how to write it

Expose what tells similar entities apart (a store's city), what people ask about (closing time, order status), and what a later tool step needs (a flight's departure time). Leave out bookkeeping such as row ids and sync tokens.

- The title names the property: "Closing Time", not "Time".
- The description says what the value holds and how to read it: unit, time zone, format, with an example when the format matters. Do not just repeat the title.
- For a property the model **writes**, the description has to be precise enough to produce a correct value: "IATA code of the departure airport, such as SFO", not "Origin".
- Use an `AppEnum` for a fixed set of values.

### Sensitive properties

**Authentication.** Entities need no authentication by default. Set a policy for the whole entity with a static `authenticationPolicy`, or per property:

```swift
struct BankAccount: AppEntity {
    static let authenticationPolicy: IntentAuthenticationPolicy = .alwaysAllowed

    @Property(title: "Balance",
              description: "The current balance, including pending transactions.",
              authenticationPolicy: .requiresLocalDeviceAuthentication)
    var balance: IntentCurrencyAmount

    @Property(title: "Routing Number", visibility: .internal)
    var routingNumber: String
}
```

The policies are `.alwaysAllowed`, `.requiresAuthentication`, and `.requiresLocalDeviceAuthentication`. A property-level policy applies only to responses Siri gives. If an **intent returns** sensitive information as its result, set `authenticationPolicy` on the intent.

**Visibility.** `visibility: .internal` keeps a property available to the app and its extensions (widgets, controls) while Siri, Shortcuts, and the model never see it. **Do not combine `.internal` with `indexingKey:` or `customIndexingKey:`**: indexing makes the property discoverable through Spotlight anyway.

### Entities that also ship before 27.3

Swift cannot mark a stored property as available only on newer releases. On an entity that is not itself `@available(... 27.3, ...)`, the new arguments fail to build when the deployment target is earlier:

```
error: 'init(title:description:authenticationPolicy:visibility:)' is only available in iOS 27.3 or newer
```

Until the deployment target reaches 27.3: make the title carry the meaning ("Closing Time (Local)"), spell out formats in the tool parameter's description, and **leave out any sensitive property you cannot hide with `.internal` yet**. A new entity marked `@available(iOS 27.3, macOS 27.3, visionOS 27.3, *)` can use the arguments at any deployment target.

## Generable entities: values the model writes

For an app-owned entity, the model has to find an existing value through Spotlight or a query. Some values are not stored anywhere: a draft, a score sheet, a delivery window, a study plan. Mark those `@AppEntity(.generable)` and the model writes the value directly from what the person said. `perform()` receives it and **no query runs**.

```swift
@AppEntity(.generable)
struct PlayerScore {
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Player Score"

    @Property(title: "Player", description: "The player's first name, such as Sophie")
    var player: String

    @Property(title: "Points", description: "How many points this player finished with, such as 10")
    var points: Int

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(player): \(points)")
    }

    init() {}
}

@AppEntity(.generable)
struct MatchResult {
    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Match Result"

    @Property(title: "Scores", description: "One score for each person who played")
    var scores: [PlayerScore]

    @Property(title: "Date", description: "When the match was played. Leave empty if it was played today.")
    var date: Date?

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(scores.count) players")
    }

    init() {}
}

@AppIntent(.tool)
struct LogMatch {
    static let title: LocalizedStringResource = "Log Match"
    static let description = IntentDescription(
        "Records the result of a board game the person played, saving each player's score."
    )

    @Parameter(title: "Game", description: "The game that was played")
    var game: GameEntity            // app-owned: Siri finds it

    @Parameter(title: "Result",
               description: "Everyone who played and how many points they scored. For example, Sophie with 10 points and Charlotte with 8.")
    var result: MatchResult         // generable: Siri writes it

    @MainActor
    func perform() async throws -> some ReturnsValue<GameEntity> {
        guard result.scores.count > 1 else { throw GameError.needsTwoPlayers }
        // ... save, then return the updated entity
    }
}
```

"I played Azul. Sophie got 72 and Charlotte got 64" makes Siri find Azul through `GameEntity`'s query and write the `MatchResult` itself.

The macro adds a `GenerableEntity` conformance, which refines `TransientAppEntity`, so `: TransientAppEntity` is optional. `struct MatchResult: TransientAppEntity, GenerableEntity` is equivalent.

### When to use it

Use `.generable` when all of these hold:

- The type is a transient value, not an object the app stores and looks up.
- The model can write a complete value from the request without the app looking anything up.
- The value is something a tool **acts on**: a plan to set, a draft to send, a result to record.

Do not use it for objects the app owns (products, contacts, documents) or for search criteria. A "search" tool taking generable criteria duplicates what Siri does on its own.

Prefer **one generable entity over a long list of related parameters**: the values are documented once, grouped by meaning, reusable across intents, and the same type can be one tool's output and another tool's input. Keep a separate parameter for a value that stands on its own (a quantity, a yes/no choice) and for app-owned entities.

### What it can contain

Every property must be one of these, or an array of one:

- a value type `@Property` supports: `String`, `Int`, `Double`, `Bool`, `Date`, `URL`, `Measurement`
- an `AppEnum`
- another generable entity

Optionals are allowed. Use them for values the person might not give, and say in the description what leaving them out means. Do not invent sentinels like "0 means no exam".

An app-owned entity inside a generable entity is a **build error**:

```
error: 'StudyPlan' is a 'GenerableEntity', but property 'course' is 'CourseEntity',
which the model cannot author.
```

While that error stands, **no App Intents metadata is exported for the target, so none of the app's intents reach Siri.** Fix it by making the app-owned entity a separate parameter on the intent. Do not make `CourseEntity` generable to silence the error.

### Rules

- **Every property needs a description precise enough to write from.** State the format or unit and give an example. Describe the overall shape again in the tool parameter's description.
- **Validate in `perform()`.** No query checked the value. Reject an empty or impossible value with an error that says what to ask for.
- **Deployment target below 27.3:** leave an existing transient entity as is and add `extension StudyPlan: GenerableEntity {}` behind `@available(iOS 27.3, macOS 27.3, visionOS 27.3, *)`. Wrap it in `#if !os(watchOS) && !os(tvOS)` for targets that build there.
- **Schema entities:** `@AppEntity(.generable, schema:)` exists, but no current entity schema is transient. Do not add `.generable` to an `@AppEntity(schema:)` entity.
- **Moving a property out of the entity into a new intent parameter changes the intent's parameters**, which affects saved shortcuts. Flag it, and make the new parameter optional when possible.
- **Testing:** build one with `definitions.transientEntities["MatchResult"].makeEntity(...)` in `AppIntentsTesting` (see `testing-intents.md`).

## Card snippets: a custom card for an entity

By default Siri shows an entity through its `DisplayRepresentation`. Conform the entity to `SnippetRepresentableEntity` and return a `CardSnippet` to show your own SwiftUI view instead, in Siri responses and conversation transcripts. Card snippets are 27.3 on every platform.

```swift
extension GameEntity: SnippetRepresentableEntity {
    static var snippet: some EntitySnippetContent<GameEntity> {
        CardSnippet(supportedFamilies: [.standard]) { game, context in
            VStack {
                Text(game.emoji).font(.system(size: 80))
                Text(game.name).font(.title2.bold())
                Text("Matches played: \(game.timesPlayed)")
                Button(game.isFavorite ? "Remove Favorite" : "Add Favorite",
                       intent: SetFavorite(game: game, isFavorite: !game.isFavorite))
            }
        }
    }
}
```

This is a different mechanism from `SnippetIntent` (see `open-and-snippet-intents.md`): a snippet intent is something **an intent returns**, a card snippet is how **an entity looks** wherever the system shows it. Both work with tools.

- **The card is rendered ahead of time into a static archive**, and can appear while the app is not running. Build it from content that is ready when the view is created: text, images, values on the entity. `AsyncImage`, a later `@State` change, or asynchronously produced data arrives too late and shows stale or missing content. Prepare data synchronously before returning the view.
- **Controls run intents, not closures.** Use `Button(intent:)` or `Toggle(isOn:intent:)`. After the intent runs, the system renders a new version of the card. The intent should return the updated entity. Add `.contentTransition(...)` to animate between versions.
- **The intent needs a memberwise init plus an explicit `init() {}`.** A custom initializer removes the implicit empty one that `AppIntent` requires.
- **Families.** `supportedFamilies` lists what the card adapts to, and `context.family` says which one to render. For a family the card does not support, the system falls back to the display representation, so keep that good. `.standard` is the full-size card. `.compact` (little room, such as Apple Watch) is documented and used in Apple's UnicornChat sample, but was reported missing from the first 27.3 beta SDK. If `.compact` does not compile, ship `.standard` only.
- **Styling.** Set a custom background with `containerBackground(_:for: .cardSnippet)`. Call `.contentMarginsDisabled()` on the snippet to draw edge to edge and apply padding yourself.
- **Tapping the card opens the app.** To open a specific item, add an `OpenIntent` adopting `@AppIntent(schema: .system.open)` for that entity type (see `assistant-schemas.md`). The system runs it instead of a plain launch.
- **Only include content suited to a system surface.** The card is shown outside the app.
