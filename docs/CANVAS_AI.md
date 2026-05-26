# Canvas AI — Developer Reference

> All paths relative to `web/modules/contrib/canvas/`

---

## What canvas_ai is

A conversational AI editor built on `drupal/ai_agents`. The user types in the chat UI (`ui/src/components/aiExtension/AiWizard.tsx`); agents read and mutate Drupal state via tool plugins; structured JSON tool outputs drive Redux layout mutations in the editor — no page reload.

---

## Dependency stack — understand before touching anything

```
canvas_ai
  └── drupal/ai_agents      web/modules/contrib/ai_agents/
        └── drupal/ai       web/modules/contrib/ai/
```

**`drupal/ai` (`web/modules/contrib/ai/`)** — the base AI module. Provides:
- The `AiFunctionCall` plugin type. All tool plugins that agents call (e.g. `GetComponentContext`, `SetComponentStructure`) are `AiFunctionCall` plugins.
- `canvas_ai` defines its own tool plugins at `modules/canvas_ai/src/Plugin/AiFunctionCall/`. These extend base classes from `drupal/ai` — do not reimplement plugin infrastructure, extend what exists.

**`drupal/ai_agents` (`web/modules/contrib/ai_agents/`)** — the agent execution layer. Provides:
- The `ai_agent` config entity type. All Canvas agents (`canvas_ai_orchestrator`, `component_agent`, etc.) are `ai_agent` config entities — their system prompts, tool lists, and settings live in `modules/canvas_ai/config/install/ai_agents.ai_agent.*.yml`.
- The orchestration loop that invokes agents, dispatches tool calls, and returns results.
- `canvas_ai` does not reimplement agent execution — it configures agents via YAML and invokes the orchestrator via `CanvasBuilder.php`.

**YOU MUST** inspect both upstream modules before adding new tools or modifying agents. The plugin base classes, interfaces, and registration patterns are defined there — do not guess or reinvent them.

| File | Read when… |
|---|---|
| `web/modules/contrib/ai_agents/src/PluginBase/AiAgentEntityWrapper.php` | Touching any agent execution logic — this is the primary file handling execution for all ai agents |
| `web/modules/contrib/ai/src/Plugin/AiFunctionCall/` | Adding or modifying any tool plugin — check base classes and interfaces here first |

---

## Request flow

1. User types in `AiWizard.tsx`
2. `src/Controller/CanvasBuilder.php` receives the request, reconstructs message history, builds Drupal context (entity type, IDs, selected component UUID, current layout), invokes the orchestrator agent
3. Orchestrator delegates to a sub-agent with a focused task prompt — **sub-agents are stateless and never receive full chat history**
4. Sub-agent calls `AiFunctionCall` tool plugins to read or mutate Drupal state
5. Tool outputs return to `AiWizard.tsx` as structured JSON → triggers Redux `layoutModel` mutations

---

## Agents

Config lives in `config/install/`. All agent system prompts, tool lists, and settings are there.

| Agent ID | Config file | Responsibility |
|---|---|---|
| `canvas_ai_orchestrator` | `ai_agents.ai_agent.canvas_ai_orchestrator.yml` | Only agent with full chat history; routes intent to sub-agents |
| `component_agent` | `ai_agents.ai_agent.component_agent.yml` | Creates/edits JS (React/Preact) code components |
| `canvas_page_builder_agent` | `ai_agents.ai_agent.canvas_page_builder_agent.yml` | Adds components incrementally to an existing `canvas_page` or `content_template` |
| `canvas_template_builder_agent` | `ai_agents.ai_agent.canvas_template_builder_agent.yml` | Builds a complete page template (all regions) from scratch |
| `canvas_title_generation_agent` | `ai_agents.ai_agent.canvas_title_generation_agent.yml` | Writes SEO-friendly titles to `canvas_page` |
| `canvas_metadata_generation_agent` | `ai_agents.ai_agent.canvas_metadata_generation_agent.yml` | Writes SEO meta descriptions to `canvas_page` |

Page builder and template builder reason purely from component metadata (props, slots, descriptions) — they never see components visually. They call `get_component_context` to retrieve this metadata before deciding what to place.

Title and metadata agents work exclusively on `canvas_page` entities via `get_entity_information`, `edit_field_content`/`create_field_content`, and `add_metadata` tools.

---

## AI-generated YAML formats

### Page builder — `set_component_structure` input

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
  - target: "abc-123/footer_slot"
    placement: inside
    components:
      - sdc.link: ...
```

**Rules enforced by the system prompt:** no null values, no empty strings, no empty slots, no invented component IDs, image props must copy exact defaults from catalog (src/alt/width/height unchanged), enum props must use catalog values only.

### Template builder — `set_template_data` input

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
header:                            # only if theme has a header region and user requests it
  - sdc.site_header:
      props: ...
```

**Rules:** minimum five sections in the content region, no comments in YAML, production-quality copy (no Lorem Ipsum), only catalog-sourced component IDs.

---

## `CanvasAiPageBuilderHelper` — the central service

`src/CanvasAiPageBuilderHelper.php` bridges raw AI YAML and the Canvas UI layout format. Handles both page builder and template builder flows.

| Method | What it does |
|---|---|
| `customYamlToArrayMapper(string $yaml)` | Entry point for page builder: parses AI YAML → adds UUIDs → merges with current layout → calculates `nodePath` for each component → returns operations array |
| `processSetTemplateDataToolInput(array $parsed, string $layoutJson)` | Entry point for template builder: same nodePath calculation over a full region map |
| `getComponentContextForAi()` | Builds the YAML catalog of all available components (props, slots, descriptions) sent to agents via `get_component_context` |
| `getAllComponentsKeyedBySource()` | Fetches SDC, JS, and block components with caching; used to build catalog |
| `getAvailableRegions(string $layoutJson)` | Returns region names + nodePathPrefix + descriptions; used to inform agents what regions exist |
| `createExpectedPageLayout(array $current, array $aiOutput)` | Simulates placing AI components into the current layout to predict the post-save layout tree |
| `hasChildComponents(string $target)` | Checks whether a region or slot already has children; determines `inside` vs `above`/`below` |

### Page builder transformation pipeline

1. Parse YAML string from AI tool call
2. `addUuidToAllComponents()` — assign a fresh UUID to every component in the AI output
3. Load current layout from `tempstore`
4. `createExpectedPageLayout()` — merge AI components into the current layout at the instructed positions (`placeComponentsInLayout()` handles `inside`/`above`/`below` logic)
5. `getCalculatedNodepath()` — walk the predicted layout tree to compute the `nodePath` array (positional indices used by the Canvas UI to locate a node in the tree) for each new component, including slot index resolution via `getSlotIndexFromSlotName()`
6. Return `{ operations: [{ operation: 'ADD', components: [{ id, nodePath, fieldValues }] }] }` — consumed by `AiWizard.tsx` to apply mutations to the Redux `layoutModel`

---

## `AiResponseValidator` — YAML validation before execution

`src/AiResponseValidator.php` validates AI-generated component structures before any layout mutation is applied.

- `validateComponentStructure(array $componentGroups)` converts AI output to `ComponentTreeItem` format via `convertToComponentTreeData()`, attaches it to a dangling `ComponentTreeItemList` field, and runs Drupal's recursive constraint validator
- Checks: component existence, version correctness, prop/input validity (via `Component::normalizeForClientSide()` and `source->clientModelToInput()`), and parent-child slot relationships
- For components that don't yet exist (e.g. a just-created `js_component`), assigns a temporary version (`temp-version-{uuid}`) so validation can proceed on everything else
- On failure: throws `ConstraintViolationException` with translated property paths pinpointing the exact violation (e.g. `components.0.[component_id].props.prop_name`)
- On success: returns silently; the calling tool passes the validated structure to `CanvasAiPageBuilderHelper` for nodePath calculation and UI dispatch