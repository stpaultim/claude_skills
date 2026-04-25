---
name: backdrop-layout-edit
description: Edit a Backdrop CMS layout — add, move, reorder, or remove blocks; change block settings or visibility conditions; create a new layout; or ship a layout as default config in a module. Use whenever the user asks to add/move/remove a block, reorganize a layout, change which region something appears in, edit `layout.layout.*.json`, fix a layout that's missing a title/messages/tabs block, or wire up a default layout for a module's `config/` directory. Also use proactively when the user reports "the messages aren't showing", "the title disappeared", "tabs are missing on this page" — these are usually layout block placement issues.
---

# Backdrop layout editing

There are three ways to edit a Backdrop layout. Pick the one that matches the goal:

| Path | When to use |
|---|---|
| **Admin UI** (`admin/structure/layouts`) | One-off changes on a live/dev site. Backdrop generates UUIDs, validates regions, and prevents mistakes. Suggest this first when a user just wants to nudge one thing. |
| **Layout API in PHP** (install/update hooks) | Shipping a layout from a module — adding/removing/configuring blocks at install or upgrade time. **This is the canonical scripted approach.** |
| **Edit JSON directly** | Hand-fixing broken state, generating defaults outside PHP, or scripted bulk migrations. Carries the risk of triple-UUID mistakes — see the gotcha below. |

## The Layout API (PHP, in hooks)

The cleanest pattern. Used in real-world recipe modules like `digital_agency`. Works inside `hook_install()`, `hook_update_N()`, or any custom code. The API takes care of UUID generation, positions/content sync, and storage.

### Core methods

```php
$layout = layout_load('home');                              // Load by name
$block = $layout->addBlock($module, $delta, $region);       // Returns the new block; ->uuid available immediately
$layout->removeBlock($uuid);                                // Removes from positions AND content
$layout->save();                                            // Persist all changes
```

### Add a block

```php
$layout = layout_load('default');
$news_block = $layout->addBlock('views', 'news_block-block', 'bottom');
$layout->addBlock('system', 'page_components:messages', 'content');
$layout->addBlock('system', 'powered-by', 'footer');
$layout->save();
```

`addBlock()` returns the new block object — keep it in a variable when you need to set its settings afterward (see "Configure a block" below). Note that delta can contain a colon: `page_components:messages` for PageComponent blocks.

### Remove a block

If you know the UUID, just call `removeBlock($uuid)`. If you don't (because the layout was provided by another module/recipe and the UUID was generated dynamically), iterate by delta:

```php
foreach ($layout->content as $block_id => $block_info) {
  if ($block_info->delta == 'breadcrumb') {
    $layout->removeBlock($block_info->uuid);
  }
}
```

This is the digital_agency pattern — robust against UUIDs you didn't generate.

### Move or reorder blocks

The Layout API doesn't expose a clean "move" or "reorder" method. Two options:

1. **Remove + re-add** — simplest, but the new block gets a new UUID, so any code referencing the old UUID needs updating.
2. **Modify `$layout->positions` directly** — preserves UUIDs:

```php
$layout = layout_load('default');
// Remove from current region
foreach ($layout->positions as $region => &$uuids) {
  $uuids = array_values(array_filter($uuids, function($u) use ($target_uuid) {
    return $u !== $target_uuid;
  }));
}
unset($uuids);
// Add to new region (or insert at specific index)
$layout->positions['footer'][] = $target_uuid;
$layout->save();
```

### Configure a block (settings, conditions, classes)

The API's `addBlock()` doesn't expose every setting. For granular control, use `config_set()` against the layout config keyed by UUID:

```php
$smart_block = $layout->addBlock('smart_blocks', 'SmartBlocksHeroBlock', 'top');
$layout->save();

config_set('layout.layout.default', 'content.' . $smart_block->uuid . '.data.settings.title_display', 'page_title');
config_set('layout.layout.default', 'content.' . $smart_block->uuid . '.data.settings.image', $fid);
```

Common settings paths:
- `data.settings.title` — custom block title
- `data.settings.title_display` — `default`, `none`, `title` (custom), `block` (block name), `page_title`
- `data.style.data.settings.classes` — extra CSS classes on the block wrapper
- `data.settings.label` / `data.settings.machine_name` — for blocks that have admin labels
- `data.conditions.0.plugin` + `data.conditions.0.data.settings.*` — visibility conditions (path, role, permission)

### A complete recipe-style example

```php
function mymodule_install() {
  $layout = layout_load('home');

  // Remove unwanted defaults by delta.
  foreach ($layout->content as $block_info) {
    if ($block_info->delta == 'powered-by') {
      $layout->removeBlock($block_info->uuid);
    }
  }

  // Add new blocks, capture UUIDs we'll need to configure.
  $hero = $layout->addBlock('smart_blocks', 'SmartBlocksHeroBlock', 'top');
  $layout->addBlock('system', 'page_components:messages', 'content');
  $layout->addBlock('system', 'powered-by', 'footer');

  $layout->save();

  // Granular settings for the hero block.
  config_set('layout.layout.home', 'content.' . $hero->uuid . '.data.settings.title', 'Welcome');
  config_set('layout.layout.home', 'content.' . $hero->uuid . '.data.settings.title_display', 'page_title');
}
```

Mirror in `hook_install` AND `hook_update_NNNN` per the project rule (see the `backdrop-update-hook` skill).

## The JSON path (when API isn't available)

Use this only when you can't run PHP — for example, when generating a layout file outside an install hook, or fixing broken state by hand. Layouts live as JSON config:

- **Active config** (installed site): `files/config_<hash>/active/layout.layout.<name>.json`
- **Module defaults**: `<module_path>/config/layout.layout.<name>.json`

### Anatomy of a layout JSON

```json
{
  "_config_name": "layout.layout.<name>",
  "name": "<name>",
  "title": "Human Readable Title",
  "path": "node/%",
  "layout_template": "moscone_flipped",
  "module": "layout",
  "weight": 0,
  "storage": 2,
  "positions": {
    "header":  ["uuid-1", "uuid-2"],
    "content": ["default"],
    "footer":  ["uuid-3"]
  },
  "conditions": [],
  "contexts": [],
  "relationships": [],
  "content": {
    "uuid-1": { "plugin": "...", "data": { ... "uuid": "uuid-1" ... } },
    "uuid-2": { ... },
    "uuid-3": { ... },
    "default": { ... }
  }
}
```

`positions` maps each region (defined by the `layout_template`) to an ordered array of block UUIDs. Array order = display order. `content` is keyed by UUID and holds each block's plugin and config.

`storage`: `1` = Default (provided by code), `2` = Database (user-modified), `4` = Module-provided default.

### The triple-UUID rule (most common gotcha)

For each block, the same UUID must appear in **three** places:

1. As a **value** inside one of the `positions[region]` arrays.
2. As a **key** in the `content` object.
3. As `data.uuid` **inside** the block's content entry.

If any of these are missing or mismatched, the block's Configure/Remove links 404 from the admin UI. **The Layout API handles this for you — direct JSON editing does not.**

The literal string `"default"` is the standard UUID for the main page-content block — keep it as-is.

### Generate a UUID

```bash
uuidgen | tr 'A-Z' 'a-z'
# OR inside the site:
ddev exec "cd /var/www/html && bee scr -e 'echo backdrop_generate_uuid();'"
```

A real UUIDv4 is what Backdrop's UI produces. The main-content `"default"` is the documented exception.

### Minimal block entry

```json
"<uuid>": {
  "plugin": "<module>:<delta>",
  "data": {
    "status": 1,
    "module": "<module>",
    "delta": "<delta>",
    "settings": {
      "title_display": "none",
      "title": "",
      "style": "default",
      "block_settings": [],
      "contexts": []
    },
    "uuid": "<uuid>",
    "style": {
      "plugin": "default",
      "data": { "settings": { "classes": "" } }
    }
  }
}
```

### Validation checklist before saving JSON edits

- [ ] Every UUID in `positions` has a matching `content[<uuid>]` entry.
- [ ] Every `content[<uuid>]` entry's `data.uuid` matches its key.
- [ ] Every `content` UUID appears in exactly one region's `positions` array.
- [ ] `_config_name` matches the file name (`layout.layout.<name>` ↔ `layout.layout.<name>.json`).
- [ ] JSON parses (`jq . file.json > /dev/null`).

## Plugin id reference (common blocks)

The `addBlock($module, $delta, $region)` signature maps to the JSON `plugin: "<module>:<delta>"`.

| `addBlock(...)` | JSON plugin id | What it is |
|---|---|---|
| `('system', 'main', ...)` | `system:main` | Main page content (UUID is literally `"default"`) |
| `('system', 'header', ...)` | `system:header` | Site name/logo/user-menu cluster |
| `('system', 'main-menu', ...)` | `system:main-menu` | Primary navigation |
| `('system', 'powered-by', ...)` | `system:powered-by` | Footer credit |
| `('system', 'breadcrumb', ...)` | `system:breadcrumb` | Breadcrumbs |
| `('system', 'page_components:title', ...)` | `system:page_components:title` | Page H1 title |
| `('system', 'page_components:title_combo', ...)` | `system:page_components:title_combo` | Title + tabs combined |
| `('system', 'page_components:tabs', ...)` | `system:page_components:tabs` | Local tasks tabs |
| `('system', 'page_components:messages', ...)` | `system:page_components:messages` | `backdrop_set_message()` output |
| `('system', 'page_components:action_links', ...)` | `system:page_components:action_links` | "Add new X" action links |
| `('layout', 'custom_block', ...)` | `layout:custom_block` | Inline custom HTML block |
| `('layout', 'hero', ...)` | `layout:hero` | Layout-defined hero block |
| `('node', 'content', ...)` | `node:content` | A specific node rendered as a block |
| `('views', '<view>-<display>', ...)` | `views:<view>-<display>` | A Views block |
| `('<module>', '<delta>', ...)` | `<module>:<delta>` | Any block defined by `hook_block_info()` |

To find a custom module's plugin id: `hook_block_info()` keys are deltas; module name is the first arg.

## Visibility conditions

Conditions live at `content[<uuid>].data.conditions` (per-block) or at the top-level `conditions` array (whole-layout).

Common condition plugins: `user_role`, `user_permission`, `path`, `node_type`, `language`.

```json
"conditions": [
  {
    "plugin": "user_role",
    "data": {
      "settings": {
        "user_roles": { "team_member": "team_member" },
        "negate": 0
      }
    }
  }
]
```

Or via PHP after `$layout->save()`:

```php
config_set('layout.layout.default', 'content.' . $block->uuid . '.data.conditions.0.plugin', 'path');
config_set('layout.layout.default', 'content.' . $block->uuid . '.data.conditions.0.data.settings.paths', 'about');
config_set('layout.layout.default', 'content.' . $block->uuid . '.data.conditions.0.data.settings.visibility_setting', '1');
```

## Critical gotchas (Backdrop-specific)

- **Messages block missing = silent failure mode.** If a layout doesn't include `system:page_components:messages`, every `backdrop_set_message()` call vanishes silently. Always verify it's present in any new layout. (Tim's drop_base install seeds this on purpose for that reason.)
- **Title flows through the title block, not `$variables['title']`.** Backdrop's H1 comes from `backdrop_get_title()` rendered by `system:page_components:title` (or `:title_combo`). It is NOT pulled from `$variables['title']` in `page.tpl.php` like Drupal 7. To output HTML in a page title, call `backdrop_set_title($html, PASS_THROUGH)` and ensure the title block is present in the layout.
- **Drop Suite layouts depend on `.l-header { position: relative }`.** Don't strip that from CSS — Seven's `::before` overlay relies on it.
- **`storage: 4` (module-provided) layouts can't be edited via UI** until they're "overridden" (becomes `storage: 2`). Don't manually flip storage in JSON; let the user override via UI, or ship `storage: 2` to begin with.
- **After editing JSON in active config**: run `bee cc all` so the cached layout config is rebuilt. The Layout API path doesn't need this — `$layout->save()` invalidates correctly.
- **After editing a module's `config/` defaults**: existing sites won't pick up the change. Either re-run `bee config-import` for that file, or write an update hook that calls `config_install_default_config('<module>')` (which only imports configs not already present in active).

## Shipping a layout as a module default

Two flavors. Pick based on whether the layout needs *runtime data* (file IDs, taxonomy term IDs, etc.) or is fully static.

### Static layout — ship as JSON in `config/`

1. Place the file at `<module>/config/layout.layout.<name>.json`.
2. Set `"storage": 2` in the JSON (it becomes editable once imported).
3. Register it in `hook_config_info()`:
   ```php
   function mymodule_config_info() {
     return array(
       'layout.layout.<name>' => array(
         'label' => t('My Layout'),
         'group' => t('Layouts'),
       ),
     );
   }
   ```
4. Fresh installs auto-import everything in `config/`. Existing sites need an update hook calling `config_install_default_config('mymodule')`. Mirror that in `hook_install()` per the project rule.

### Dynamic layout — build it in `hook_install()` with the API

When the layout depends on file IDs, taxonomy terms, or other entities created during install, use the Layout API approach. Load the existing layout (e.g. the core `default` or `home`), mutate it, save. The digital_agency module's install file is a clean reference for this pattern.

```php
function mymodule_install() {
  // Create files/terms/nodes first…
  $hero_fid = _mymodule_install_hero_image();

  // Then mutate the layout.
  $layout = layout_load('home');
  foreach ($layout->content as $block_info) {
    if ($block_info->delta == 'powered-by') {
      $layout->removeBlock($block_info->uuid);
    }
  }
  $hero = $layout->addBlock('smart_blocks', 'SmartBlocksHeroBlock', 'top');
  $layout->save();
  config_set('layout.layout.home', 'content.' . $hero->uuid . '.data.settings.image', $hero_fid);
}
```
