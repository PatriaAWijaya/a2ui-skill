# A2UI — Agent-to-UI protocol (Google)

Build agentic experiences where the agent describes an interactive UI as a stream of JSON and the client renders it with its own native widgets. Studied from https://a2ui.org/ (25 Nov 2026), protocol v1.0. Source repo: https://github.com/google/a2ui. Spec and SDK docs: https://a2ui.org/concepts/overview/, https://a2ui.org/concepts/catalogs/, https://a2ui.org/concepts/sdks/.

## When to use

- The user asks for an agent that shows UI, not just chat text: a form, a card, a picker, a booking flow, a dashboard that updates as the agent thinks.
- The user asks to build, integrate, or debug an A2UI agent, renderer, catalog, or surface.
- The user asks how an agent UI works across web/mobile, or how to run A2UI inside an MCP App.
- Do NOT use for a plain chatbot reply with no UI, and do NOT invent components: the agent may only emit components the renderer's catalog knows.

## Core idea

UI is structured data, never executable code. The agent never sends HTML, CSS, or scripts. Both sides agree on a **catalog** contract; the model composes only components and functions from that catalog; the SDK compiler validates the output against the catalog before anything is sent; the renderer draws it with native widgets (React, Lit, Angular, Flutter, SwiftUI, Jetpack Compose).

## Pipeline

```
catalog.json  →  skill generator  →  A2UI skill (prompt + grammar)
    →  model output (Express DSL or Direct JSON)
    →  compiler parses + validates against catalog
    →  wire messages over AG-UI, A2A, MCP, or WebSockets
    →  renderer draws the surface, binds the data model
    →  user taps send `action` messages back to the agent
```

Wire messages (v1.0):

- `createSurface` — `{surfaceId, catalogId}`. Opens the surface. In v1.0 the compiler may inline initial `components` and `dataModel` here.
- `updateComponents` — `{surfaceId, components: [...]}`. A flat list; parents reference children by ID (`child`, `children`), never nested trees.
- `updateDataModel` — `{surfaceId, path, value}`. Sets values in the surface data model (JSON Pointer path, `/` for root).
- `action` (renderer to agent) — `{name, surfaceId, sourceComponentId, timestamp, context}`. Sent when the user triggers something, e.g. taps a Button. Local state edits (a stepper changing a bound number) update the data model on the client and send nothing.

Data bindings in wire JSON use `{"@path": "/outbound/price"}` (v1.0; bare `path` in v0.9.1).

## Catalogs

A catalog is a JSON document with `catalogId` (a URI, e.g. `https://example.com/catalogs/travel/v1/catalog.json`), `components`, `functions`, and Markdown `instructions` (added to the LLM system prompt by the Agent SDK). The renderer advertises `supportedCatalogIds` at session start; the agent picks one in `createSurface.catalogId`. In v1.0 a surface can mix catalogs with `@catalogId` per component and `catalogId` per function call.

### Components

Each entry is a JSON Schema:

- `"component": {"const": "FlightCard"}` names the type.
- Dynamic properties use `$ref` to `DynamicString`, `DynamicNumber`, `DynamicBoolean`, etc. A dynamic value accepts a literal, a data binding (`@path`), or a function call (`@call`).
- Children use `$ref` to `ComponentId` or `ChildList` (flat list, referenced by ID).
- Placement rules: `allowedParents`, `allowedChildren` (v1.0).
- Validation: `$ref` to `Checkable` adds a `@checks` array of `{condition, message}` rules.
- Actions: `$ref` to `Action` (send an event to the agent, or call a function).
- Envelope keys in v1.0 take the `@` prefix: `@catalogId`, `@accessibility`, `@metadata`, `@checks`.

Example component schema (custom `FlightCard`):

```json
{
  "FlightCard": {
    "type": "object",
    "description": "Shows a flight's number, fare, and action buttons.",
    "allowedParents": ["Column", "List"],
    "properties": {
      "component": {"const": "FlightCard"},
      "flightNumber": {"$ref": "common_types.json#/$defs/DynamicString"},
      "fareAmount": {"$ref": "common_types.json#/$defs/DynamicNumber"},
      "actions": {"$ref": "common_types.json#/$defs/ChildList"}
    },
    "required": ["component", "flightNumber"]
  }
}
```

### Functions

Each entry declares `returnType` (`string`, `number`, `boolean`, `array`, `object`, `any`, `void`, or v1.0 `validationResult`) and an `args` schema, invoked as `{"@call": "formatCurrency", "args": {...}}`.

- Renderer-side functions run locally on the client (formatting, validation, `openUrl`, client side effects). In v1.0 the agent can also invoke one remotely via `callRendererFunction` and read `rendererFunctionResponse`.
- Agent-side functions (v1.0) run on the server (e.g. live seat inventory). When a component evaluates an `@call` not registered in the renderer's local catalog, the renderer dispatches `callAgentFunction` and applies `agentFunctionResponse`. In v0.9.1 functions run only on the renderer; the agent learns of interactions through `action` events.

### Basic catalog (18 components, 14 functions)

v1.0 schema: `https://a2ui.org/specification/v1_0/catalogs/basic/catalog.json`. Adopt it as is, restyle it to the host design system, combine it with domain catalogs, or replace it.

- Layout: `Row`, `Column`, `List`, `Card`, `Tabs`, `Modal`, `Divider`
- Display and media: `Text`, `Image`, `Icon`, `Video`, `AudioPlayer`
- Inputs: `Button`, `TextField`, `CheckBox`, `ChoicePicker`, `Slider`, `DateTimeInput`
- Functions: `formatString`, `formatNumber`, `formatCurrency`, `formatDate`, `pluralize`, `required`, `regex`, `length`, `numeric`, `email`, `openUrl`, `and`, `or`, `not`

## Inference format: Express

The model does not write wire JSON by hand. It writes the compact Express DSL inside sentinel tags; the compiler turns it into wire messages. Grammar rules the generated A2UI skill carries:

- Surround the whole block with literal `<a2ui>` and `</a2ui>`.
- Component constructors can be assigned to variables or nested inline; keyword arguments (`param=value`) and positional arguments with `_` placeholders are supported.
- Data bindings: prefix absolute data-model paths with `$`, e.g. `$/user/firstName`.
- Component signatures come from the catalog, e.g. `FlightCard(flightNumber, origin, destination, price, passengers)`.

Example model output (flight booking):

```
<a2ui>
$/outbound/pax = 1
$/outbound/price = 455
$/return/pax = 2
$/return/price = 970
root = Card(Column([
  FlightCard(flightNumber="LH 454", origin="BER", destination="SFO", price=$/outbound/price, passengers=$/outbound/pax),
  FlightCard(flightNumber="UA 901", origin="SFO", destination="BER", price=$/return/price, passengers=$/return/pax),
  Button(Text("Confirm booking"), variant="primary", action=Event("confirm_booking", {outboundPax: $/outbound/pax, returnPax: $/return/pax}))
]))
</a2ui>
```

The compiler validates each component against the catalog as the stream arrives and reports invalid output immediately, so the agent can self-correct before transmission. The other inference format is chunk-healed Direct JSON. The Agent SDK negotiates the active catalog and protocol version against the capabilities the renderer reports.

## SDKs (three layers, protocol v1.0 / v0.9.1 / v0.8 in TS and Python)

- Core SDK: protocol models, surface state, expression evaluation. Packages: TypeScript `@a2ui/web_core`, Python `a2ui-core`, Dart `a2ui_core`, Swift `A2UISwiftCore`, Kotlin `androidx.a2ui`.
- Agent SDK: catalog-driven skill generation, output compilation, stateful validation, optional server-side `@macro` expansion (Python). Packages: TypeScript `@a2ui/agent`, Python `a2ui-agent-sdk` (+ `a2ui-adk` for Google ADK, `a2ui-a2a` for A2A), Dart `a2ui_agent`. Swift/Kotlin target client rendering only.
- Renderer SDK: mounts surface state in a UI framework, built-in Basic Catalog, custom catalog registration. Packages: Flutter `a2ui_flutter`, React `@a2ui/react`, Lit `@a2ui/lit`, Angular `@a2ui/angular`, SwiftUI `A2UISwiftUI`, Jetpack Compose `androidx.a2ui.compose` (+ `material3-a2ui`). On web, `@a2ui/markdown-it` renders Markdown in text components and `@a2ui/catalog-mcp` adds MCP app components.
- Go: community libraries only (`satorunooshie/a2ui-go`, `tmc/a2ui`, `charmbracelet/a2tea`, `muncus/a2term`).
- Rust: NO official or community SDK exists yet. To build one, follow the language-agnostic specs in `blueprints/modules`, verify against the YAML conformance suites in `conformance/core` and `conformance/agent`, and open a PR to add the package to Community Projects.

### Agent side (TypeScript)

```ts
import {readFileSync} from 'node:fs';
import {A2uiGenerator, CatalogConfig, ExpressFormatFactory} from '@a2ui/agent';
import {Catalog} from '@a2ui/web_core';

const travelCatalog = Catalog.fromJson(JSON.parse(readFileSync('./catalogs/travel.json', 'utf8')));
const generator = new A2uiGenerator(
  [new CatalogConfig(travelCatalog)],
  undefined,
  new ExpressFormatFactory(),
);
```

The generator builds the LLM system prompt from the catalog and validates the model's messages. Python and Dart equivalents exist; see "Using a catalog in an agent" on a2ui.org.

### Renderer side (Flutter)

Register each component schema with a native widget (`ComponentImplementation`) and each local function with code (`FunctionImplementation`), build a `Catalog`, pass it to a `MessageProcessor`. React, Lit, Angular, SwiftUI, and Jetpack Compose follow the same shape; see "Registering a catalog in a renderer" on a2ui.org.

## Quickstart demo (restaurant finder, ~5 minutes)

Prerequisites: Node.js 18+, `uv`, a Gemini API key from Google AI Studio.

```sh
git clone https://github.com/google/a2ui.git
cd a2ui
export GEMINI_API_KEY=<key>
cd samples/client/lit && npm install && npm run demo:all   # http://localhost:5173
```

That launcher also starts the Python A2A agent (`samples/agent/adk/restaurant_finder`, `uv run .`, http://localhost:10002). Try: "Book a table for 2", "Find Italian restaurants near me". Do NOT paste a real production API key into a shared demo without reviewing the agent code first; the demo agent calls Gemini with your key.

## A2UI in MCP Apps

In hosts that support MCP Apps, the same A2UI surface can run inside an MCP App. The server returns A2UI content as a static payload (server-driven UI pattern) or streams it from an agent; `@a2ui/catalog-mcp` provides the MCP app components on web.

## Rules of thumb

- Start from the basic catalog; add ONE domain catalog for the app's own components. Keep the catalog small: every component the model can emit is surface the renderer must implement and the compiler must validate.
- Put layout guidance in the catalog `instructions` ("Place FlightStatusCard inside a Column. Format fares with formatCurrency.") instead of hand-writing prompts; the skill generator keeps prompts in sync when components change.
- Prefer data bindings over duplicated literals: one `$/outbound/pax` value drives every component that shows it, and local edits update all bound nodes without a round trip.
- Validate before transmitting: let the compiler reject unknown components, bad placement, and schema mismatches, and feed the error back to the model for self-correction.
- Functions that touch money, booking, or personal data belong on the agent side (v1.0 `callAgentFunction`); formatting and validation belong on the renderer.
- Protocol versions matter: v1.0 uses `@path` / `@call` / `@checks` / `@catalogId`; v0.9.1 uses bare `path` / `call` / `checks`. Match the version the renderer reports.
