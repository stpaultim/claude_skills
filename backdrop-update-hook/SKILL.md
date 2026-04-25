---
name: backdrop-update-hook
description: Write a Backdrop CMS update hook (hook_update_N) following Backdrop's CMRR numbering convention and the "mirror in hook_install" pattern. Use whenever the user asks to add, write, or generate an update hook for a Backdrop module — or when ANY change to a Backdrop module touches schema (db_create_table, db_add_field, db_change_field), config defaults, fields/instances, taxonomy seeding, menu links, permissions, or any state that an existing site needs to acquire on next bee updb. Also use proactively whenever you're about to add a field, column, table, or config value to an installed Backdrop module — fresh installs use hook_install, existing installs use hook_update_N, and they MUST stay in sync.
---

# Backdrop update hook

## When this matters

Backdrop has two installation paths and they're easy to desync:

- **Fresh install** runs `hook_install()`. Update hooks are skipped entirely.
- **Existing site** runs `hook_update_N()` via `bee updb`. `hook_install()` is not re-run.

If you only write the update hook, fresh sites silently miss the change. If you only update `hook_install()`, existing sites silently miss the change. SimpleTest runs install only — so update-hook-only logic is invisible to tests too.

The fix is the **`_ensure_X()` helper pattern**: one idempotent function called from both places.

## Step 1 — Pick the update number (CMRR)

Backdrop update numbers are 4 digits in `CMRR` format:
- **1 digit** core compatibility (always `1` for Backdrop 1.x)
- **1 digit** module's major release
- **2 digits** sequential

| Context | First update number |
|---|---|
| Module ported from Drupal 7, first Backdrop update | `1000` |
| 1.x-1.x release series | `1100` |
| 1.x-2.x release series | `1200` |
| 1.x-3.x release series | `1300` |

**`7000`–`9999` are reserved for Drupal 7 compatibility.** Never start a new Backdrop update there. If you see `hook_update_last_removed()` returning `7301`, the *next* update is still `1000` (or `1100`, etc.) — not `7302`.

To pick the correct number:

1. Read the module's `.install` file.
2. Find the highest existing `hook_update_N`. The next one is `N + 1`.
3. If there are no existing updates in the right CMRR band, start fresh at the band's base (`1000` / `1100` / `1200`).
4. Check the module's `.info` for `version = 1.x-2.x-…` — that tells you which band you're in.

## Step 2 — Decide if you need a mirrored install helper

Yes, if the change adds anything that must exist for the module to function:

- New schema (table, column, index)
- New field or field instance
- New taxonomy term, role, permission grant, menu link
- New config default, state value, or variable

No (update hook alone is fine), if:

- One-time data migration (e.g. converting old values to a new format) — fresh sites never had the old data
- Cleanup of removed legacy state
- Cache rebuild or cache_clear that's only meaningful for sites that ran older code

When in doubt: **mirror it.** Idempotent helpers cost nothing to call twice; missing state on a fresh install is a bug that takes a while to surface.

## Step 3 — Write the helper, idempotent

Use a private helper named `_<module>_ensure_<thing>()`. Wrap every operation in a guard so it's safe to call twice.

```php
/**
 * Ensure the {table_or_field} exists. Idempotent.
 */
function _mymodule_ensure_priority_field() {
  if (field_info_field('field_priority')) {
    return;
  }
  $field = array(
    'field_name' => 'field_priority',
    'type' => 'list_integer',
    'settings' => array(
      'allowed_values' => array(
        1 => 'Low',
        2 => 'Medium',
        3 => 'High',
      ),
    ),
  );
  field_create_field($field);
}
```

### Common guards

| Operation | Guard |
|---|---|
| Add a field | `if (field_info_field($name))` |
| Add a field instance | `if (field_info_instance($entity_type, $field, $bundle))` |
| Add a column | `if (db_field_exists($table, $col))` |
| Add a table | `if (db_table_exists($table))` |
| Add a taxonomy term | `taxonomy_get_term_by_name($name, $vocab)` |
| Add a permission to a role | filter against `user_permission_get_modules()` first — `user_role_grant_permissions()` silently drops the entire list if any permission is unknown to Backdrop yet |
| Add a menu link | `menu_link_get_preferred()` or query `menu_links` by path |

### Backdrop-specific gotchas to bake in

- **New TaxonomyTerm needs `'parent' => array(0)`.** Without it, `taxonomy_term_save()` writes `taxonomy_term_data` but skips `taxonomy_term_hierarchy`, and the term overview admin page shows "No terms available" even though the term exists.
- **Adding a new table?** `bee cc all` may not flush schema cache. After creating the table, run `db_query("DELETE FROM {cache_bootstrap} WHERE cid = 'schema'")` or document that the user must.
- **Menu links** — call `menu_rebuild()` before `menu_link_save()` on fresh installs, or the parent path lookup fails.
- **`hook_install()` runs before `hook_enable()`** — don't reference module-defined entity types or other modules' hooks before they've registered.

## Step 4 — Wire the helper into both places

```php
/**
 * Implements hook_install().
 */
function mymodule_install() {
  _mymodule_ensure_priority_field();
  _mymodule_ensure_priority_instance('node', 'task');
}

/**
 * Add a priority field to task nodes.
 */
function mymodule_update_1102() {
  _mymodule_ensure_priority_field();
  _mymodule_ensure_priority_instance('node', 'task');
}
```

The update hook docblock is what shows up in `bee updb`'s prompt — write it as a sentence describing the user-visible change ("Add a priority field to task nodes."), not as restating the function name.

## Step 5 — Verify before handing off

- Re-read the `.install` file end-to-end. Are the new helper, the install hook, and the update hook all consistent?
- Is the update number monotonically increasing within its CMRR band?
- If the change is a new table, did you mention the schema cache flush?
- If the change adds a permission, does any role need the grant? If so, mirror that too (with the `user_permission_get_modules()` filter).

## Quick reference — the canonical shape

```php
/**
 * Implements hook_install().
 */
function mymodule_install() {
  _mymodule_ensure_X();
  _mymodule_ensure_Y();
}

/**
 * Implements hook_update_last_removed().
 */
function mymodule_update_last_removed() {
  return 7301;  // last D7 update, if this module was ported
}

/**
 * One-sentence description of what changed for site builders.
 */
function mymodule_update_1100() {
  _mymodule_ensure_X();
}

/**
 * One-sentence description of the next change.
 */
function mymodule_update_1101() {
  _mymodule_ensure_Y();
}

/* ---------- Helpers (idempotent) ---------- */

function _mymodule_ensure_X() {
  if (/* X already exists */) {
    return;
  }
  // create X
}

function _mymodule_ensure_Y() {
  if (/* Y already exists */) {
    return;
  }
  // create Y
}
```
