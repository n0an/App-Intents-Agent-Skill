# Verifying Siri AI: metadata analyzer and Siri Evaluations (27.3)

Unit tests prove a tool works. They cannot prove Siri **picks** it, fills it correctly, or loads the right skill. The Siri AI Developer Preview adds tooling for that, from cheapest to most realistic:

1. **`sirimetadataanalyzer`** - static check of the built metadata. No device. Run on every metadata change.
2. **`AppIntentsTesting`** - runs the intents out of process, no Siri. See `testing-intents.md`.
3. **Siri Evaluations** - scripted conversations through real Siri on a device, graded against expectations.
4. **Manual Siri testing** on a device, with varied wording, with and without onscreen context.

Apple's documentation also names a **Siri Integration Advisor** in Xcode that gives feedback on the App Intents implementation. The docs do not detail it yet.

## Siri Metadata Analyzer

`sirimetadataanalyzer` reads an app's **built** App Intents metadata and reports how well Siri's model can use it: missing or unclear titles and descriptions, entities that expose too little, tools that closely match a system schema.

```sh
xcrun sirimetadataanalyzer path/to/MyApp.app
```

It also accepts the `Metadata.appintents` directory or the `extract.actionsdata` file inside it. Find the built app with:

```sh
xcodebuild -showBuildSettings -scheme MyApp | grep -E ' (BUILT_PRODUCTS_DIR|FULL_PRODUCT_NAME) ='
```

By hand from Xcode: build (Cmd-B), choose Product > Show Build Folder in Finder, open `Products/Debug-iphoneos`, type `xcrun sirimetadataanalyzer ` in Terminal, and drag `MyApp.app` onto the window to paste its path. The result looks like this:

```sh
xcrun sirimetadataanalyzer ~/Library/Developer/Xcode/DerivedData/MyApp-<hash>/Build/Products/Debug-iphoneos/MyApp.app
```

Keep that Terminal window open and rerun the same command (Up, Return) after every build. The summary line (`5 intents (5 tools), 3 entities`) is also a quick check that a new tool or entity actually made it into the metadata.

If `xcrun` cannot find the tool, the active developer directory is an older Xcode. Select Xcode 27.3 under Xcode > Settings > Locations > Command Line Tools (or with `xcode-select`).

| Option | Use |
|---|---|
| `--format text\|json\|diagnostics\|preview` | Output format. Default `text`. Use `json` to act on findings. |
| `--language <code>` | Analyze one localization, such as `fr`. Run once per shipping language. |
| `--config <file>` | JSON that disables rules, overrides severities, or suppresses findings |

**Exit status:** `0` no error-level findings, `2` some, `1` the analysis could not run (the path is not built App Intents metadata).

Text output:

```
Metadata.appintents - 2 intents (0 tools), 3 entities

  info       SMA109  app
             The app has 2 intents but none is a tool, so Siri can't call any of them. ...

0 errors · 0 warnings · 0 suggestions · 1 info
```

Each finding has a rule id, a severity (`error`, `warning`, `suggestion`, `info`), the intent / parameter / entity it applies to, and a message that says what to change. Findings that need context:

| Rule | Meaning | Note |
|---|---|---|
| `SMA109` | The app has intents but none is a tool | Add `.tool` to the intents people would ask Siri for |
| `SMA202` | Parameter has no description | Fine to leave when the title says everything ("Quantity" on an `Int`) |
| `SMA203` | Entity property has no description | Applies to every entity. On an entity that ships before 27.3, see the fallback in `siri-ai-entities.md` |
| `SMA601` | Entity exposes no properties | The model sees only the display representation |
| `SMA701` | Tool closely matches a system schema | Adopt the schema if the tool does what it covers. Otherwise make the title and description say what is specific to the app |

Workflow: build, run with `--format json`, fix every `error` then the `warning`s **in source metadata**, rebuild, repeat until it exits `0` and what remains is deliberate.

- **It reads build output, not source.** After a failed build it silently reports the previous build's findings. Confirm the build succeeded first.
- **Do not suppress findings to get a clean run.** Use `--config` only for behavior the developer decided to keep.
- **It does not check** overlapping tools ("Add" vs "Add Product"), vague skill descriptions, or Swift type names in a skill body.
- **A clean run is not proof that Siri works.** It checks the shape of the metadata.

## Siri Evaluations

The `SiriEvaluations` framework scripts requests to Siri, records which intents Siri calls, which parameter values it passes, which entities it finds, which skills it loads, and what it replies, then grades each against expectations. It builds on `Evaluations`, `AppIntentsTesting`, and Swift Testing.

### Prerequisites

- **A UI Testing Bundle target** (File > New > Target > UI Testing Bundle). Siri evaluations run only there. Xcode 27.3 lets UI test targets use Swift Testing.
- The test target's **deployment target is 27.3 or later** (the app keeps its own). Do not mark a `@Suite` or `@Test` `@available(iOS 27.3, *)`: Swift Testing rejects it.
- The app is signed by the **same development team** as the test target.
- **A physical device** on 27.3 or later. No Simulator. Developer Mode on, Enable UI Automation on (Settings > Developer), Siri and Apple Intelligence on with models downloaded.
- The app is installed and **launched once since install**, so its intents are registered and its entities indexed. Reinstalling resets this.
- At least one intent is a tool.
- Use a dedicated test device with fictional data. Siri really performs the intents. A device with few apps and little data scores higher than real devices, so stage realistic data and install apps that handle similar requests.

### Test file

Import `SiriEvaluations` and `Testing` only. `SiriEvaluations` re-exports `AppIntentsTesting` and `Evaluations`.

```swift
import SiriEvaluations
import Testing

// Declared once per test target.
struct CometCalEvaluation: Evaluation {
    let name: String                 // titles the report
    let dataset: SiriScenarioLoader

    func subject(from sample: SiriScenario) async throws -> SiriScenarioSubject {
        try await resetAppState()                              // per-scenario setup
        let subject = try await SiriService.run(scenario: sample)
        do { try await resetAppState() } catch { Issue.record(error) }   // cleanup must not throw
        return subject
    }

    var evaluators: Evaluators { SiriExpectationEvaluator() }

    func aggregateMetrics(using aggregator: inout MetricsAggregator) {
        Self.computeSiriMetrics(into: &aggregator)
    }

    private func resetAppState() async throws {
        try await IntentDefinitions(bundleIdentifier: "com.example.CometCal")
            .intents["ResetDemoDataIntent"].makeIntent().run()   // a #if DEBUG test-only intent
    }
}

@Suite("CometCal - Siri", .serialized)
struct CometCalSiriEvaluationTests {
    static let app = IntentDefinitions(bundleIdentifier: "com.example.CometCal")
    static let CreateEventIntent = app.intents["CreateEventIntent"]     // keyed by Swift type name
    static let DeleteEventIntent = app.intents["DeleteEventIntent"]
    static let EventEntity = app.entities["EventEntity"]
    static let CalendarEntity = app.entities["CalendarEntity"]
    static let PlanEventSkill = app.skills["plan-event"]                // keyed by the skill's name:

    static let createEvent = SiriScenario {
        UserTurn("In CometCal, create an event called Comet Watch tomorrow at 3PM in my Deep Space calendar") {
            Calls {
                CreateEventIntent
                    .expect(title: .equals("Comet Watch"),
                            calendar: CalendarEntity.expectIdentifier("deep-space"))
                    .result(EventEntity.expect(title: .equals("Comet Watch")))
            }
            Calls(.disallowed) { DeleteEventIntent }
        }
    }

    static let evaluation = CometCalEvaluation(
        name: "CometCal - Create an event",
        dataset: SiriScenarioLoader { createEvent }
    )

    @Test("Siri creates the event with the right title and calendar", .evaluates(evaluation))
    func createsEvent() {
        let result = EvaluationContext.current.result
        #expect(result.aggregateValue(.mean(of: .intentsAllPass)) == 1.0)
    }
}
```

- **`.serialized` is required.** A device runs one Siri conversation at a time.
- Pass `sample` to `SiriService.run(scenario:)` unchanged. A subject run from another scenario is not graded.
- One evaluation value per behavior, one `@Test` per evaluation. Paraphrases of the same request are separate scenarios in one loader: `SiriScenarioLoader { a; b }`.
- Name the app in each utterance ("In CometCal, ...") unless the scenario deliberately tests app-less routing.
- Every intent, entity, parameter, and skill name comes from the app's source. Parameter labels are the Swift property names.

### Expectations

Expectations go in a turn's trailing block, or in `SiriScenario { ... } expectation: { ... }` when any turn may satisfy them.

| To check | Write |
|---|---|
| Siri picked the intent | `Calls { CreateEventIntent }` |
| A scalar parameter | `CreateEventIntent.expect(title: .equals("..."))` |
| An entity parameter resolved to one entity | `.expect(calendar: CalendarEntity.expectIdentifier("<id>"))` |
| The intent's return value | `.result(EventEntity.expect(title: .equals("...")))` or `.result(.notNil)` |
| Several intents in order / in any order | `Calls { A; B }` / `Calls(.unordered) { A; B }` |
| Siri did **not** do something | `Calls(.disallowed) { DeleteEventIntent }` |
| Siri found the right entity | `Finds { EventEntity.expect(title: .contains("...")) }` |
| An entity was on screen | `OnScreen { EventEntity.expectIdentifier("<id>") }` |
| What Siri replied | `SiriResponse(.contains("...", caseSensitive: false))` (turn block only) |
| A skill was loaded | `Loads(.required) { PlanEventSkill }` or `Loads(.preferred) { ... }` |

Matchers: `.equals`, `.contains`, `.contains(regex:)`, `.notNil`, `.isNil`, `.matches(#Predicate { ... }, requirement: "...")`.

- Prefer `.equals` when the whole value is known. `.contains` cannot catch Siri stuffing extra words into a parameter.
- Constrain only what the utterance determines. Do not assert dates or times as exact strings.
- To expect absence write `.isNil`. A literal `nil` leaves the value unchecked.
- A `.disallowed` call counts when it ran **and** when Siri chose it and asked the person to confirm. Pair action scenarios with `Calls(.disallowed)` for the destructive tools.
- A clarification is a turn. If Siri should ask for a missing value, expect a `SiriResponse` on that turn and put the answer and the `Calls` on the next `UserTurn`.
- `OnScreen` needs the app staged in the foreground with the entity visible.
- `Finds` can only check properties Siri actually read: the indexed ones when the entity came from Spotlight, the query's results otherwise.
- Cover **every app skill** with a `Loads` expectation in its own evaluation, so a skill that did not load is reported separately from a wrong intent call. Only `.required` is scored.

### Metrics

Assert on `result.aggregateValue(.mean(of: <metric>))`. Each metric is 1 or 0 per scenario.

| Metric | Facet |
|---|---|
| `.intentsAllPass` | `Calls`, every ordering including `.disallowed` |
| `.foundEntitiesAllPass` | `Finds` |
| `.onScreenEntitiesAllPass` | `OnScreen` |
| `.skillsLoadedAllPass` | `Loads(.required)` |
| `.siriResponseAllPass` | `SiriResponse` |

Assert only on facets the scenarios constrain. A facet with no expectations is not scored and its mean reads `-1`. The older `.intentsPass`-style names are deprecated aliases.

### Running

- Run from Xcode with the device selected, unlocked, and untouched. The device asks for its passcode to turn on Automation Mode. Each turn can take up to a minute.
- **Siri is nondeterministic. Run the suite more than once.** One failure among several paraphrases is a signal to investigate. A failure that repeats across reruns and paraphrases is systematic.
- The report is in the Report navigator under the test run > Evaluations: summary cards, one row per scenario, and per turn the utterance, each check, and Siri's response. From a result bundle: `xcrun xcresulttool export attachments --path <bundle>.xcresult --output-path <dir>` exports one `.xcevalresult` per evaluation.
- The test's `#expect` is only the summary. Read the per-scenario rows to explain a failure.

### Reading a failure

Decide first whether the failure is in the **environment**, the **test**, or the **app**. Only app failures lead to App Intents changes.

- **Environment:** every scenario fails the same way, Diagnostics shows fewer completed turns than the scenario has, or the Spotlight index is empty. Recheck the prerequisites. Do not change the scenarios or the app.
- **Test:** the expectation does not match what the utterance asks for. Fix the expectation and say so. Never loosen an expectation just to pass.
- **App:** walk the failing rows in turn order. The first failure in a turn usually explains the rest. Then read what the failure points at: the intent's `title`, `description`, `@Parameter` descriptions, return type; the entity's `displayRepresentation`, query, and indexing; the skill's description.

Typical fix: Siri folded "tomorrow at 3PM" into `title` because the parameter had no description and the intent had no date parameter. Add a description ("The event's name only, without its date or time") and a `startDate` parameter, then rebuild, **reinstall, launch**, and rerun.

Do not fix a failure by renaming an intent or entity type, changing an id scheme, or adding / removing / renaming parameters without flagging it: those break saved shortcuts and every scenario that names them.
