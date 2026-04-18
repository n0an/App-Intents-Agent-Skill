<h1 align="center">App Intents Agent Skill</h1>

<p align="center">
    <img src="https://img.shields.io/badge/iOS-16+-2980b9.svg" alt="iOS 16+" />
    <img src="https://img.shields.io/badge/swift-5.9+-F05138.svg" alt="Swift 5.9+" />
    <img src="https://img.shields.io/badge/license-MIT-lightgrey.svg" alt="MIT License" />
    <a href="https://agentskills.io/home">
        <img src="https://img.shields.io/badge/Agent%20Skills-Compatible-purple.svg" alt="Agent Skills Compatible" />
    </a>
</p>

An agent skill that helps AI coding agents like Claude Code, Codex, Cursor, and Gemini write correct Swift App Intents code - exposing app actions and data to Siri, Shortcuts, Spotlight, widgets, Control Center, and Apple Intelligence.

It uses the [Agent Skills](https://agentskills.io/home) format, so it works smoothly with Claude Code, Codex, Gemini, Cursor, and more.


## What It Covers

- **Intents** - `AppIntent`, `OpenIntent`, `SnippetIntent`, dialog, grammar agreement, return types, `IntentDescription`, `isDiscoverable`
- **Parameters** - `@Parameter`, primitives, entity parameters, `@AppEnum` (`typeDisplayName`), parameter prompts, `requestValue` vs `needsValueError`
- **Entities** - `AppEntity`, `IndexedEntity`, the shadow-struct pattern for SwiftData, display representations
- **Queries** - `EnumerableEntityQuery`, `EntityQuery`, `EntityStringQuery`, `UniqueIDEntityQuery`
- **Snippets + Buttons** - `ShowsSnippetView` (inline), `ShowsSnippetIntent` + `SnippetIntent` (indirect), `Button(intent:)` in SwiftUI/widgets
- **Shortcuts + Siri** - `AppShortcutsProvider`, the `\(.applicationName)` rule, `updateAppShortcutParameters()`, `SiriTipView`, `ShortcutsLink`
- **Spotlight** - `IndexedEntity`, `CSSearchableIndex`, attribute sets, indexing strategies
- **Dependencies** - `@Dependency`, `AppDependencyManager`, data-controller pattern
- **Widgets** - `WidgetCenter.reloadAllTimelines()` after intent writes
- **SwiftData** - `ModelContainer` vs `ModelContext` sendability, safe cross-actor patterns
- **Apple Intelligence** - `@AssistantEntity` / `@AssistantIntent` schemas for journal, photos, mail, etc.
- **Anti-patterns** - catches `@Model` as `AppEntity`, missing `\(.applicationName)`, `@Query` inside intents, unregistered intents, missing `isDiscoverable` on helpers, stale widgets, stale shortcut parameters, and more


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

App Intents is a fast-moving framework that has changed significantly across iOS 16, 17, and 18. Most LLM training data either predates App Intents entirely or reflects the older SiriKit approach. As a result, agents routinely generate code that:

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
- **Covers newer APIs** like `IndexedEntity`, `OpenIntent`, `ShowsSnippetView`, `@AssistantEntity` / `@AssistantIntent` schemas
- **Enforces data-flow best practices** like `AppDependencyManager` injection, main-actor-bound data controllers, and `ModelContainer`-only cross-actor transfer


## Contributing

Contributions are welcome - whether adding new checks, improving existing examples, or fixing typos.

- Keep Markdown concise. There is a token cost to using skills, so respect the token budgets of users.
- Do not repeat things LLMs already know. Focus on edge cases, surprises, and common mistakes.
- All work must be licensed under the MIT license.


## License

Available under the [MIT License](LICENSE), which permits commercial use, modification, distribution, and private use.
