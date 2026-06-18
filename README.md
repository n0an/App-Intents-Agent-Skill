<h1 align="center">App Intents Agent Skill</h1>

<p align="center">
    <img src="https://img.shields.io/badge/iOS-16+-2980b9.svg" alt="iOS 16+" />
    <img src="https://img.shields.io/badge/swift-5.9+-F05138.svg" alt="Swift 5.9+" />
    <img src="https://img.shields.io/badge/version-1.2.0-blueviolet.svg" alt="Version 1.2.0" />
    <img src="https://img.shields.io/badge/WWDC%202026-iOS%2027-FF2D55.svg" alt="Covers WWDC 2026 / iOS 27" />
    <img src="https://img.shields.io/badge/license-MIT-lightgrey.svg" alt="MIT License" />
    <a href="https://agentskills.io/home">
        <img src="https://img.shields.io/badge/Agent%20Skills-Compatible-purple.svg" alt="Agent Skills Compatible" />
    </a>
</p>

> **🍎 Updated for WWDC 2026** - now covers the iOS 27 App Intents APIs: long-running & cancellable intents, on-screen awareness, `EntityCollection`, `SyncableEntity`, and the `AppIntentsTesting` framework.

An agent skill that helps AI coding agents like Claude Code, Codex, Cursor, and Gemini write correct Swift App Intents code - exposing app actions and data to Siri, Shortcuts, Spotlight, widgets, Control Center, and Apple Intelligence.

It uses the [Agent Skills](https://agentskills.io/home) format, so it works smoothly with Claude Code, Codex, Gemini, Cursor, and more.


## What It Covers

- **Intents** - `AppIntent`, `OpenIntent`, `SnippetIntent`, `ForegroundContinuableIntent`, `DeleteIntent`, `ShowInAppSearchResultsIntent`, `TargetContentProvidingIntent`, `URLRepresentableIntent`, `ProgressReportingIntent`, `LongRunningIntent`, `CancellableIntent`, `WidgetConfigurationIntent`, `ControlConfigurationIntent`, `PredictableIntent`
- **Execution lifecycle** - the 30-second budget, `LongRunningIntent` + `performBackgroundTask` (iOS 27), `CancellableIntent` + `withIntentCancellationHandler` (iOS 26.4), background GPU access, `allowedExecutionTargets` to pin an intent to the app or an extension
- **Foreground continuation** - `ForegroundContinuableIntent` (iOS 17), `requestToContinueInForeground`, modern `supportedModes` + `continueInForeground` (iOS 26)
- **Dialog + return types** - `ProvidesDialog`, `ReturnsValue`, `ShowsSnippetView`, `ShowsSnippetIntent`, `OpensIntent`, `OpenURLIntent`, `IntentDialog(full:supporting:)`, grammar agreement
- **Metadata** - `IntentDescription(categoryName:searchKeywords:resultValueName:)`, `isDiscoverable`, `openAppWhenRun`, hard limits (10 shortcuts, 1000 phrases)
- **Parameters** - `@Parameter` (with Xcode 16 inferred titles), `@AppEnum` (with `typeDisplayName`), `DynamicOptionsProvider`, `IntentParameterDependency`, `EntityCollection` (identifiers-only at scale), native `Duration` / `PersonNameComponents`, `valueState` (set / cleared / unset for update intents), measurement options, array size per widget family, `requestValue` / `needsValueError` / `requestConfirmation` / `requestChoice`, conditional `parameterSummary` (`Switch`/`Case`/`When`/`otherwise`, widget-family conditions)
- **Entities** - `AppEntity`, `IndexedEntity`, `TransientAppEntity`, `FileEntity`, `SyncableEntity` (cross-device ids), `OwnershipProvidingEntity` (shared/public confirmation), `RelevantEntities` (proactive suggestions), shadow-struct pattern for SwiftData, `@Property`, `@ComputedProperty(indexingKey:)`, `@DeferredProperty`, synonyms, pluralized `TypeDisplayRepresentation`, thumbnails (URL/Data/system/bundled), `@UnionValue` parameters
- **Queries + Find intents** - `EntityQuery`, `EntityStringQuery`, `EnumerableEntityQuery`, `EntityPropertyQuery` (auto-generated Find intent with comparators + sort), `UniqueIDEntityQuery`, `IndexedEntityQuery` (Spotlight reindexing), `IntentValueQuery` (structured Siri search + visual intelligence)
- **Transferable + URL** - `Transferable` conformance for sharing, `IntentValueRepresentation` / `ValueRepresentation` for structured system types (`IntentPerson`, `PlaceDescriptor`), `URLRepresentableEntity` + `URLRepresentableIntent` for no-code universal-link opens, `OpenURLIntent` return type for post-create navigation
- **Snippets + Buttons** - `ShowsSnippetView` (inline), `ShowsSnippetIntent` + `SnippetIntent` (indirect), interactive `requestConfirmation(actionName:snippetIntent:)` flows, iOS 26 snippet refresh cycle + `SnippetIntent.reload()`, 340pt height ceiling, Result vs Confirmation snippet types, `Button(intent:)`
- **Shortcuts + Siri** - `AppShortcutsProvider`, the `\(.applicationName)` rule, `updateAppShortcutParameters()` (SwiftUI + UIKit), `SiriTipView`, `ShortcutsLink`, Flexible Matching (iOS 17+), Negative Phrases, AppShortcuts String Catalog, accent colors in Info.plist, Xcode's App Shortcuts Preview tool, watchOS / HomePod specifics
- **Widgets + relevance** - `WidgetConfigurationIntent`, `ControlConfigurationIntent`, `RelevantIntentManager` + `RelevantIntent` for Smart Stack / complications
- **Spotlight** - `IndexedEntity`, `CSSearchableIndex`, `associateAppEntity(_:priority:)` for existing pipelines, `@ComputedProperty(indexingKey:)`, custom attribute keys
- **Dependencies** - `@Dependency`, `AppDependencyManager`, data-controller pattern, `AppIntentsPackage` (frameworks, Swift Packages, static libs), cross-module entities, `UISceneAppIntent` + `AppIntentSceneDelegate` for UIKit, `contentIdentifier` + `handlesExternalEvents` for scene routing
- **Widgets** - `WidgetCenter.reloadAllTimelines()` after intent writes, App Group sharing pattern for interactive widget state
- **SwiftData** - `ModelContainer` vs `ModelContext` sendability, safe cross-actor patterns
- **Siri + Apple Intelligence** - `@AppEntity` / `@AppIntent` / `@AppEnum` schema macros, 13+ domains (`.calendar.*`, `.messages.*`, `.photos.*`, `.journal.*`, `.mail.*`, `.browser.*`, `.visualIntelligence.*`, `.system.searchInApp`, ...), three content-discovery paths (semantic index / structured search / in-app search), multi-schema build errors + Xcode fix-its, custom Siri responses, interaction donations (`IntentDonationManager`), confirmation + ownership, **Use Model** action via `AttributedString` parameters
- **On-screen awareness + content transfer** - the four annotation APIs (`.userActivity`, `.appEntityIdentifier`, collection `forSelectionType`, custom-canvas `.appEntityUIElements`), UIKit/AppKit equivalents, `displayRepresentations` fast path, entity annotations on User Notifications / Now Playing / AlarmKit
- **Visual Intelligence** - `IntentValueQuery` + `SemanticContentDescriptor`, now on iOS / iPadOS / macOS, `@UnionValue` multi-type results, `semanticContentSearch` "More results", receiving data via system stores (EventKit / Contacts / HealthKit)
- **Testing** - the new `AppIntentsTesting` framework (out-of-process integration: `IntentDefinitions`, `makeIntent`/`run()`, `entities(matching:)`, `spotlightQuery()`, `viewAnnotations()`, test-only intents), plus direct `perform()` struct tests, mocking `@Dependency`, Swift Testing patterns, what you can / can't unit-test
- **Anti-patterns** - ~40 catches including `@Model` as `AppEntity`, missing `\(.applicationName)`, `@Query` inside intents, unregistered intents, missing `@Property`, `AppEntity` instead of `TransientAppEntity`, missing `Transferable`, duplicate `perform()` on `URLRepresentableIntent`, mutation / expensive work in `SnippetIntent.perform()`, resolving entities at scale instead of `EntityCollection`, `LongRunningIntent` without progress, plain `nil` checks instead of `valueState`, cross-process write conflicts, `TransientAppEntity` for annotations, over-donating interactions, and more


## Installing

You can install this skill into Claude Code, Codex, Gemini, Cursor, and more by using `npx`:

```bash
npx skills add https://github.com/n0an/App-Intents-Agent-Skill --skill app-intents
```

If you get the error `npx: command not found`, it means you don't currently have Node installed. You need to run this command to install Node through Homebrew:

```bash
brew install node
```

And if *that* fails it usually means you need to [install Homebrew](https://brew.sh) first.

When using `npx`, you can select exactly which agents you want to use during the installation. You can also select whether the skill should be installed just for one project, or whether it should be made available for all your projects.

### Alternative install methods

**Claude Code:**

```bash
/plugin install n0an/App-Intents-Agent-Skill
```

**Gemini:**

```bash
gemini extensions install https://github.com/n0an/App-Intents-Agent-Skill.git --consent
```

Alternatively, you can clone this whole repository and install it however you want.


## Using App Intents

The skill is called App Intents, and can be triggered in various ways. For example, in Claude Code you would use this:

> /app-intents

And in Codex you would use this:

> $app-intents

In both cases you can provide specific instructions if you want only a partial review. For example, `/app-intents Fix the entity query in BookmarkEntity.swift` on Claude, or `$app-intents Add an OpenIntent that navigates to a selected article` in Codex.

You can also trigger the skill using natural language:

> Use the App Intents skill to review my Shortcuts integration in this project.


## Why Use an Agent Skill for App Intents?

App Intents is a fast-moving framework that has changed significantly across iOS 16, 17, 18, 26, and the 27 releases (WWDC 2026). Most LLM training data either predates App Intents entirely or reflects the older SiriKit approach. As a result, agents routinely generate code that:

- Tries to make a SwiftData `@Model` class conform to `AppEntity` - which no longer compiles under Swift 6 because `AppEntity` requires `Sendable`
- Uses SwiftUI `@Query` inside an intent, which silently does nothing because intents aren't views
- Forgets `\(.applicationName)` in Siri phrases, which fails the `AppShortcutsProvider` macro at build time
- Creates `ModelContainer` and `ModelContext` inside `perform()` instead of injecting a shared data controller via `@Dependency`
- Returns the wrong `IntentResult` variant, producing runtime type-check crashes that aren't caught by the compiler
- Omits `\(.applicationName)` and `shortcutTileColor`, skips `SiriTipView`/`ShortcutsLink`, or never registers the intent in `AppShortcutsProvider`
- Reaches for `NSUserActivity` or old `SiriKit` APIs when modern App Intents cover the use case cleanly

This skill:

- **Catches anti-patterns** LLMs default to, like `@Model` entity conformance, bare `String(format:)` dialog, and unregistered intents
- **Provides copy-pasteable patterns** for every intent kind (action, open, snippet, focus, control widget) with correct return types
- **Covers newer APIs** like `IndexedEntity`, `OpenIntent`, `ShowsSnippetView`, `@AssistantEntity` / `@AssistantIntent` schemas, and the 27 releases additions (`LongRunningIntent`, `EntityCollection`, `SyncableEntity`, on-screen awareness, `AppIntentsTesting`)
- **Enforces data-flow best practices** like `AppDependencyManager` injection, main-actor-bound data controllers, and `ModelContainer`-only cross-actor transfer


## Contributing

Contributions are welcome - whether adding new checks, improving existing examples, or fixing typos.

- Keep Markdown concise. There is a token cost to using skills, so respect the token budgets of users.
- Do not repeat things LLMs already know. Focus on edge cases, surprises, and common mistakes.
- All work must be licensed under the MIT license.


## License

Available under the [MIT License](LICENSE), which permits commercial use, modification, distribution, and private use.
