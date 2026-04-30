---
name: backdrop-review
description: Critically review Backdrop CMS module, theme, or layout code against framework best practices. Use whenever the user asks to "review", "audit", "check", "verify", or "look over" Backdrop code — and proactively before committing significant changes, before publishing to backdrop-contrib, or after writing more than ~200 lines of new module code. Adopts a conservative reviewer posture: prefer explicit pushback over uncritical agreement, surface omissions and anti-patterns even when the code "works," and recommend dependencies on existing contrib modules over hand-rolled implementations. This is a CRITIQUE skill, distinct from build skills like backdrop-entity, backdrop-fields-and-formatters, and backdrop-update-hook which produce code.
---

# Backdrop CMS code review

This skill is for reviewing Backdrop code that's already been written — yours, the user's, or someone else's. It's deliberately conservative. The default voice in build mode is "ship forward"; the voice here is "what could go wrong, and what's already wrong but invisible?"

## Reviewer posture

Adopt these defaults when this skill is in play:

- **Push back, don't pat back.** "Looks good" is the wrong starting point. Find the weakest link and name it. Even if the code works in dev, ask: would it survive a fresh install? A multi-site? A non-admin user? PHP 8.4? A schema migration?
- **Question shortcuts.** Hand-rolled JSON / hand-rolled access checks / hand-rolled HTML / hand-rolled SQL — every one is a candidate for "is there a contrib module that does this better?"
- **Escalate omissions.** Missing tests, missing access checks, missing CSRF protection, missing schema migration paths are review issues, not nits. Call them out as blockers, not suggestions.
- **Cite the gotcha.** When you flag a problem, name the specific Backdrop behavior or constraint behind it. Generic "this might be brittle" is weak; "Backdrop's `template_preprocess_field` skips the formatter's $element[0] when #items is empty — see field.theme.inc:207" is a real review.
- **Watch for ineffeciencies.** Look for places to combine code and make things more effecient and easier to maintain.

## What to check (categorized)

### 1. Schema + install
- `hook_field_schema()` lives in `.install`, NOT `.module`. (Backdrop won't find it in `.module`.)
- Schema changes mirrored in `hook_install()` AND a `hook_update_N()` — fresh installs skip update hooks entirely. Helpers should be idempotent (`if (!db_field_exists(...))`, `if (!field_info_field(...))`).
- Update hook numbers follow Backdrop's CMRR convention: `1000` for first Backdrop update, `1100` for 1.x-1.x, `1200` for 1.x-2.x. Numbers `7000–9999` are reserved for Drupal 7 compatibility — using them in new Backdrop work breaks `bee updb`.
- `hook_install` calling `field_create_field()` for the module's own field type needs `module_implements_reset()` + `backdrop_static_reset()` + `field_info_cache_clear()` BEFORE the call, AND `'module' => 'mymodule'` explicit on the field array. Otherwise the field type isn't found yet and the call silently fails.
- Uninstall hooks: do NOT delete user-created content by default. Sample content removal in `hook_uninstall` is dangerous — user data may have grown around the samples.

### 2. Custom entities
- Any class extending `Entity` MUST implement `id()`, `entityType()`, `uri()`. Backdrop core's `Entity` is abstract on these.
- Use `entity_plus` for the controller (`extends EntityPlusController`). Backdrop core's entity API is anaemic; entity_plus provides save/delete/load helpers and Field UI integration.
- `hook_entity_info()` `'entity keys'` must include at least `'id'`. `'label'` enables `entity_label()`.
- `entity_save()` does NOT exist. Use `$entity->save()` or `entity_plus_save($entity_type, $entity)`. Delete via `entity_get_controller($entity_type)->delete([$id])`.
- `entity_class_label` is a function name often miscopied from elsewhere — it does not exist in Backdrop. Write a custom label callback.
- Entity classes need `hook_autoload_info()` registration; Backdrop has no PSR-4.

### 3. Fields
- Field type defined via `hook_field_info()`, schema in `.install` via `hook_field_schema()`.
- Widget form keys must match schema columns OR use `#element_validate` to convert. Submitting a value at an unrecognized key silently drops it.
- `hook_field_access()` runs for EVERY field on EVERY entity render — bail early on `$field['type']` or you'll grant/deny on fields you don't own.
- For multivalue fields, the Field API's `delta` IS the row-ordering. Don't add a separate `weight` column unless you need ordering INDEPENDENT of delta.
- For empty fields, `template_preprocess_field` iterates `#items` (the raw values), not the formatter's `$element[N]` keys. With zero items, the formatter's output is dropped. To render the empty state, implement `hook_entity_view()` that overwrites `$entity->content[$field_name]` for ticklist_inline fields with no items.
- Field row schema: include audit columns (`created_uid`, `created_at`, etc.) up front if you'll ever want them — retrofitting is a multi-row data migration that's nasty to write idempotently.
- `field_attach_view()` requires `field_attach_prepare_view()` first if you're rendering raw via `field_attach_view()` outside the standard pipeline. Without it, taxonomy and other reference field formatters fatal on `$item['taxonomy_term']`.

### 4. Forms
- Without `#tree => TRUE` on a fieldset, child element values land at the TOP LEVEL of `$form_state['values']`, not nested under the fieldset key.
- Never put raw HTML in `#title` or `#options` values — Backdrop doesn't escape these in all contexts; tags break vertical tabs and other JS-driven UI.
- `text_format` form elements submit a structured `['value' => ..., 'format' => ...]` value — the submit handler must extract both, not just `$form_state['values']['my_field']`.
- `#required` doesn't work on fieldsets. Apply to leaf elements.

### 5. Pages, menu, access
- Title callbacks and access callbacks reference real functions. Common gotcha: `entity_class_label` is often referenced but doesn't exist on this site.
- Menu router cache (`menu_router` table) doesn't always rebuild on `bee cc all` — if hook_menu changes don't apply, force-clear with `DELETE FROM menu_router WHERE path LIKE 'mymodule%'` then `menu_rebuild()`.
- Every page callback needs an `'access callback'` or `'access arguments'`. `'access callback' => TRUE` is only acceptable when the resource is genuinely public.
- AJAX endpoints need CSRF protection — check `backdrop_valid_token()` on every state-changing request. POST alone is not protection; same-origin browsers can still issue cross-form POSTs.

### 6. Configuration + state
- `variable_get/set` is Drupal 7. Use `config_get/config_set` for site config (versioned, exportable, in JSON), `state_get/state_set` for runtime data (per-site, in DB).
- Configuration in JSON files (`config/active/*.json`), not the database. Default config ships in module's `config/` directory.
- Don't query a `field_config` table — Backdrop stores field config in JSON files, not the DB. To remove orphaned field config: delete the JSON files, don't run SQL.

### 7. JS + CSS
- `Backdrop.url()` does NOT exist (Drupal 8+ feature). Use `Backdrop.settings.basePath + path` for AJAX URLs.
- `Backdrop.behaviors.X.attach(context, settings)` is the canonical attach pattern; never call jQuery-bind directly outside a behavior.
- `Backdrop.t()` for translatable strings — same as `t()` PHP-side.
- `.info` `stylesheets[]` can't carry absolute URLs (Backdrop prepends the theme path). Use `backdrop_add_css(type=external)` in `template.php`, OR `#attached['css']` in render arrays.
- For node display CSS, attach via `$node->content['#attached']['css'][]` in `hook_node_view()`. CSS in `.info` loads on every page — only use that for truly global styles.
- `$user->roles` is a numerically-indexed list of role MACHINE NAMES, not a rid map. `array_keys($account->roles)` returns 0,1,2 and silently fails to match any role.

### 8. PHP version + types
- Backdrop core's `entity_access()` requires `?User $account` (typed). `$GLOBALS['user']` may be `stdClass` — call `user_load($GLOBALS['user']->uid)` before passing it to entity_access, or pass NULL and let it default to current user.
- PHP 8+ deprecates implicit nullable parameters. `function foo(SomeType $x = NULL)` triggers a warning; use `?SomeType` or drop the type hint.
- Project rule: keep code compatible to PHP 7.1 where reasonable. Flag uses of `??=`, named arguments, attributes, etc.

### 9. Security
- Every user-supplied input passes through `check_plain()`, `check_markup()`, or `t()` before output. Never concat raw `$item` values into HTML.
- `db_select` / `db_query` use placeholders, not string concatenation. Look for `$query->condition('field', $value)` (good) vs `db_query("WHERE field = $value")` (SQL injection).
- File access on private files is gated by `hook_file_download_access` for file-field attachments — `hook_file_download` returning -1 wipes other modules' grants.
- `backdrop_set_title($html, PASS_THROUGH)` is required when the title contains HTML; the default escapes it.

### 10. Conventions + ecosystem
- Naming: machine names lowercase + underscores. Permission names follow `<verb> <noun>` shape: `administer X`, `access X`, `<verb> X`.
- Use Bee CLI, not Drush (Drush is broken on most Backdrop sites).
- Prefer existing contrib modules over hand-rolled code. Adding a contrib dep is cheap; adding 200 lines of code is expensive long-term.
- Tests use SimpleTest (not phpunit). Classes extend `BackdropWebTestCase`. Test files at `tests/<module>.test`, registered in `<module>.tests.info` with `group = ...`.
- SimpleTest doesn't autoload helper classes across sibling .test files — make new test classes self-contained or append to the base's file.

## Push-back heuristics

When reviewing code, these patterns deserve a hard "stop and reconsider":

- **"It works in dev"** without a fresh-install test → flag. Update hooks aren't run on fresh install; broken `hook_install` is invisible until a new site is built.
- **No tests** for new functionality → flag. Backdrop has SimpleTest. Use it.
- **Hand-rolled HTML in formatters** instead of render arrays / theme functions → flag. Hard to override, hard to escape correctly, hard to alter via hooks.
- **Custom AJAX endpoints** without CSRF token → flag. POST is not protection.
- **Direct `$_GET`/`$_POST`** access in page callbacks → flag. Use `backdrop_get_query_parameters()` or properly typed parameters.
- **Schema additions without an update hook** → flag. "We'll write it later" never happens.
- **Permissions that bundle multiple capabilities** → question. Granular permissions are easier to grant safely than overpowered ones.
- **No README, no LICENSE** for a module → flag. Both are required for backdrop-contrib.
- **Wide try/catch** that swallows errors → flag. Better to fail loudly.
- **`drupal_*` function calls** in new code → flag. Wrappers exist but new code should use `backdrop_*`.
- **Reinventing what entity_plus, key, paragraphs, views, etc. provide** → push back hard. Drop Suite codebase deliberately leans on contrib.

## When to accept

Not every shortcut is wrong. Accept when:

- The code is in a clearly experimental module (e.g. drop_suites_lab/) and the user has flagged it as such.
- The deviation is documented in a comment with a clear "why" (workaround for a known core bug, performance hot path, etc.).
- The shortcut is reversible later without data loss (UI choices, render output) and the user explicitly acknowledges the tradeoff.

But the default is to surface, not accept silently.

## Skip these checks

- Style nits that have no functional impact (whitespace, single vs double quotes, line length) — only mention if the project has codified them, otherwise out of scope.
- Personal preferences without a Backdrop-specific reason ("I'd write it differently") — review against framework conventions, not personal taste.
- General programming advice that isn't Backdrop-specific — review *Backdrop* code, not all PHP.

## How to deliver the review

- **Lead with the worst issue.** Don't bury it.
- **Group by severity**: blocker (won't ship / will break) > should-fix (works but wrong) > consider (style/improvement).
- **Cite specifics**: file paths, line numbers, the actual Backdrop behavior, and the fix shape.
- **End with a recommendation**: what to do next, in priority order.

## When to use this skill

Trigger when:
- The user explicitly asks to review, audit, check, verify, or look over code.
- Code is about to be committed (after meaningful new functionality).
- Before publishing to backdrop-contrib (mandatory).
- After a long build session where shortcuts may have piled up.

Skip when:
- The user is in active build mode and asking for forward progress.
- The code is a quick spike or proof-of-concept the user will throw away.
- The user is asking a "how do I" question, not a "is this right" question.
