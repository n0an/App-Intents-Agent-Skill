# Open intents and snippet views

Two different shapes for "bring a thing to the user":

- `OpenIntent` - **launch the app and navigate to the thing**. Good when the user wants to interact (edit, reply, continue reading).
- `AppIntent & ShowsSnippetView` - **show a summary right there**, no app switch. Good when the user wants information, not interaction.

They can coexist; pick per use case.

## `OpenIntent`

`OpenIntent` is a subprotocol of `AppIntent`. It requires a `target` parameter (the entity being opened) and automatically opens the app when the intent completes:

```swift
import AppIntents

struct OpenArticleIntent: OpenIntent {
    static let title: LocalizedStringResource = "Open article"

    @Dependency var navigator: AppNavigator

    @Parameter(title: "Article")
    var target: ArticleEntity     // MUST be named `target`

    func perform() async throws -> some IntentResult {
        try await navigator.navigate(to: target)
        return .result()
    }
}
```

The `target` property name is required; the protocol keys off it. The app is opened automatically *after* `perform()` returns - your job in `perform()` is to update navigation state so that when the app comes to the foreground, the correct screen is on top.

### Navigation wiring

Most apps route navigation through a main-actor-bound controller that drives a `NavigationStack`:

```swift
@Observable @MainActor
final class AppNavigator {
    var path: [Article] = []

    func navigate(to entity: ArticleEntity) async throws {
        let id = entity.id
        let results = try await store.articles(matching: #Predicate { $0.id == id })
        if let article = results.first {
            path = [article]
        }
    }
}
```

```swift
struct ContentView: View {
    @Bindable var navigator: AppNavigator

    var body: some View {
        NavigationStack(path: $navigator.path) {
            ArticleList()
                .navigationDestination(for: Article.self, destination: ArticleEditor.init)
        }
    }
}
```

Inject `AppNavigator` through `@Dependency` so intents can reach it. See `dependencies.md`.

### When the app isn't running

`OpenIntent` works even when the app has never been launched. The app process starts, `App.init()` runs (registering dependencies), the intent fires, navigation state is set, then the window appears on screen already at the right place. This is why **all cross-intent setup belongs in `App.init()`** - not in `.onAppear`, not in view modifiers.

## Snippet views

A snippet view is a compact SwiftUI scene rendered by the system in response to the intent. The user doesn't leave their current context; they just see the answer.

```swift
import AppIntents
import SwiftUI

struct SummarizeArticleIntent: AppIntent {
    static let title: LocalizedStringResource = "Summarize article"

    @Parameter(title: "Article")
    var article: ArticleEntity

    func perform() async throws -> some IntentResult & ProvidesDialog & ShowsSnippetView {
        .result(dialog: "\(article.title)") {
            VStack(alignment: .leading, spacing: 8) {
                Image(systemName: "doc.text")
                    .font(.largeTitle)
                Text(article.title).font(.headline)
                Text(article.summary).font(.body)
            }
            .padding()
        }
    }
}
```

The snippet is encoded and transferred like a widget. Don't use `List`, `ScrollView`, or any interactive control that needs a live `UIViewController` behind it - they will either fail to render or behave oddly. Stick to static layout: `VStack`, `HStack`, `Text`, `Image`, `Label`, `Spacer`, backgrounds, padding.

### Snippet-only imports

The file producing a snippet needs both frameworks:

```swift
import AppIntents
import SwiftUI
```

### Dark mode gotcha

Snippets rendered by Siri do not live-update when the user toggles dark mode *while the snippet is on screen*. Dismissing and triggering again picks up the current appearance correctly.

## OpenIntent vs Snippet - picking the right one

| Situation | Pick |
|---|---|
| User will read, interact, edit | `OpenIntent` |
| User wants a one-shot fact or summary | `AppIntent & ShowsSnippetView` |
| User wants a value to chain in Shortcuts | `AppIntent & ReturnsValue` |
| Simple confirmation ("Done") | `AppIntent & ProvidesDialog` |

For "show me my latest note" - a snippet is usually better, because the point is to read it, not to edit it. For "open my latest note so I can keep writing" - use `OpenIntent`.

## Bridging Spotlight selection to `OpenIntent`

When a user taps an app entity in Spotlight results, the system looks for an `OpenIntent` whose `target` matches that entity type and invokes it. As long as:

1. The entity conforms to `IndexedEntity`.
2. The entity has been indexed (see `spotlight.md`).
3. An `OpenIntent` exists with `target: YourEntity`.

...tapping the Spotlight result routes through your `OpenIntent` automatically. No additional wiring.

In simulator this sometimes takes a few minutes after first launch before it starts working reliably - the index builds up in the background. On device it's generally faster.
