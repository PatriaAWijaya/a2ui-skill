# a2ui-skill

A Muse skill for building agentic experiences with Google's **A2UI protocol**: the agent describes interactive UI as a stream of JSON from a component catalog, the SDK compiles and validates it, and the client renders it with native widgets.

Studied from https://a2ui.org/ (protocol v1.0, 25 Nov 2026). Source repo of the protocol: https://github.com/google/a2ui.

## What's inside

- `SKILL.md` — the skill: pipeline, wire messages, catalog authoring, basic catalog inventory, Express inference format, Agent/Renderer SDKs, quickstart demo, MCP Apps.
- `skill.yaml` — skill metadata (name, description, category).

## Install as a Muse skill

```sh
git clone https://github.com/PatriaAWijaya/a2ui-skill.git ~/workspace/custom-skills/a2ui
```

The skill is then discoverable by name (`a2ui`) for any task about building, integrating, or debugging an A2UI agent, renderer, catalog, or surface, or running agent-generated UI inside an MCP App.

## In short

UI is structured data, never executable code. The agent never sends HTML, CSS, or scripts. Both sides agree on a catalog contract; the model composes only components and functions from that catalog; the compiler validates the output before anything is sent; the renderer draws it with native widgets (React, Lit, Angular, Flutter, SwiftUI, Jetpack Compose).

```
catalog.json → skill generator → A2UI skill → model output (Express / Direct JSON)
  → compiler validates → createSurface / updateComponents / updateDataModel
  → renderer draws the surface → user taps send `action` messages back
```

## License

Apache-2.0, matching the A2UI project itself. See `LICENSE`.
