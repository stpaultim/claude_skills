---
name: backdrop-entity
description: Create a custom Backdrop CMS entity that is fieldable (Manage Fields + Manage Display tabs work), exposed to Views, and has admin UI for create/edit/list. Use whenever the user asks to "create an entity", "add a new content type that isn't a node", "make this thing fieldable", "expose X to Views", "add Manage Display support", or "ship a default View for this entity". Also use proactively whenever the user is designing a data model that is not a great fit for nodes/users/taxonomy — custom entities are the right answer when the data has its own URL space, custom access logic, distinct admin pages, or relationships that nodes can't model cleanly. Pair with backdrop-update-hook for schema changes.
---

# Backdrop custom entity

A complete Backdrop entity has five working pieces:

1. **Schema** — a base table for entity rows
2. **`hook_entity_info()`** — declares the entity to Backdrop
3. **Entity class + controller** — load/save behavior (use entity_plus)
4. **Field UI integration** — Manage Fields + Manage Display tabs
5. **Views integration** — entity property info + a default listing view

If any of the five is missing, the entity feels half-built. The skill below walks the right order: schema → entity_info → class → fields → views.

## Decision: when to build a custom entity at all

Custom entities are the right answer when:
- The data has its own URL space (`drop/<thing>/<id>`) and admin pages
- Access rules differ from nodes (e.g. team-internal, ticket-scoped)
- Listings need to be Views with relationships nodes can't model
- The thing is conceptually NOT content (a CRM contact, a time entry, a ticket)

Use nodes when:
- The thing is editorial content with title + body and the standard publish workflow
- You want the standard node display ecosystem (comments, revisions, moderation)

Use taxonomy when:
- You need bounded, hierarchical categorization
- The data is a label, not a record

## Always use entity_plus

Backdrop core's entity API is anaemic. The contrib `entity_plus` module provides the controller, view callbacks, save/delete/access wrappers, and Views auto-integration that custom entities actually need. Add `entity_plus` (and `entity_ui` for admin) as `dependencies[]` in your `.info` file. This is the de-facto standard for every custom entity in the Drop Suite ecosystem.

## Step 1 — Schema

Define a base table in `hook_schema()`. The primary key is the entity ID. If you have bundles (multiple types within the entity), include a bundle column.

```php
$schema['mymodule_thing'] = array(
  'description' => 'Base table for thing entities.',
  'fields' => array(
    'tid'      => array('type' => 'serial', 'unsigned' => TRUE, 'not null' => TRUE),
    'type'     => array('type' => 'varchar', 'length' => 64, 'not null' => TRUE, 'default' => ''),
    'label'    => array('type' => 'varchar', 'length' => 255, 'not null' => TRUE, 'default' => ''),
    'created'  => array('type' => 'int', 'not null' => TRUE, 'default' => 0),
    'changed'  => array('type' => 'int', 'not null' => TRUE, 'default' => 0),
    'uid'      => array('type' => 'int', 'unsigned' => TRUE, 'not null' => TRUE, 'default' => 0),
  ),
  'primary key' => array('tid'),
  'indexes' => array(
    'type' => array('type'),
    'uid'  => array('uid'),
  ),
);
```

Ship the table in `hook_install()` *and* an `hook_update_NNNN()` per the project rule (see `backdrop-update-hook` skill).

## Step 2 — Declare via `hook_entity_info()`

This is the most consequential hook. Get it right and the rest falls into place.

```php
function mymodule_entity_info() {
  $info = array();

  $info['mymodule_thing'] = array(
    'label'            => t('Thing'),
    'plural label'     => t('Things'),
    'description'      => t('A thing tracked by My Module.'),
    'entity class'     => 'MymoduleThing',
    'controller class' => 'MymoduleThingController',
    'base table'       => 'mymodule_thing',
    'fieldable'        => TRUE,
    'entity keys'      => array(
      'id'     => 'tid',
      'bundle' => 'type',     // omit if single-bundle
      'label'  => 'label',
    ),
    'bundle keys'      => array('bundle' => 'type'),  // omit if single-bundle
    'bundles'          => array(
      'standard' => array(
        'label' => t('Standard Thing'),
        'admin' => array(
          'path'             => 'drop/things/manage/standard',
          'access arguments' => array('administer mymodule'),
        ),
      ),
    ),
    'view modes'       => array(
      'full'   => array('label' => t('Full'),   'custom settings' => FALSE),
      'teaser' => array('label' => t('Teaser'), 'custom settings' => TRUE),
    ),
    'uri callback'     => 'entity_class_uri',
    'access callback'  => 'mymodule_thing_access',
    'module'           => 'mymodule',
    'admin ui'         => array(
      'path'             => 'drop/things',
      'file'             => 'mymodule.admin.inc',
      'controller class' => 'MymoduleThingUIController',
      'menu wildcard'    => '%mymodule_thing',
    ),
  );

  return $info;
}
```

**Critical keys explained**:
- `'fieldable' => TRUE` — without this, Field UI tabs never appear.
- `'bundles'` with `'admin'` paths — without this, the **Manage Fields** and **Manage Display** tabs don't render. Each bundle's admin path is where Field UI hangs the tabs.
- `'view modes'` — what shows up as columns on the Manage Display admin form. `'full'` is the minimum.
- `'admin ui'` — provides list/add/edit/delete admin pages via `entity_ui`.
- `'menu wildcard'` — typically `%<entity_type>`, used in router paths to auto-load the entity.

### Bundles: when explicit, when implicit

Use **explicit bundles** (multiple entries in `'bundles'` array) when different types of the entity have different attached fields — like CRM contacts (Person/Organization/Household, each with its own field set).

Use a **single implicit bundle** (omit `bundle` from entity keys, omit `bundle keys`, list one bundle in `'bundles'`) when the entity has one shape — like a Time Entry. If you need a "type" column for filtering but all rows share the same fields, that column is a regular schema field, not a bundle.

The CRM contact (multi-bundle) and CRM activity (single-bundle) entities in `dropcrm.module` are the canonical references for both shapes.

## Step 3 — Entity class and controller

Both should extend the entity_plus base classes. Place them in `mymodule.entity.inc` and register via `hook_autoload_info()`.

```php
class MymoduleThing extends Entity {
  public function defaultLabel() {
    return $this->label;
  }
  public function defaultUri() {
    return array('path' => 'drop/things/' . $this->identifier());
  }
}

class MymoduleThingController extends EntityPlusController {
  public function create(array $values = array()) {
    $values += array(
      'type'    => 'standard',
      'label'   => '',
      'created' => REQUEST_TIME,
      'changed' => REQUEST_TIME,
      'uid'     => $GLOBALS['user']->uid,
    );
    return parent::create($values);
  }
}
```

Save and delete via the entity_plus convention — **`$entity->save()` not `entity_save()`**, and delete via `entity_get_controller('mymodule_thing')->delete(array($id))`. Mixing the two APIs causes silent data loss.

For the admin UI controller (`MymoduleThingUIController`) extend `EntityDefaultUIController` and override the methods you need (commonly `hook_menu()` adjustments, the overview list, or the edit form).

`hook_autoload_info()`:
```php
function mymodule_autoload_info() {
  return array(
    'MymoduleThing'             => 'mymodule.entity.inc',
    'MymoduleThingController'   => 'mymodule.entity.inc',
    'MymoduleThingUIController' => 'mymodule.admin.inc',
  );
}
```

## Step 4 — Field UI integration

Two things to confirm:

1. The bundle's admin path in `hook_entity_info()` is reachable. Visit it after install — Backdrop should show **Manage Fields** and **Manage Display** tabs automatically. If the tabs are missing, recheck `'fieldable' => TRUE` and bundle `'admin'` paths.

2. **Pseudo-fields for Manage Display** via `hook_field_extra_fields()`. If your entity view callback renders sections that are not Field API fields (computed values, summary blocks, related-entity lists), register them so site builders can reorder/hide them on Manage Display:

```php
function mymodule_field_extra_fields() {
  $extra = array();
  $extra['mymodule_thing']['standard']['display']['summary'] = array(
    'label'       => t('Summary'),
    'description' => t('Computed summary line.'),
    'weight'      => 0,
  );
  return $extra;
}
```

Then in the entity view callback, read display settings and apply them:

```php
$display = field_extra_fields_get_display('mymodule_thing', $entity->type, $view_mode);
if (!empty($display['summary']['visible'])) {
  $build['summary'] = array(
    '#markup' => check_plain($entity->summary()),
    '#weight' => $display['summary']['weight'],
  );
}
```

Hidden pseudo-fields should set `#access => FALSE`, not be omitted — that lets `hook_entity_view_alter()` re-enable them if needed.

## Step 5 — Views integration

entity_plus auto-generates Views handlers from `hook_entity_property_info()`. Expose every schema column you'd want to filter/sort/display:

```php
function mymodule_entity_property_info() {
  $info = array();
  $properties = &$info['mymodule_thing']['properties'];

  $properties['tid'] = array(
    'label'       => t('Thing ID'),
    'type'        => 'integer',
    'description' => t('The unique identifier.'),
    'schema field' => 'tid',
  );
  $properties['type'] = array(
    'label'       => t('Type'),
    'type'        => 'token',
    'options list' => 'mymodule_thing_type_options',
    'schema field' => 'type',
  );
  $properties['label'] = array(
    'label'       => t('Label'),
    'type'        => 'text',
    'schema field' => 'label',
  );
  $properties['created'] = array(
    'label'       => t('Created'),
    'type'        => 'date',
    'schema field' => 'created',
  );
  $properties['author'] = array(
    'label'       => t('Author'),
    'type'        => 'user',
    'getter callback' => 'entity_property_getter_method',
    'schema field' => 'uid',
  );

  return $info;
}
```

Property `type` values that matter: `text`, `integer`, `decimal`, `boolean`, `date`, `token` (for option lists), `user`, `node`, `<entity_type>` (for entity references).

After clearing cache, Views' "Add view" page will list your entity as a base table with all properties available as fields/filters/sorts.

### Ship a default listing view

Don't make site builders re-create the listing every install. Ship a default view as JSON.

1. Build the view in the UI on dev.
2. Export it: copy `files/config_<hash>/active/views.view.<name>.json` to `<module>/config/views.view.<name>.json`.
3. Register it in `hook_config_info()`.
4. Import it on install AND in an update hook (per the project rule):
   ```php
   function _mymodule_ensure_default_view() {
     $config = config('views.view.things_list');
     if ($config->isNew()) {
       config_install_default_config('mymodule', 'views.view.things_list');
     }
   }
   ```
5. Remove any hardcoded menu items for the path the view serves — Views owns it.

## Access callback pattern

Standard shape used across Drop Suite:

```php
function mymodule_thing_access($op, $entity = NULL, $account = NULL) {
  global $user;
  if (!$account) {
    $account = $user;
  }
  if (user_access('administer mymodule', $account)) {
    return TRUE;
  }
  // Op-specific checks: 'view', 'create', 'update', 'delete'.
  if ($op === 'view') {
    return user_access('access mymodule', $account);
  }
  return FALSE;
}
```

Permission name shape (Drop Suite convention):
- `administer <module>` — full control, restricted, implies all ops
- `access <module>` — baseline view + create (granted to team_member)
- `<action> <module>` — gated specific actions (e.g. `delete things`)

## Critical gotchas

- **`field_attach_view()` requires `field_attach_prepare_view()` first.** Without it, taxonomy term reference formatters crash with "Undefined array key taxonomy_term" the moment a taxonomy field is attached. Entity_plus's `view()` method handles this; only custom view callbacks need it explicitly.
- **`$user->roles` is values, not keys.** Iterate as values; `array_keys($account->roles)` returns integers and silently fails to match role machine names.
- **Backdrop has no `#type 'entityreference'` form element.** Use `#type => 'textfield'` plus `#autocomplete_path` pointing to a callback that returns label/id pairs. Entity reference *fields* (Field API) work fine; the standalone form widget doesn't exist.
- **No PSR-4 / no namespaces.** Register every class via `hook_autoload_info()`. Forgetting this gives a "Class not found" fatal at the worst time (install).
- **`hook_install()` mirrors update logic.** Anything an update hook creates (schema, default views, default config) must also exist in `hook_install()` via the same idempotent helper, or fresh sites and SimpleTest runs miss it.
- **Schema cache after new tables.** `bee cc all` may not flush the schema cache. After creating a new entity table, run `db_query("DELETE FROM {cache_bootstrap} WHERE cid = 'schema'")` if Backdrop reports the table as missing.

## Implementation checklist

Use this as the verification list before declaring the entity "done":

- [ ] Schema in `hook_schema()`, mirrored in `hook_install()` + `hook_update_NNNN()`
- [ ] `hook_entity_info()` with `fieldable: TRUE`, bundle admin paths, view modes
- [ ] Entity class and controller (extending `Entity` and `EntityPlusController`)
- [ ] `hook_autoload_info()` registers both classes
- [ ] `hook_field_extra_fields()` for non-Field-API rendered sections
- [ ] Entity view callback honours `field_extra_fields_get_display()`
- [ ] `hook_entity_property_info()` exposes every relevant column
- [ ] Default View shipped in `config/`, imported in install + update
- [ ] Access callback follows the `<module>_<entity>_access` shape
- [ ] Manage Fields + Manage Display tabs visible on the bundle admin page
- [ ] Views "Add view" page lists the entity as a base table
- [ ] Default view renders the listing with no manual configuration
