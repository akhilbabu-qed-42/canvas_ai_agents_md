# Drupal Canvas & Canvas AI: Architecture Reference

All paths are relative to the repo root: `web/modules/contrib/canvas/`.

---

## 1. What is Canvas?

Drupal Canvas is a visual page-builder module for Drupal 10/11. It provides:
- A React-based in-browser editor (rendered inside an iframe) where editors assemble pages by placing components into regions and slots.
- A custom content entity (`canvas_page`) for standalone pages, plus a `content_template` config entity that attaches canvas layouts to existing Drupal content entities via view modes.
- A component model that unifies three different component sources — Single Directory Components (SDC), Drupal block plugins, and JavaScript (React/Preact) code components — under a single abstraction (`canvas_component` config entity).
- A versioned auto-save system so in-progress edits never block publishing.

---

## 2. Component Entities and Sources

### The `canvas_component` Config Entity
`src/Entity/Component.php` — wraps any component source. Key fields: `id`, `label`, `source` (plugin ID), `source_local_id`, `provider`, `active_version`, `versioned_properties`.

### Component Sources (Plugin System)
The `ComponentSource` plugin type (`src/ComponentSource/ComponentSourceInterface.php`) abstracts rendering and input resolution. Three concrete source plugins exist:

| Source | Plugin file | What it wraps |
|--------|-------------|---------------|
| `sdc` | `src/Plugin/Canvas/ComponentSource/SingleDirectoryComponent.php` | Drupal Single Directory Components (Twig + YAML props) |
| `block` | `src/Plugin/Canvas/ComponentSource/BlockComponent.php` | Any Drupal block plugin |
| `js_component` | `src/Plugin/Canvas/ComponentSource/JsComponent.php` | React/Preact code components (`js_component` entity) |

The `js_component` config entity (`src/Entity/JavaScriptComponent.php`) stores `source_code` (JS), `css`, and a JSON Schema (`schema`) that defines the component's props. These are React/Preact components built and edited entirely within the Canvas UI — their source lives in the `js_component` Drupal config, not in any file on disk.

A `Fallback` plugin (`src/Plugin/Canvas/ComponentSource/Fallback.php`) handles broken or missing sources gracefully.

### Component Trees and Slots
A component tree is stored as a multi-value field (`ComponentTreeItem` field type, `src/Plugin/Field/FieldType/ComponentTreeItem.php`). Each item carries: `uuid`, `component_id`, `inputs` (array of `PropSource` definitions), `parent_uuid`, `slot`, and `label`. The tree is hierarchical — a component placed inside a slot of another component references its parent via `parent_uuid` + `slot`.

`canvas_page` and `content_template` both carry a component tree field. `page_region` entities hold global regions (header/footer) that persist across pages.

### Prop Sources
How a prop's value is resolved at render time is determined by its `PropSource` type (`src/PropSource/`):

- `StaticPropSource` — hard-coded value
- `EntityFieldPropSource` — pulled from a Drupal entity field
- `HostEntityUrlPropSource` — URL of the host entity
- `AdaptedPropSource` — chains/transforms another prop source

---

## 3. Other Key Entities

| Entity type | Class | Purpose |
|-------------|-------|---------|
| `canvas_page` | `src/Entity/Page.php` | Standalone canvas page (content entity) |
| `content_template` | `src/Entity/ContentTemplate.php` | Attaches canvas layout to existing content via view mode |
| `js_component` | `src/Entity/JavaScriptComponent.php` | Stores JS/React component code and schema |
| `pattern` | `src/Entity/Pattern.php` | Reusable component presets |
| `page_region` | `src/Entity/PageRegion.php` | Global theme regions (header, footer) |
| `asset_library` | `src/Entity/AssetLibrary.php` | Shared CSS/font/image assets |
| `brand_kit` | `src/Entity/BrandKit.php` | Design tokens and brand styling |

---

## 4. Internal HTTP API

The UI communicates with Drupal exclusively through a custom REST API (no JSON:API or REST module). Routes are defined in `canvas.routing.yml`. Base path: `/canvas/api/v0/`.

Key endpoint groups:

| Group | Endpoints |
|-------|-----------|
| Config CRUD | `GET/PATCH/POST/DELETE /config/{entity_type}/{entity}` |
| Content CRUD | `GET/PATCH/POST/DELETE /pages/{entity_type}/{entity}` |
| Layout | `GET/PATCH/POST /layout/{entity_type}/{entity}` |
| Auto-save | `GET /auto-saves/pending`, `POST /auto-saves/publish`, `DELETE /auto-saves/{entity_type}/{entity}` |
| Component instance form | `GET/POST /form/component-instance/{entity_type}/{entity}` |
| Media / Artifacts | `POST /media/upload`, `POST /artifacts/upload` |
| Preview | Rendered via layout GET with preview flag |
| Notifications | `GET /notifications`, `POST /notifications/mark-as-read` |
| Usage tracking | `GET /usage/component`, `GET /usage/component/{id}` |

Controllers live in `src/Controller/`. Access is controlled by `_canvas_authentication_required` plus entity-level access checks.

---

## 5. UI Architecture (`ui/`)

The frontend is a React + Redux Toolkit application embedded as a Drupal library. Entry point and key structure:

```
ui/src/
├── app/
│   ├── store.ts              # Redux store (redux-undo on layout + pageData)
│   └── hooks.ts              # useAppDispatch, useAppSelector
├── features/
│   ├── layout/layoutModelSlice.ts        # Component tree state (RegionNode → ComponentNode → SlotNode)
│   ├── pageData/pageDataSlice.ts         # Page title, description, owner
│   ├── form/formStateSlice.ts            # Active form field state
│   ├── ui/uiSlice.ts                     # Panning, hover, route state
│   ├── ui/dialogSlice.ts                 # Modal/dialog state
│   ├── code-editor/codeEditorSlice.ts    # JS/CSS editor state
│   ├── notifications/notificationsSlice.ts
│   └── personalization/personalizationSlice.ts
├── services/
│   ├── baseQuery.ts                      # Fetch wrapper with auto-save awareness
│   ├── componentAndLayout.ts             # Layout read/write RTK Query service
│   ├── content.ts                        # canvas_page CRUD
│   ├── preview.ts                        # Preview rendering
│   ├── assetLibrary.ts / brandKit.ts     # Asset management
│   ├── patterns.ts                       # Pattern CRUD
│   └── notificationsApi.ts              # Polling for backend notifications
├── components/
│   ├── aiExtension/AiWizard.tsx          # Chat UI; parses tool outputs to drive UI mutations
│   └── ...                              # 20+ other UI component subdirectories
└── types/                               # TypeScript types (Component.ts, Form.ts, Content.ts, etc.)
```

### Layout Model Shape
The Redux `layoutModel` slice represents the page as a tree:
```
RootLayoutModel {
  layout: RegionNode[]     // top-level regions (e.g. "content", "header")
  model: ComponentModels   // flat map of uuid → component metadata
}

RegionNode → ComponentNode[] → SlotNode[] → ComponentNode[] (recursive)
```

Undo/redo is handled by `redux-undo` wrapping the `layoutModel` and `pageData` slices. The store uses a "history eraser" middleware pattern to keep both slices in sync across undo steps.

---

## 6. Canvas AI Sub-Module

Canvas AI (`modules/canvas_ai/`) is a sub-module that adds conversational AI to the editor. It is built on the **Drupal AI Agents** module.

### How it works
1. The user types in the chat UI (`ui/src/components/aiExtension/AiWizard.tsx`).
2. `modules/canvas_ai/src/Controller/CanvasBuilder.php` receives the request, reconstructs message history, builds Drupal context (entity type, IDs, selected component UUID, current layout), and invokes the orchestrator agent.
3. The orchestrator delegates to specialized sub-agents. Sub-agents are stateless — they receive only a specific task prompt from the orchestrator, not the full chat history.
4. Each agent calls `AiFunctionCall` tool plugins to read or mutate Drupal state.
5. Tool outputs are returned to `AiWizard.tsx` as structured JSON, which triggers layout mutations and UI updates directly.

### Agents

All agent system prompts, tool lists, and configuration are in:
`modules/canvas_ai/config/install/`

| Agent ID | Config file | Responsibility |
|----------|-------------|----------------|
| `canvas_ai_orchestrator` | `ai_agents.ai_agent.canvas_ai_orchestrator.yml` | Sole agent with full chat history; routes intent to sub-agents |
| `canvas_component_agent` | `ai_agents.ai_agent.canvas_component_agent.yml` | Creates/edits JS (React/Preact) code components |
| `canvas_page_builder_agent` | `ai_agents.ai_agent.canvas_page_builder_agent.yml` | Adds components incrementally to an existing `canvas_page` or `content_template` |
| `canvas_template_builder_agent` | `ai_agents.ai_agent.canvas_template_builder_agent.yml` | Builds a complete page template (all regions) from scratch |
| `canvas_title_generation_agent` | `ai_agents.ai_agent.canvas_title_generation_agent.yml` | Writes SEO-friendly page titles to the `canvas_page` entity |
| `canvas_metadata_generation_agent` | `ai_agents.ai_agent.canvas_metadata_generation_agent.yml` | Writes SEO meta descriptions to the `canvas_page` entity |

### Page Builder and Template Builder: component selection
Both agents operate on `canvas_component` entities (SDC, block, or JS components). They do not see components visually — they reason purely from component metadata: props, slots, and descriptions configured at `Configuration → AI → Canvas AI Component Description Settings`. Agents call `get_component_context` to retrieve this metadata before deciding what to place on the page.

Title and metadata agents work exclusively on `canvas_page` entities via `get_entity_information`, `edit_field_content`/`create_field_content`, and `add_metadata` tools.

### AI-generated YAML: structure and rules

Both the page builder and template builder agents are instructed (in their system prompts) to produce YAML. The exact formats are:

**Page builder — `set_component_structure` input:**
```yaml
operations:
  - target: "content"              # region name OR "parent-uuid/slot_name" for a slot
    reference_uuid: "abc-123"      # UUID of anchor component; omit when placement is "inside"
    placement: above               # inside | above | below
    components:
      - sdc.my_hero:
          props:
            title: "Welcome"
            image:
              src: /path/from/catalog   # MUST be copied exactly from catalog defaults
              alt: "Alt text"
              width: 601
              height: 402
          slots:
            content:
              - sdc.button:
                  props:
                    label: "Get started"
  - target: "abc-123/footer_slot"  # second operation for a different location
    placement: inside
    components:
      - sdc.link: ...
```

Rules enforced by the system prompt: no null values, no empty strings, no empty slots, no invented component IDs, image props must copy exact defaults from catalog (src/alt/width/height unchanged), enum props must use catalog values only.

**Template builder — `set_template_data` input:**
```yaml
content:                           # region name; only "content" unless user asks for header/footer
  - sdc.hero_banner:
      props:
        title: "Welcome"
      slots:
        slot_name:
          - sdc.button:
              props:
                label: "Shop now"
  - sdc.features_grid:
      props:
        heading: "Why choose us"
header:                            # only present if theme has a header region and user requests it
  - sdc.site_header:
      props: ...
```

Rules: minimum five sections in the content region, no comments in YAML, production-quality copy (no Lorem Ipsum), only catalog-sourced component IDs.

### `CanvasAiPageBuilderHelper`: the core canvas_ai service

`modules/canvas_ai/src/CanvasAiPageBuilderHelper.php` is the central service that bridges raw AI YAML output and the Canvas UI layout format. It handles both the page builder and template builder flows.

**Key responsibilities and public methods:**

| Method | What it does |
|--------|-------------|
| `customYamlToArrayMapper(string $yaml)` | Entry point for page builder: parses AI YAML → adds UUIDs → merges with current layout → extracts `nodePath` for each component → returns operations array |
| `processSetTemplateDataToolInput(array $parsed, string $layoutJson)` | Entry point for template builder: same nodePath calculation over a full region map |
| `getComponentContextForAi()` | Builds the YAML catalog of all available components (props, slots, descriptions) that agents receive via `get_component_context` |
| `getAllComponentsKeyedBySource()` | Fetches SDC, JS, and block components with caching; used to build catalog |
| `getAvailableRegions(string $layoutJson)` | Returns region names + nodePathPrefix + descriptions from config; used to inform agents what regions exist |
| `createExpectedPageLayout(array $current, array $aiOutput)` | Simulates placing AI components into the current layout to predict the post-save layout tree |
| `hasChildComponents(string $target)` | Checks whether a region or slot already has children; used to decide `inside` vs `above`/`below` |

**Transformation pipeline (page builder):**
1. Parse YAML string from AI tool call.
2. `addUuidToAllComponents()` — assign a fresh UUID to every component in the AI output.
3. Load current layout from `tempstore`.
4. `createExpectedPageLayout()` — merge the AI components into the current layout at the instructed positions (`placeComponentsInLayout()` handles `inside`/`above`/`below` logic).
5. `getCalculatedNodepath()` — walk the predicted layout tree to compute the `nodePath` array (positional indices used by the Canvas UI to locate a node in the tree) for each new component, including slot index resolution via `getSlotIndexFromSlotName()`.
6. Return `{ operations: [{ operation: 'ADD', components: [{ id, nodePath, fieldValues }] }] }` — the format the Canvas UI (`AiWizard.tsx`) consumes to apply mutations to the Redux `layoutModel`.

### `AiResponseValidator`: YAML validation before execution

`modules/canvas_ai/src/AiResponseValidator.php` validates AI-generated component structures before any layout mutation is applied.

**What it does:**
- `validateComponentStructure(array $componentGroups)` converts the AI output to `ComponentTreeItem` format via `convertToComponentTreeData()`, attaches it to a dangling `ComponentTreeItemList` field, and runs Drupal's recursive constraint validator on it.
- Checks: component existence, version correctness, prop/input validity (via `Component::normalizeForClientSide()` and `source->clientModelToInput()`), and parent-child slot relationships.
- For components that don't yet exist in the system (e.g., a just-created `js_component`), it assigns a temporary version (`temp-version-{uuid}`) so validation can still proceed on everything else.
- On failure: throws `ConstraintViolationException` with translated property paths pinpointing the exact violation (e.g., `components.0.[component_id].props.prop_name`).
- On success: returns silently. The calling tool proceeds to pass the validated structure to `CanvasAiPageBuilderHelper` for nodePath calculation and UI dispatch.

---

## 7. Quality Gates (Mandatory After Every Code Change)

Run both checks on any modified PHP file before considering a change done.

```
# PHPStan (adjust the file path for the file you changed)
ddev exec ./vendor/bin/phpstan analyze \
  --configuration=web/modules/contrib/canvas/phpstan.neon \
  --autoload-file=web/modules/contrib/canvas/phpstan_rules/autoload.php \
  web/modules/contrib/canvas/<path to changed file>

# PHPCS
ddev exec ./vendor/bin/phpcs \
  --standard=web/modules/contrib/canvas/phpcs.xml \
  --no-cache \
  web/modules/contrib/canvas/<path to changed file>
```

---

## 8. Comment and Documentation Rules

Doc comments and inline comments must describe **what the code does right now** — nothing else. Do not reflect conversation history, prior iterations, design rationale from chat, or "we decided to…" narratives. A reader opening the file cold should learn the current behavior, not the journey that produced it. Future work, deferred decisions, or known gaps must be captured as proper `@todo` tags (one per item, with a concrete action), not as prose buried in a class or function docblock.

---

*Module: `canvas_ai` | Status: Active*
