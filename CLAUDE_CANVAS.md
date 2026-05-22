# Drupal Canvas — Contributor Reference

This is a comprehensive map of the Drupal Canvas module for working on the codebase. All paths are relative to this directory (`web/modules/contrib/canvas/`). Drupal Canvas requires **Drupal core ≥ 11.2** and PHP 8.3.

> "Canvas is visual-first": page = a component tree first, data wired in second. This inverts Drupal's traditional data-first model.

---

## 1. What Canvas is

A React + Redux page builder for Drupal that lets non-developers compose pages in the browser. It provides:

- A **custom content entity** `canvas_page` for standalone pages.
- A **`content_template` config entity** that attaches Canvas layouts to existing content entities by entity type + bundle + view mode (currently only `Node`).
- A **unified component model** wrapping three sources (SDC, block plugins, JS code components) behind a single `component` config entity.
- A **shape-matching system** that lets component props be wired to Drupal entity fields with type safety, or filled with ad-hoc "static" data.
- A **versioned auto-save system** so in-progress edits don't block publishing.
- An **AI sub-module** (`canvas_ai`) — see `modules/canvas_ai/`.

---

## 2. Documentation map (read these first when working in an area)

Authoritative docs live in `docs/`. The hierarchy:

| Doc | Topic | Read when working on… |
|-----|-------|----------------------|
| `docs/intro.md` | 30,000-ft overview | New contributors |
| `docs/components.md` | Component sources (SDC/Block/JS), eligibility criteria, Fallback | Component source plugins |
| `docs/data-model.md` | `ComponentTreeItem` field type, layout/model UI shape, versioning | Field types, validation, updaters |
| `docs/config-management.md` | All Canvas config entities (`Component`, `JavaScriptComponent`, `PageRegion`, `Pattern`, `ContentTemplate`, `Folder`, `BrandKit`, `StagedConfigUpdate`) | Config entities & config schema |
| `docs/shape-matching.md` | `PropShape`, `PropSource`, matchers, `PropSourceSuggester`, prop expressions | Prop binding & shape matching |
| `docs/redux-integrated-field-widgets.md` | How Drupal Form API field widgets are synced into Redux for real-time preview | Field widget integration & transforms |
| `docs/twig-to-react-in-canvas-stark.md` | `canvas_stark` admin theme: custom HTML elements → React via Hyperscriptify | Custom widget rendering |
| `docs/notifications.md` | Notification system + REST API (gated by `canvas_dev_mode`) | Notifications |
| `docs/personalization.md` | Segment / Variant / Switch / Case model | `canvas_personalization` submodule |
| `docs/react-codebase/page-preview.md` | `<iframe>` preview, IFrameSwapper, overlay UI | Preview rendering / UI overlay |
| `docs/react-codebase/code-editor.md` | In-browser JS/CSS code component compiler pipeline (SWC, Lightning CSS, Tailwind) | Code-component editor |
| `docs/react-codebase/import-sorting.md` | Import order rules | React code edits |
| `docs/testing/{setup,phpunit,playwright,static,gitlab-ci}.md` | Test infrastructure | Writing tests |
| `docs/release-process.md` | Tagging, hotfix branches | Releases |
| `docs/adr/*` | Architectural Decision Records (11 ADRs) | Big-picture decisions |
| `API.md`, `openapi.yml` | HTTP API contract | API changes |

Each domain doc also points at the Drupal.org issue queue *component* and `CODEOWNERS` entries — use those to file/track issues.

### Key ADRs

- **ADR-2**: SDC slots are the basis of the component tree; field types populate SDC props.
- **ADR-3**: Auto-generate a `Component` config entity for every eligible SDC.
- **ADR-4**: `canvas_page` is a content entity.
- **ADR-5**: Keep the front end simple (data shaping happens server-side).
- **ADR-6**: One field row per component instance (no compound field-item storage).
- **ADR-7/8**: Bundle import-map deps; mark them external when re-imported from nested deps.
- **ADR-9**: Extensions API architecture.
- **ADR-10**: Dynamic config schema per component instance for translatability.
- **ADR-11**: New `content-entity-reference` prop shape for code components.

---

## 3. Backend (PHP) architecture

### 3.1 Top-level source layout (`src/`)

| Dir / file | Purpose |
|---|---|
| `Entity/` | Content + config entities (`Page`, `ContentTemplate`, `Component`, `JavaScriptComponent`, `PageRegion`, `Pattern`, `Folder`, `BrandKit`, `AssetLibrary`, `StagedConfigUpdate`, plus `VersionedConfigEntityBase`/Interface) |
| `Controller/` | All HTTP API controllers (`Api*Controllers.php`) and admin/UI controllers (`CanvasController`, `ComponentAuditController`, etc.) |
| `ComponentSource/` | Interfaces & manager for the component source plugin type (`ComponentSourceInterface`, `ComponentSourceManager`, updater + slot-aware variants) |
| `Plugin/Canvas/ComponentSource/` | The four concrete sources: `SingleDirectoryComponent`, `BlockComponent`, `JsComponent`, `Fallback`, plus `GeneratedFieldExplicitInputUxComponentSourceBase` (the reusable infra for sources without their own input UX) |
| `Plugin/Field/FieldType/ComponentTreeItem.php` | The Canvas field type that stores a component tree |
| `Plugin/DataType/` | `ComponentTreeStructure`, `ComponentInputs`, low-level data types |
| `Plugin/Validation/Constraint/` | All Canvas validation constraints (component existence, prop shape storability, JS component metadata, etc.) |
| `Plugin/DataTypeOverride/` | Overrides for core data types (e.g. `UriOverride`) |
| `PropShape/` | `PropShape`, `StorablePropShape`, `CandidateStorablePropShape`, repositories |
| `PropSource/` | `StaticPropSource`, `EntityFieldPropSource`, `HostEntityUrlPropSource`, `AdaptedPropSource` |
| `PropExpressions/StructuredData/` | The prop-expression DSL (`FieldPropExpression`, `ReferenceFieldPropExpression`, `Evaluator`, `Labeler`, …) |
| `ShapeMatcher/` | `EntityFieldPropSourceMatcher`, `HostEntityUrlPropSourceMatcher`, `AdaptedPropSourceMatcher`, `PropSourceSuggester` |
| `JsonSchemaInterpreter/` | Translates SDC JSON schema → primitive `data type` + validation constraints (`JsonSchemaType::computeStorablePropShape()`, `toDataTypeShapeRequirements()`) |
| `AutoSave/` | The auto-save manager and storage |
| `Audit/` | `ComponentAudit` (where is a component used?), used by config dependency removal |
| `Config/` | `CanvasConfigOverrides` (auto-save → live config overlay), `CanvasConfigUpdater` |
| `Form/` | Server-side forms (`ComponentInstanceForm`, page forms) |
| `Hook/` | OOP hook handler classes (per Drupal 11 hook attribute system) |
| `EntityHandlers/` | Custom storage handlers (e.g. `JavascriptComponentStorage`, `StagedConfigUpdateStorage`) |
| `EventSubscriber/` | Route & request subscribers |
| `Render/` | `CanvasTemplateRenderer`, `CanvasPreviewRenderer`, main-content renderers |
| `Theme/` | Theme engine integration |
| `Twig/` | Twig extensions used by Canvas + `canvas_stark` theme |
| `Validation/`, `TypedData/` | Misc validation infra |
| `Resource/` | Internal HTTP API resource helpers |

### 3.2 Component sources

Plugin type `component_source` (manager: `src/ComponentSource/ComponentSourceManager.php`). To add a new source: implement `ComponentSourceInterface` (or extend `ComponentSourceBase` / `GeneratedFieldExplicitInputUxComponentSourceBase`). Sources without their own input UX **must** produce an SDC `ComponentMetadata` so shape-matching can generate a field-based input UX — see `JsComponent::buildEphemeralSdcPluginInstance()` for the canonical example.

Eligibility checks: `ComponentMetadataRequirementsChecker` (SDC), `BlockComponent::checkRequirements()` (blocks). Failures are stored in `ComponentIncompatibilityReasonRepository` and surfaced in `/admin/appearance/component/status`.

Fallback flow: when a config dependency is removed, `Component::onDependencyRemoval()` consults `ComponentAudit`; if the component is in use, the source is swapped to `Fallback` and `fallback_metadata` is used so existing trees keep rendering — see `docs/components.md` §3.4.

### 3.3 The component tree (`ComponentTreeItem`)

Each row in a Canvas field is one component instance with six stored field props:

`uuid · component_id · component_version · parent_uuid · slot · inputs`

…plus three computed props (`component`, `parent_item`, `inputs_resolved`). Top-level instances have null `parent_uuid`/`slot`; deltas determine sibling ordering. The same shape is reused inside config entities (with `inputs` as a JSON string in YAML).

Translatability: the `inputs` column is always translatable; the tree column group is configurable as translatable/untranslatable for asymmetric vs. symmetric translations. Source plugins may declare an `inputs_config_schema_generator` (see `ComponentInstanceInputsConfigSchemaGeneratorInterface`) — single source of truth for which input keys translate (used by both `content_translation` and `config_translation`).

Validation: `\Drupal\canvas\Plugin\Validation\Constraint\ComponentTreeStructureConstraint` + per-instance `ComponentSourceInterface::validateComponentInput()`.

### 3.4 UI data model on the wire

`/canvas/api/v0/layout/...` returns `{ layout, model }` where `layout` is a tree of three node types (`region`, `component`, `slot`) and `model` is a flat UUID-keyed map of resolved input values. See `docs/data-model.md` §3.4 for the full schema. The `content` region is special-cased server-side. Server-side validation refuses to save a tree where `model` and `layout` are out of sync.

### 3.5 Shape matching (the heart of "wire-a-field-to-a-prop")

Two phases:

1. **Matchers (objective)** — `EntityFieldPropSourceMatcher`, `HostEntityUrlPropSourceMatcher`, `AdaptedPropSourceMatcher` independently return *all* prop sources of their type whose shape fits a given `PropShape`.
2. **Suggester (subjective)** — `PropSourceSuggester` filters out irrelevant fields (revision log message, default-translation flag, …) and reorders by "obviousness" before presenting to the user.

`StaticPropSource` (for unstructured data) is computed via `JsonSchemaType::computeStorablePropShape()`. Modules may implement `hook_canvas_storable_prop_shape_alter()` to change the chosen field type/widget/storage settings.

**Prop expressions** are the low-level pointer DSL: `ℹ︎␜entity:node:article␝title␞99␟value`. Two representations (string ↔ typed PHP object). See `PropExpressions/StructuredData/` and `PropExpressionTest`. You only need these when implementing the alter hook above or a new prop source.

Canvas extends SDC's JSON Schema with:

- `contentMediaType: text/html` + `x-formatting-context: inline|block` → CKEditor 5 props.
- `contentMediaType: image/*` + `x-allowed-schemes: [http, https]` → constrained URI props.
- `$ref: json-schema-definitions://canvas.module/content-entity-reference` + `x-allowed-entity-type-id` / `x-allowed-bundle` → entity-reference props (code components only today).

### 3.6 HTTP API

Base path `/canvas/api/v0/`. Defined in `canvas.routing.yml`. Controllers in `src/Controller/Api*Controllers.php`. **Canvas does not use JSON:API or REST module** — see `docs/config-management.md` §3 for the rationale. Access uses `_canvas_authentication_required` + entity-level access. Routes flagged `canvas_external_api: true` are intended for third-party use; the optional `canvas_oauth` submodule provides OAuth2 for them.

Endpoint groups (see `API.md`/`openapi.yml` for full contract):

- Config CRUD: `/config/{entity_type}/{entity}` (GET/PATCH/POST/DELETE)
- Content CRUD: `/pages/{entity_type}/{entity}`
- Layout: `/layout/{entity_type}/{entity}`
- Auto-save: `/auto-saves/pending`, `/auto-saves/publish`, `/auto-saves/{entity_type}/{entity}`
- Component instance form: `/form/component-instance/{entity_type}/{entity}`
- Media / Artifacts: `/media/upload`, `/artifacts/upload`
- Notifications: `/notifications`, `/notifications/read`
- Usage tracking: `/usage/component[/{id}]`
- Preview: served via the layout GET with a preview flag
- Extensions: `/extensions/*`

### 3.7 Auto-save & "Publish all"

Most Canvas mutations land in auto-save (a per-user/per-entity overlay) rather than active storage. Auto-save is owned by `src/AutoSave/`, exposed via the routes above. `CanvasConfigOverrides` (a config override) makes auto-save data win on Canvas-internal routes so previews look like the working copy. `JavaScriptComponent` and `PageRegion` rely on this overlay for the "Working copy vs Published" distinction (see `docs/config-management.md` §3.2.1).

`StagedConfigUpdate` is the special case: a config entity that *only* exists in auto-save; on publish its `actions` array is applied to a target config object.

### 3.8 Versioning

Both `Component` and `JavaScriptComponent` are *versioned* config entities (`VersionedConfigEntityBase`). Each version has a deterministic hash. Stored component instances reference a *specific* version. Updaters (`ComponentInstanceUpdaterInterface`, with `ComponentInstanceUpdateAttemptResult`) decide whether a stale instance can be auto-updated:

- **Safe** (auto-update): add/remove optional props, add/remove slots, optional↔required toggles, widget-only changes, default/example changes.
- **Unsafe** (manual intervention): changing a prop shape.

Auto-update fires on entering an editor or rendering a preview — see `ApiLayoutController::buildRegion()` and `ApiConfigControllers::normalize()`.

### 3.9 Page rendering

A `ContentTemplate` is consulted on render only when (1) one exists for this entity type + bundle + view mode and (2) the entity type is `Node` (current limitation). When used, `hook_entity_display_build_alter()` is **bypassed** — Canvas does not render through field formatters.

For theme regions (header/footer/etc.), once a theme has at least one *enabled* `PageRegion`, Canvas's `CanvasPageVariant` PageDisplayVariant takes over from the Block module's `BlockPageVariant`.

### 3.10 Hooks worth knowing

- `hook_canvas_storable_prop_shape_alter()` — change how unstructured data is stored for a prop shape.
- `hook_field_widget_info_alter()` with `$info[...]['canvas']['transforms']` — declare how a widget's form values map to a prop value (see `canvas.redux_integrated_field_widgets.inc`).
- `hook_library_info_alter()` is leveraged by Canvas to attach any `canvas.transform.*` libraries to the UI automatically.

See `canvas.api.php` for the documented hook surface.

---

## 4. Frontend (React + Redux) — `ui/`

### 4.1 Tech stack

- **React + TypeScript**, built with the Canvas build pipeline (`npm run build` in `ui/`).
- **Redux Toolkit** + **RTK Query** for state and server I/O.
- **redux-undo** wraps `layoutModel` and `pageData` for undo/redo. A "history eraser" middleware keeps both slices in sync across undo steps.
- **Radix UI Themes** for component primitives.
- **Hyperscriptify** (`ui/src/utils/hyperscriptify.js`, local fork) renders custom HTML elements emitted by the `canvas_stark` theme as React components — see `docs/twig-to-react-in-canvas-stark.md`.

### 4.2 Source layout (`ui/src/`)

```
app/
  store.ts              # Redux store (combined slices + RTK Query APIs, undo wrapper)
  hooks.ts              # typed useAppDispatch / useAppSelector
features/               # Redux slices, one folder per domain
  layout/layoutModelSlice.ts    # The component tree (RegionNode → ComponentNode → SlotNode)
  pageData/pageDataSlice.ts     # Title, description, owner
  form/formStateSlice.ts        # Active form field state (Redux-integrated widgets)
  ui/uiSlice.ts                 # Pan/hover/route/selection (shared across viewports + Layers panel)
  ui/dialogSlice.ts
  code-editor/codeEditorSlice.ts
  notifications/notificationsSlice.ts
  personalization/personalizationSlice.ts
  pagePreview/previewSlice.ts
  configuration/, editor/, editorFrame/, error-handling/, extensions/, pattern/, validation/, welcome/, brandKit/
services/               # RTK Query endpoints
  baseQuery.ts                  # Fetch wrapper aware of auto-save semantics
  componentAndLayout.ts         # Layout GET/PATCH
  content.ts                    # canvas_page CRUD
  preview.ts, assetLibrary.ts, brandKit.ts, patterns.ts
  notificationsApi.ts           # adaptive polling
components/             # 20+ UI component subdirectories
  form/                 # Form.tsx, inputBehaviors.tsx, twig-to-jsx-component-map.js
  aiExtension/          # AiWizard.tsx (chat UI for canvas_ai)
types/                  # TS types: Component.ts, Form.ts, Content.ts, …
utils/
  transforms.ts                 # Built-in widget transforms (mainProperty, dateTime, etc.)
  hyperscriptify.js
```

### 4.3 Layout model (Redux)

```
RootLayoutModel {
  layout: RegionNode[]      // top-level regions (content, header, …)
  model:  ComponentModels   // flat UUID → resolved input map
}

RegionNode → ComponentNode[] → SlotNode[] → ComponentNode[] … (recursive)
```

Each `ComponentNode.type` is `<component_id>@<version_hash>`. Slot IDs are `<parentUuid>/<slotName>`.

### 4.4 Redux-integrated field widgets — how Drupal Form API forms get into React

Server-rendered Drupal form arrays are emitted by the **`canvas_stark`** admin theme as custom HTML elements (`<drupal-canvas-form>`, `<drupal-canvas-input>`, etc.). Hyperscriptify converts them into React components via the map in `ui/src/components/form/twig-to-jsx-component-map.js`.

Each form-control React component is wrapped by `inputBehaviors` (`ui/src/components/form/inputBehaviors.tsx`), which:

- Updates the Redux `formState` slice on change (and updates the preview).
- Runs client-side JSON-Schema validation against the prop's schema.
- Applies **transforms** (`ui/src/utils/transforms.ts`) to reshape form values into the canonical prop value before dispatching.

Built-in transforms: `mainProperty`, `firstRecord`, `mediaSelection`, `dateTime`, `dateRange`. Custom transforms ship as JS libraries with the `canvas.transform.` prefix and register on `Drupal.canvasTransforms`. Multi-cardinality fields get `multiple: true` automatically; weight-aware ordering kicks in when entries carry `weight`/`_weight`.

The `data-canvas-no-update` attribute opts an input out of preview/store updates (useful for autocomplete intermediates).

### 4.5 Preview architecture

```
Preview.tsx
└─ Viewport.tsx (one per breakpoint)
   ├─ IframeSwapper.tsx (two iframes A/B for flicker-free swaps)
   └─ ViewportOverlay.tsx (portalled above the editor frame; mirrors regions/components/slots)
```

The overlay is a transparent **mirror** of the iframe DOM, sized via ResizeObserver/MutationObserver. All interaction (hover, click, drag, right-click) is handled in the overlay, never in the iframe. The server annotates rendered HTML with comments marking each region/component/slot; `useComponentHtmlMap` builds the UUID → element map on every preview refresh.

### 4.6 Code-component compiler

In-browser editor for `js_component` entities. Per-component source JS is compiled with **SWC**, source CSS with **Lightning CSS**. The global asset library's Tailwind config is compiled with the **Tailwind CSS** compiler whenever any code-component's source JS changes (so utility class indexing stays current). See `docs/react-codebase/code-editor.md` for the flowchart.

---

## 5. Sub-modules (`modules/`)

| Module | Purpose |
|---|---|
| `canvas_ai` | Conversational AI editor, built on `drupal/ai_agents`. Sub-agents: orchestrator + component / page-builder / template-builder / title / metadata. YAML-driven; central service `CanvasAiPageBuilderHelper` translates AI YAML → layout operations consumed by `AiWizard.tsx`. See root `CLAUDE.md` for details |
| `canvas_dev_mode` | Gates dev-only features (notifications system, debug helpers) |
| `canvas_dev_translation` | Test-only translation helpers |
| `canvas_oauth` | OAuth2 for external-facing API routes (`canvas_external_api: true`) |
| `canvas_personalization` | Segment / Variant / `switch` + `case` components (see `docs/personalization.md`) |
| `canvas_vite` | Optional Vite-based asset pipeline |

---

## 6. Theme: `canvas_stark`

A dedicated admin theme (`themes/canvas_stark/`) used to render Drupal Form API output in a React-friendly way. It renders form-control elements as custom HTML elements (`<drupal-canvas-input>`, `<drupal-canvas-textarea>`, …) which Hyperscriptify (in the UI) converts to React. See `docs/twig-to-react-in-canvas-stark.md`. Naming: `drupal-canvas-*` for elements originating from Drupal render arrays, `canvas-*` for those originating in Twig.

---

## 7. Coding standards & conventions

### 7.1 PHP

- **Drupal coding standards** (`phpcs.xml` extends core's), with a relaxed docblock requirement (no enforced boilerplate yet — but you still must doc *what* code does, not its history).
- **PHPStan level 8** with `mglaman/phpstan-drupal` and a custom rule set in `phpstan_rules/`. `phpat/phpat` enforces architectural boundaries. Baselines: `phpstan-11.3-and-higher-ignores.neon`.
- Custom **PHP_CodeSniffer** sniffs live in `coding_standards/Canvas/Sniffs/` — Canvas-specific rules around test conventions (`PhpunitAnnotationsSniff`, `KernelTestBaseSniff`, `ForbiddenTestTypesSniff`).
- Final, typed, dependency-injected services. New code should use OOP hook handlers in `src/Hook/` per Drupal 11's hook attribute system.
- Doc-comments describe *current* behavior only. No "we used to…" prose. Open items go in `@todo` tags with a concrete action.

### 7.2 JS / TS / CSS

- **ESLint** (`eslint.config.mjs`), **Prettier** (`.prettierrc.json`), **Stylelint**, **cspell** (`.cspell.json`).
- **Import order** is enforced by `@ianvs/prettier-plugin-sort-imports` — full table in `docs/react-codebase/import-sorting.md`. TL;DR: builtins → `react` → external (no `.`/`@`) → external `@`-scoped → `<THIRD_PARTY_MODULES>` → `@/...` local → relative → types (same order) → CSS.

### 7.3 Commits

[Conventional Commits](https://www.conventionalcommits.org). `feat:` → minor bump, `fix:`/`chore:` → patch.

---

## 8. Quality gates — **run after every PHP change**

From the **module root** (`web/modules/contrib/canvas/`):

```bash
# Static analysis (always run on the modified file or full tree)
composer run phpstan
composer run phpcs
composer run phpcbf      # to auto-fix what can be fixed

# Tests (subset; run full suite in CI only)
composer run phpunit -- --help
```

Or, for a single file (e.g. inside DDEV):

```bash
ddev exec ./vendor/bin/phpstan analyze \
  --configuration=web/modules/contrib/canvas/phpstan.neon \
  --autoload-file=web/modules/contrib/canvas/phpstan_rules/autoload.php \
  web/modules/contrib/canvas/<changed-file>

ddev exec ./vendor/bin/phpcs \
  --standard=web/modules/contrib/canvas/phpcs.xml --no-cache \
  web/modules/contrib/canvas/<changed-file>
```

For JS/TS/CSS:

```bash
cd ui && npm run lint           # eslint + prettier + stylelint + cspell
```

OpenAPI lint (after touching `openapi.yml` or routes):

```bash
npx @redocly/cli@latest lint openapi.yml
```

DDEV shortcuts (`ddev-drupal-xb-dev` add-on):

- `ddev xb-site-install` — reinstall a clean test site.
- `ddev xb-ui-build` — build the React UI.
- `ddev xb-phpcs`, `ddev xb-phpstan`, `ddev xb-phpunit`, `ddev xb-eslint`, `ddev xb-fix`.

---

## 9. Testing

| Type | Location | Runner |
|---|---|---|
| Unit | `tests/src/Unit/` | PHPUnit |
| Kernel | `tests/src/Kernel/` | PHPUnit |
| Functional / FunctionalJavascript | `tests/src/Functional*/` | PHPUnit |
| Playwright (E2E) | `tests/src/Playwright/` | `npm run test:playwright` |
| Fixtures (recipes) | `tests/fixtures/recipes/{base,test_site,test_site_personalization}` | Drupal recipes API |

### Conventions

- Source plugin coverage starts from `ComponentSourceTestBase`.
- Prop-expression coverage is split between a unit test (`Tests/Unit/PropExpressionTest`) and a kernel test (`Tests/Kernel/PropExpressionKernelTest`) reusing the same cases — fast feedback wins.
- Field-type/field-widget ecosystem completeness is enforced by `EcosystemSupport/FieldInstanceSupportTest` and `EcosystemSupport/FieldWidgetSupportTest` — when adding support for a new field type or widget, extend those.
- Custom PHPCS sniffs prevent forbidden test types and missing annotations — see `coding_standards/Canvas/Sniffs/Tests/`.
- Playwright tests parallelize per file (serial within a file). Use the `DrupalSite` fixture and the `drupal` page-object. Tests must be retry-safe (no cross-test state).
- During early MR work, disable E2E in CI by setting `_CANVAS_E2E_TESTS: false` in `.gitlab-ci.yml`.

### Setup

See `docs/testing/setup.md`. TL;DR: `composer run install-dev-deps` (or `ddev exec -d /var/www/html/modules/contrib/canvas composer install-dev-deps`).

---

## 10. Local dev workflow (DDEV recommended)

```bash
mkdir ~/Sites/canvas-dev && cd ~/Sites/canvas-dev
ddev config --project-type=drupal --php-version=8.3 --docroot=web
ddev composer create drupal/recommended-project:11.x@dev --no-install
ddev add-on get drupal-canvas/ddev-drupal-xb-dev
ddev xb-setup
ddev xb-dev-extras
```

Add `$settings['extension_discovery_scan_tests'] = TRUE;` to `sites/default/settings.php`.

### Hidden dev tips (from `CONTRIBUTING.md`)

- **Hold `V`** in the editor to hide the React overlay and inspect the iframe DOM directly. Click into the iframe with `V` held to "lock" the hidden state.
- Reinstall the site (`ddev xb-site-install`) and rebuild the UI (`ddev xb-ui-build`) frequently — Canvas does *a lot* of cache-sensitive auto-discovery.
- When working on a fresh sqlite DB: `rm web/sites/default/.ht.sqlite`.

---

## 11. Issue & contribution workflow

1. Issue queue: <https://www.drupal.org/project/issues/canvas>. Each domain doc lists its issue-queue component (Data model, Shape matching, Redux-integrated field widgets, Config management, …). Match `CODEOWNERS` to find reviewers.
2. Source: <https://git.drupalcode.org/project/canvas>. Contribution model is GitLab merge requests on issue forks.
3. Architectural decisions go in `docs/adr/` via `adr new "..."` (npryce/adr-tools).
4. Releases: tag `vX.Y.Z` on `1.x` (or a `hotfix-YYYYMMDD` branch); automation creates the unprefixed `X.Y.Z` release tag with built assets. See `docs/release-process.md`.

---

## 12. Glossary cheat sheet

- **component** — code that produces markup + optional CSS/JS, parameterized by inputs.
- **component** — the config entity wrapping a component (one per eligible SDC/block/JS).
- **component type / source** — `sdc`, `block`, `js`, `fallback`.
- **component instance** — UUID + version + inputs + (optionally) parent_uuid+slot.
- **component tree / layout** — a hierarchical list of component instances stored in a Canvas field.
- **slot** — a named hole inside a component that can hold child instances.
- **region** — a top-level theme region (e.g. `content`, `header`); behaves like a slot at the page level.
- **prop / explicit input** — a typed argument to a component (SDC/JS); for blocks the equivalent is plugin settings.
- **implicit input** — input not supplied by the editor (route, time, user, contexts) — currently only partial support.
- **prop shape** — the normalized JSON-schema shape of a prop (no titles/descriptions).
- **prop source** — how a prop's value is retrieved: `static`, `entity_field`, `host_entity_url`, `adapted`, (TBD) `remote`.
- **prop expression** — DSL pointing at exactly which field props to read (string ↔ typed PHP object).
- **conjured field** — a synthetic field instance (not in code, not in config) backing a static prop source.
- **structured vs. unstructured data** — entity fields vs. inline static values.
- **page region** — a config entity storing a component tree per theme region.
- **pattern** — a reusable saved sub-tree (absorbed at insertion time; not live-linked).
- **content template** — a config entity binding a layout to entity-type+bundle+view-mode.
- **brand kit** — global config entity for fonts and design tokens.
- **folder** — UI-only grouping of components/patterns/code-components.
- **canvas page** — `canvas_page` content entity, the standalone landing page.
- **auto-save / working copy** — per-user/per-entity overlay store; "Publish all" promotes to live.
- **versioned config entity** — `Component` / `JavaScriptComponent`, with hash-named versions.
- **fallback** — special source plugin that retains data when a real source disappears.

---

*Status: active. When in doubt, search the in-tree `docs/` first, then the issue queue.*
