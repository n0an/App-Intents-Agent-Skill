# Siri AI: tool intents (27.3 Developer Preview)

With the Siri AI Developer Preview (iOS / iPadOS / macOS / visionOS 27.3, Xcode 27.3), Siri's model can call **any custom app intent** as a tool, fill its parameters from the conversation, and chain several of them. Before 27.3 Siri only reached your app through a system schema or an exact App Shortcut phrase.

There is no new SDK or entitlement. The app describes its actions and data with App Intents, and the Siri model picks a tool and fills it in:

```
Person asks Siri
  → Siri model finds entities (Spotlight, entity queries)
  → calls a tool (your AppIntent.perform())
  → reads the result or the thrown error
  → answers, shows a card, or calls the next tool
```

Related files: `siri-ai-entities.md` (property descriptions, generable entities, card snippets), `app-skills.md` (`Skills.skillsbundle`), `siri-evaluations.md` (metadata analyzer and Siri Evaluations).

## Preview status

- **Availability.** `ToolIntent`, `AppIntentTraits`, `GenerableEntity`, `AppEntityTraits` and the `@AppIntent(_:)` / `@AppEntity(_:)` trait overloads exist on iOS, iPadOS, Mac Catalyst, macOS, and visionOS 27.3. They are **unavailable on watchOS and tvOS**.
- **Distribution.** Siri AI features work in development-signed builds and for **internal** TestFlight testers only. External TestFlight and App Store builds do not get them during the preview.
- **Device.** Test Siri itself on a physical device with Apple Intelligence turned on and its models downloaded. Launch the app once after install so its intents register and its entities reach Spotlight. The intents can still be exercised in the Simulator with `AppIntentsTesting`, and the metadata analyzer needs no device.
- **Churn.** Apple states the App Intents API and Siri behavior may change during the preview. Check the SDK when something here does not compile.
- **First use.** In the 27.3 beta, the first time Siri reaches an app the person is asked to allow access to that app through Siri (observed behavior, not documented).

## Schema, tool, or App Shortcut?

```
Action matches a schema in an Apple Intelligence + Siri primary domain
(messages, mail, notes, calendar, photos, audio, ...)?
  → @AppIntent(schema:) only. Adding .tool is a build error.

Action matches a schema in a Shortcuts-only domain
(books, browser, journal, presentation, reader, spreadsheet, whiteboard, wordProcessor)?
  → @AppIntent(.tool, schema: ...)

Custom action a person would ask Siri for in one sentence?
  → @AppIntent(.tool)

Widget configuration, a helper step, a snippet intent, something that needs a specific screen?
  → not a tool
```

Apple's web documentation and the Xcode 27.3 guidance disagree on where `files` and `video` belong. Let the compiler decide: a primary-domain schema combined with `ToolIntent` fails to build.

## When Siri can call a tool

Siri offers an app's tools to the model when one of these is true:

- the request names the app ("In Larder, plan shakshuka for Friday")
- the app is in the foreground
- a Live Activity from the app is running
- an entity from the app came back from a search
- the person continues a conversation that already mentioned the app

While the app is in the background, the request has to name the app.

## Adopting `@AppIntent(.tool)`

```swift
@AppIntent(.tool)
struct AddToCart {
    static let title: LocalizedStringResource = "Add to Cart"
    static let description = IntentDescription(
        "Adds a product to the shopping cart.",
        searchKeywords: ["basket", "buy"]
    )

    @Parameter(title: "Item", description: "The product to add")
    var item: ProductEntity

    @Parameter(title: "Quantity", description: "How many to add", default: 1)
    var quantity: Int

    @Dependency private var cart: Cart

    func perform() async throws -> some ReturnsValue<CartEntity> {
        let updated = try await cart.add(item, quantity: quantity)
        return .result(value: CartEntity(updated))
    }
}
```

The macro adds a `ToolIntent` conformance, which refines `AppIntent`, so `: AppIntent` is optional. `struct AddToCart: AppIntent, ToolIntent` is equivalent. `ToolIntent` has no requirements of its own, and adding it changes nothing about how the intent behaves in Shortcuts or Spotlight.

## Metadata is the prompt

The model decides when to call a tool and how to fill it from the intent's metadata. There is **no separate tool name or model-only description**: the text Shortcuts and Spotlight show is the text Siri reads. Do not invent a `toolDefinition` or similar property.

Write for someone who has never seen the app:

- **`title`** names the action. **`description`** says what it does and what it affects: "Adds a product to the shopping cart", not "Cart action".
- **`searchKeywords`** on `IntentDescription` carry words people say that the title and description lack. Siri uses them to retrieve the right tool.
- **Parameter `description`** states meaning plus format, unit, range, or time zone: "The day to plan the meal for, in the person's local time zone", "Duration in minutes". Skip it only when the title says everything ("Quantity" on an `Int`).
- **Say what an empty optional means**: "The list to add to. Leave empty for Groceries."
- **Tell similar tools apart in their descriptions**: "For one item that isn't tied to a recipe, use the Add Item to Shopping List tool."
- **Use an `AppEnum`** instead of a `String` for a closed set, and give a mode parameter a described case per action. If the actions take different parameters, split the tool.
- **Refer to tools by title in natural language** ("the Add to Cart tool"), never by Swift type name. How Siri names tools internally is not API.

## What Siri reads back

- **Return a value when a later step needs one.** `some ReturnsValue<OrderEntity>` lets the model pass the order to the next tool or describe it in its answer. A bare `.result()` gives it nothing to continue with.
- **Return the fresh entity after a mutation.** The entity that came in as a parameter holds the old values.
- **Read-only tools are worth having.** One "Get Meal Plan" or "Get Matches" tool that returns the data lets Siri answer many differently worded questions. Do not write a tool per question.
- **Errors are instructions to the model.** Siri reads the thrown error and decides what to do next. Only errors conforming to `CustomLocalizedStringResourceConvertible` carry a message. Say what went wrong and what would fix it:

```swift
enum LarderError: Error, CustomLocalizedStringResourceConvertible {
    case emptyIngredients
    case invalidDayCount

    var localizedStringResource: LocalizedStringResource {
        switch self {
        case .emptyIngredients: "A recipe needs at least one ingredient. Ask the person what goes in it."
        case .invalidDayCount:  "The number of days has to be between 1 and 31."
        }
    }
}
```

- **Validate every value in `perform()`.** The values come from a language model. Guard ranges and required counts, then throw a clear error.

## Which intents become tools

Make an intent a tool when a person would reasonably ask Siri to do it, in one sentence, with parameters Siri can fill from that sentence. That **includes** actions with real-world effects (buying, booking, sending, deleting). They are often the most valuable tools.

- Keep or add confirmation for anything that deletes or overwrites data. Validate first, confirm second, mutate third. A cancel throws out of `perform()`, so do not catch it:

```swift
@MainActor
func perform() async throws -> some IntentResult & ProvidesDialog {
    guard !game.model.matches.isEmpty else { throw GameError.noMatches(game.name) }
    try await requestConfirmation(dialog: "Delete the latest match of \(game.name)?")
    game.model.matches.removeLast()
    try await Game.save()
    return .result(dialog: "The latest \(game.name) match is deleted.")
}
```

- Do not confirm additions that are easy to undo. Confirm only the destructive branch (replacing an existing item, not filling an empty slot).
- State the effect in the description: "Places the order and charges the saved payment method."
- **Tools are always callable.** A tool cannot be enabled or disabled, and Siri may launch the process with no UI. Do not make an intent a tool if it only works on a particular screen or while the app is running. It works whenever called, or throws an error that says what the person has to do.
- **Changes must persist.** Siri can run the tool while the app is off screen and the process can exit afterward. Write to the persisted store, not in-memory state.
- **Keep `perform()` thin.** Call the app's existing model code. Register every `@Dependency` in `App.init()`: reading an unregistered dependency crashes.
- **Make writes idempotent where it is cheap.** People repeat requests, and a duplicate is rarely what they want. Find-or-create by a natural key instead of inserting blindly.

## Entity parameters: let Siri find them

**Do not write search or find tools.** A "search recipes" tool duplicates what Siri already does. Siri finds entities itself and passes them to tools as parameters. Write tools that **act on** an entity.

| Mechanism | Use when | Notes |
|---|---|---|
| `IndexedEntity` (preferred) | The entities can be indexed in Spotlight: documents, recipes, products, contacts | Lookups do not launch the app. Works even when the request does not name the app. See `spotlight.md`. |
| `EnumerableEntityQuery` | A small bounded set that does not fit indexing: the person's lists, courses | Every lookup runs `allEntities()` in the app. For privacy, Siri queries the app only when the person names it. |

- Every non-transient entity a tool uses needs `entities(for:)` on its query.
- **For tools, Siri does not use `suggestedEntities()`, `EntityStringQuery.entities(matching:)`, or `EntityPropertyQuery`.** Keep them for Shortcuts pickers, but do not add them to make an entity findable for a tool.
- Apple's documentation also lists parameters configured with a `DynamicOptionsProvider` as resolvable.
- **Identifiers must be stable across launches.** An `id` created with `UUID()` in seed data changes on every launch and breaks every saved reference.
- The model reads `displayRepresentation` to tell results apart or to ask which one the person meant. Give it a title that names the thing and a subtitle that distinguishes similar ones. Never show a UUID or row id.
- Describe the entity's properties. See `siri-ai-entities.md`.

## Schema intents as tools

Add `.tool` to the schema macro. It does everything `@AppIntent(schema:)` does and adds `ToolIntent`:

```swift
@AppIntent(.tool, schema: .journal.createEntry)
struct LogCookedMealIntent {
    var title: String?
    var message: AttributedString
    var mediaItems: [IntentFile]
    var entryDate: Date?
    var location: CLPlacemark?

    // The schema still dictates the shape: this must return an @AppEntity(schema: .journal.entry).
    func perform() async throws -> some ReturnsValue<JournalEntryEntity> { ... }
}
```

`.tool` does not relax the schema's contract. Combining `ToolIntent` with a primary-domain schema is a build error however it is written (macro, inline conformance, or extension):

```
error: 'ToolIntent' can't be adopted by an intent that conforms to
'.messages.sendMessage' because the schema is in a primary domain
```

Fix: remove `.tool`. The schema already makes the intent available to Siri.

## Tools and App Shortcuts

**Adding any tool intent to an app makes its App Shortcuts unavailable to Siri.** The App Shortcuts keep working in Spotlight, the Action button, and the Shortcuts app, so do not delete them. When adopting tools, make sure the tools cover what the App Shortcut phrases let people do with Siri before.

## Trait syntax

`AppIntentTraits` is a set. The macro reads the traits as written in source:

```swift
@AppIntent(.tool)
@AppIntent([.tool])
@AppIntent(.tool, schema: .journal.createEntry)
```

| Written | Result |
|---|---|
| A trait stored in a variable or computed with `union(_:)` | Build error. Write traits inline. |
| The same trait twice | Build error |
| `@AppIntent([])` with no schema | Build error |
| `@AppIntent([], schema: ...)` | Warning, no effect |

## Deployment target below 27.3

The macro takes its availability from the type. Availability is a product decision, so ask when the deployment target is earlier than 27.3.

**Existing intent that already ships.** Do not mark it `@available(iOS 27.3, ...)`: that removes it from Shortcuts, Spotlight, and App Shortcuts on earlier releases. Add the conformance in a gated extension:

```swift
struct AddToCart: AppIntent { ... }       // unchanged

@available(iOS 27.3, macOS 27.3, visionOS 27.3, *)
extension AddToCart: ToolIntent {}
```

**New intent that only needs to exist on 27.3 and later:**

```swift
@available(iOS 27.3, macOS 27.3, visionOS 27.3, *)
@AppIntent(.tool)
struct AddToCart { ... }
```

Targets that also build for watchOS or tvOS need `#if !os(watchOS) && !os(tvOS)` around the conformance, because `@available(..., *)` alone does not exclude them.

Adding `.tool` or `.generable` does not change an intent's or entity's identity. Renaming the type, renumbering an `AppEnum`, changing an `id` scheme, or adding / removing / renaming a required parameter does, and breaks saved shortcuts and donations. Flag such changes instead of making them as a side effect.

## Adoption order

1. **Find or create the intents.** Survey the model layer and main screens, list what a person would ask Siri for, and check each action against a schema first.
2. **Make each tool clear from its own metadata.** Titles, descriptions, parameter descriptions, useful results, clear errors. Make entity parameters findable.
3. **Add `.tool`.**
4. **Make plain-value parameters generable** where the model can write the value from the request (`siri-ai-entities.md`).
5. **Build and run the Siri Metadata Analyzer**, fix its findings, and repeat on every metadata change (`siri-evaluations.md`).
6. **Add app skills** only for task context and multi-tool orchestration the metadata cannot carry (`app-skills.md`).
7. **Evaluate real requests** with Siri Evaluations and on device (`siri-evaluations.md`).

The inner loop while iterating: change, build, analyze, then try Siri on the device.

## Not the same as Foundation Models

`ToolIntent` and `@AppEntity(.generable)` are for **Siri's model calling the app**. The FoundationModels framework's `Tool` protocol and `@Generable` macro are for **the app's own `LanguageModelSession`** calling the app's code. The ideas match (describe a type in natural language so a model can fill it in), the APIs do not. An app can use both.

## See also

`anti-patterns.md` lists the common mistakes: search tools, tools that depend on UI state, `.tool` on a primary-domain schema, generic errors, and metadata the model cannot use.
