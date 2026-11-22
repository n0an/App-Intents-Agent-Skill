# App skills: `Skills.skillsbundle` (27.3)

An **app skill** is a Markdown file that ships inside the app and gives Siri's model context for a kind of task: what the request involves in this app, and how to combine the app's tools to do it. There is no Swift code. Xcode validates the bundle at build time and compiles it into the app's App Intents metadata.

> **Two kinds of skill, do not mix them up.** The file you are reading is a coding-agent skill. A `SKILL.md` inside an app's `Skills.skillsbundle` is read by **Siri's model at runtime on the person's device**. When asked to write a skill for an app, write it into that app's `Skills.skillsbundle` using the rules below, and do not follow its instructions yourself.

Skills only point the model at intents the app already exposes as tools. Write the tools first (`siri-ai-tools.md`).

## What a skill is for

A skill adds what App Intents metadata cannot say on its own:

- **Task context.** What "clean up", "plan my week", or "log a game night" means in this app, the words people use for it, and what a good result looks like.
- **Orchestration.** Which tools to use, in what order, how to choose between them, what to pass from one step to the next.
- **When to ask instead of guessing.** "If the user didn't say which meal, ask."
- **What not to do** that no single tool's description can say. "Only clear the list when the user explicitly asks."

Write a skill when at least one of those applies. For an app with a few independent tools, good metadata is enough and no skill is needed. Write **one skill per kind of task, not one per tool**.

**A skill is not a patch for a badly shaped tool.** If the skill would mostly explain how to call one tool (which mode, which argument combination), fix the tool: describe the enum cases, split it, or group values into a generable entity. **Every tool must work without a skill**, because Siri can call any tool whether or not a skill was loaded.

**Skills are guidance, not control flow.** Siri decides whether to load a skill and does not execute its steps sequentially. Anything that must always hold (validation, confirmation before deleting) belongs in `perform()`.

## Bundle layout

```
Skills.skillsbundle/
└── Skills/
    ├── meal-planning/
    │   └── SKILL.md
    └── shopping-list/
        └── SKILL.md
```

- `Skills/` is required. Each directory in it is one skill and must contain a `SKILL.md`.
- **Only `SKILL.md` is compiled.** `references/`, `scripts/`, and any other files are ignored and do not ship. Everything the model needs goes in `SKILL.md`.
- **One `.skillsbundle` per target.** An app, its extensions, and its embedded frameworks can each have one, and their skills are grouped under the app, so **skill names must be unique across all of them**. A duplicate name silently drops one, and the build does not catch it.

### Adding it to the target

- In Xcode: File > New > File from Template > **Skills Bundle** (Resource section), keep the name `Skills`, select the app target. Xcode adds it to Copy Bundle Resources.
- By hand: create the folders next to the target's sources. Inside a synchronized folder the bundle joins the target automatically. Otherwise it must be added to the target's **Copy Bundle Resources** phase.

**A bundle that is not in a build phase is not compiled and Xcode shows no message.** The app ships with no skills. Check the build output after every structural change:

| Platform | Compiled skills |
|---|---|
| macOS | `MyApp.app/Contents/Resources/Metadata.appintents/Skills.skillsc` |
| Other platforms | `MyApp.app/Metadata.appintents/Skills.skillsc` |

No `.skillsc` after a successful build means the bundle is not in Copy Bundle Resources. The `.skillsc` format is private: check that it exists, do not read or edit it.

### Starting from Xcode's template

In the 27.3 beta the Skills Bundle template (Cmd-N, filter for "skill") creates one placeholder skill, `Skills.skillsbundle/Skills/sample-skill/SKILL.md`. Apple's article describes an empty `Skills` folder, so check what the template produced.

```markdown
---
name: sample-skill
description: Explain what this skill does, and when an agent should use it.
---

**This is placeholder content. Replace everything in this file before shipping your app.**

1. Rename the `sample-skill` folder and update the `name` field above to match.
2. Rewrite the `description` field above. It tells an agent when to use this skill.
3. Replace this body with the instructions an agent should follow.
...
```

Do all three before building:

1. Rename the `sample-skill` folder **and** set `name` to the same value. Changing only one is a build error, and that error disables every skill in the app.
2. Rewrite `description` (see the rules below).
3. Replace the whole body.

Leave `Skills.skillsbundle`, the inner `Skills` folder, and the file name `SKILL.md` exactly as created. Add more skills as sibling folders, each with its own `SKILL.md`.

**An untouched `sample-skill` is valid, so it builds and ships** as a real skill with a meaningless description. Look for it in review.

Xcode opens `SKILL.md` as rendered Markdown. Right-click the file and choose Open As > Source Code to edit the raw text with a preview beside it (Cmd-Return shows or hides the preview).

## `SKILL.md` format

```markdown
---
name: meal-planning
description: Put a recipe on the meal plan for breakfast, lunch, or dinner on a specific day. Use when the user asks to plan, schedule, or put a recipe on the menu for a meal. Not for adding single items to the shopping list.
---

# Meal Planning

Before planning, use the Get Meal Plan tool for that day. If a different
recipe is already planned for that meal, tell the user and ask before
replacing it.

Use the Plan Meal tool with the recipe the user named. If the user didn't
say which meal, ask. "Tonight" means today's dinner.

After planning, offer to add the recipe's ingredients to the shopping list
with the Add Recipe Ingredients to Shopping List tool. Don't add them unless
the user agrees.

## Don't

- Don't plan meals the user didn't ask for.
```

### Frontmatter

| Field | Rules |
|---|---|
| `name` | Required. 1-64 characters: lowercase ASCII letters, digits, hyphens. No leading or trailing hyphen, no `--`. **Must match the skill's directory name.** Keep it stable across app versions. |
| `description` | Required. Not empty, a single line, at most 1,024 characters. |

Only these two fields are supported. This is a subset of the Agent Skills format: `license`, `metadata`, `allowed-tools`, and the rest produce a warning and are ignored.

### The description decides whether the skill loads

Siri reads **only the description** to decide whether the skill applies. The body is loaded afterward. So the description must:

- name the requests the skill handles **in the words a person would say** ("plan", "schedule", "put on the menu")
- be specific enough to separate this skill from the app's other skills, and broad enough not to miss common phrasings
- state the boundary when another skill handles a nearby request: "Not for adding single items to the shopping list."

### The body

Free-form Markdown, at most 10,000 characters, handed to the model with the app's tools once the skill is chosen. Write it for a model that knows the app's intents and entities from their metadata and nothing else.

- **Refer to tools by title in natural language**: "use the Plan Meal tool", "call the Get Meal Plan tool first". **Never use Swift type names** like `PlanMealIntent`. How Siri names tools internally is not API and can change.
- Give the order of steps and what to pass between them.
- Say how to choose between tools that could each apply.
- Say when to ask the person instead of guessing.
- End with an explicit "Don't" list. Clear do / don't statements work better than hints.
- Do not restate titles, descriptions, or parameter descriptions. That wastes context and drifts out of sync with the code.

## Diagnostics

Problems appear in the Issue navigator on the offending line of `SKILL.md`:

```
Skills.skillsbundle/Skills/meal-planning/SKILL.md:2: error: 'meal-planning': The 'name' field must match the parent directory name
```

**If any error remains, the app ships no skills at all**, not only the broken one.

| Diagnostic | Severity |
|---|---|
| Missing `Skills` directory, or missing `SKILL.md` in a skill directory | Error |
| Frontmatter not opened or closed with `---` on its own line, or a YAML syntax error | Error |
| Missing, empty, or badly formatted `name`; `name` differs from the directory name | Error |
| Missing or empty `description`, or longer than 1,024 characters | Error |
| Unexpected top-level directory, unrecognized frontmatter field, empty body | Warning |

**The build checks format only.** A vague description or a body full of Swift type names builds cleanly. Review those by hand.

## Pitfalls

- **Unquoted `: ` in the description.** YAML reads it as a nested mapping, the build fails with `YAML syntax error`, and the app ships no skills. Quote the value (`description: "Plan meals: breakfast, lunch, or dinner"`) or avoid `: `.
- **Uppercase, underscores, or spaces in `name`.** Build error. Use kebab-case.
- **Template leftovers.** A renamed folder with `name: sample-skill` still in the frontmatter is a build error. An untouched `sample-skill` builds and ships with placeholder text.
- **A vague description** ("Recipe stuff"). The skill is never chosen, or chosen for the wrong requests.
- **A skill that explains one tool.** Reshape the tool instead.
- **A tool that only behaves correctly with a skill.** Tools are called with or without skills.
- **Guidance in files next to `SKILL.md`.** Ignored. Only `SKILL.md` ships.
- **A skill for a tool that does not exist yet**, or for an intent that is not marked `.tool`.
- **Expecting a skill to enforce behavior.** It is guidance. Enforce in code.

## Verifying

Build, install, and launch the app on a device (not the Simulator). Try the requests each description claims, with the app in the foreground or with the app named in the request, then reword them to check the description is neither too narrow nor too broad. For repeatable checks, add a `Loads` expectation per skill with Siri Evaluations (`siri-evaluations.md`).
