# Canvas — Developer Quick Reference

@docs/CANVAS.md
@docs/CANVAS_AI.md

> All paths relative to `web/modules/contrib/canvas/`

---

## Workflow — follow this order, no skipping

1. **Read the task fully.** Check linked tickets and any referenced docs.
2. **Ask clarifying questions first.** Do not assume intent, scope, or approach.
3. **Inspect before implementing.** Read the relevant reference files below. Your training data is outdated — the real API is in `web/core/`.
4. **Search core first.** Before writing any service, utility, or hook — check `web/core/core.services.yml` and `web/core/lib/Drupal/Core/`. If core already covers it, surface that and ask before writing new code.
5. **Write code last.**

---

## IMPORTANT: This is Drupal 11 — not Drupal 8 or 9

**YOU MUST use current patterns. The following are non-negotiable:**

- **OOP hook handlers** in `src/Hook/` using PHP 8 `#[Hook]` attributes — not procedural `hook_*()` functions in `.module` files. Check `web/core/modules/node/src/Hook/NodeModuleHooks.php` for the canonical pattern.
- **PHP 8 attributes for plugin definitions** — not docblock annotations. Check `web/core/lib/Drupal/Core/Block/Attribute/Block.php` before defining any plugin.
- **Autowiring and autoconfiguration** where the installed core supports it — check `web/core/modules/node/node.services.yml` before writing service definitions.
- **Typed, final, dependency-injected services** — no static calls to `\Drupal::*` in new code.

If you are unsure whether a pattern is current, read the reference files below before proceeding. Do not guess.

---

## Reference files — read before touching these areas

| File | Read when… |
|---|---|
| `web/core/lib/Drupal/Core/Hook/Attribute/Hook.php` | Adding any hook |
| `web/core/modules/node/src/Hook/NodeModuleHooks.php` | Implementing a hook handler class |
| `web/core/lib/Drupal/Core/Block/Attribute/Block.php` | Defining any plugin |
| `web/core/modules/node/node.services.yml` | Registering a service |
| `web/core/core.services.yml` | Looking for an existing injectable service |
| `web/core/lib/Drupal/Core/Extension/module.api.php` | Checking hook availability or deprecations |
| `web/core/core.api.php` | Any area you haven't worked in before |

---

## Quality gates — run after every PHP change, before calling anything done

```bash
# Static analysis
ddev exec ./vendor/bin/phpstan analyze \
  --configuration=web/modules/contrib/canvas/phpstan.neon \
  --autoload-file=web/modules/contrib/canvas/phpstan_rules/autoload.php \
  web/modules/contrib/canvas/<changed-file>

# Code style
ddev exec ./vendor/bin/phpcs \
  --standard=web/modules/contrib/canvas/phpcs.xml \
  --no-cache \
  web/modules/contrib/canvas/<changed-file>

# Single test file
ddev exec vendor/bin/phpunit -c ./web/core/phpunit.xml.dist <path-to-test>

# Single test method
ddev exec vendor/bin/phpunit -c ./web/core/phpunit.xml.dist <path-to-test> --filter <methodName>
```

For JS/TS/CSS changes:
```bash
cd ui && npm run lint
```

DDEV shortcuts: `ddev xb-phpcs`, `ddev xb-phpstan`, `ddev xb-phpunit`, `ddev xb-eslint`, `ddev xb-fix`

PHPStan is level 8. Fix real violations — do not add to the baseline without discussion.

---

## Comment and documentation rules

Doc comments describe **what the code does right now** — nothing else. No conversation history, no "we decided to…" narratives, no prior iteration summaries. A reader opening the file cold should learn current behavior, not the journey that produced it.

Open items → `@todo` tags with a concrete action, one per item. Not prose buried in docblocks.