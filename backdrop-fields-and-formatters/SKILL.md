---
name: backdrop-fields-and-formatters
description: Build a custom Backdrop CMS field type — schema, widget(s), formatter(s), validation, and optionally field-level permissions. Use whenever the user asks to "create a field", "make a custom field type", "add a widget for X", "add a formatter for X", "store {a, b, c} as a single field row", or "control who can view/edit a field". Also use proactively when a data shape is "multiple sub-values per parent record that travel together" (color = {r,g,b}; checkbox row = {label, checked, weight}; address = {street, city, zip}) — that's a field, not a separate entity. Pair with backdrop-entity when building a fieldable entity that hosts the new field, and with backdrop-update-hook when the field schema needs to ship to existing sites.
---

# Backdrop custom field type

A complete Backdrop field has six working pieces:

1. **`hook_field_info()`** — declares the field type, label, default widget + formatter
2. **`hook_field_schema()`** — defines column shape per field row (in `.install`, NOT `.module`)
3. **`hook_field_widget_info()` + `hook_field_widget_form()`** — the edit-form widget(s)
4. **`hook_field_formatter_info()` + `hook_field_formatter_view()`** — the rendered output formatter(s)
5. **`hook_field_validate()` + `hook_field_is_empty()` + `hook_field_widget_error()`** — validation + emptiness rules
6. **`hook_field_access()`** — optional, gates view/edit per user (use `field_permission_example` shape)

Read the canonical examples first:
- `modules/contrib/examples-1.x-1.x/field_example/` — full field with 2 formatters, 3 widgets, validation, JS color picker
- `modules/contrib/examples-1.x-1.x/field_permission_example/` — minimum field + per-user view/edit access

This skill summarizes the patterns; reach for those files when shaping new code.

## When to use a field instead of a separate entity

A field is the right answer when:
- The data is a **bundle of sub-values that travel together** (color RGB, address parts, checkbox row)
- The data has **no independent identity** (you don't link to it, comment on it, or assign permissions per row at a granularity finer than the host)
- It needs **per-host independent storage** (each node has its own values; deleting the host deletes the field rows)

Use a separate entity when items need their own URL, permissions, comments, revisions, or cross-host references. The conventional escape hatch when a field row outgrows its scope: migrate it to a referenced entity. Plan for that, but don't pre-pay for it.

## Step 1 — `hook_field_info()`

Declares the field type. Reference: `field_example_field_info()`.

```php
function mymodule_field_info() {
  return array(
    'mymodule_thing' => array(
      'label'             => t('Thing'),
      'description'       => t('A thing with sub-parts that travel together.'),
      'default_widget'    => 'mymodule_thing_widget',
      'default_formatter' => 'mymodule_thing_default',
      // Optional: 'settings' => array('default_setting' => 'value'),
      // Optional: 'instance_settings' => array('per_instance_setting' => 'value'),
    ),
  );
}
```

`settings` are field-wide (set when the field is created); `instance_settings` are per-attachment. Use instance_settings for "this attachment of the field on this bundle behaves slightly differently" — e.g. a public-facing instance might allow open contributions while an internal-facing instance does not.

## Step 2 — `hook_field_schema()` (in `.install`, NOT `.module`)

Defines the columns for one field row. Backdrop's Field API will create a `field_data_<field_name>` table with these columns plus housekeeping (`entity_type`, `entity_id`, `revision_id`, `bundle`, `delta`, `language`).

Reference: `field_example_field_schema()` in `field_example.install`.

```php
function mymodule_field_schema($field) {
  return array(
    'columns' => array(
      'label'   => array('type' => 'varchar', 'length' => 512, 'not null' => FALSE),
      'checked' => array('type' => 'int', 'size' => 'tiny', 'not null' => TRUE, 'default' => 0),
      'weight'  => array('type' => 'int', 'not null' => TRUE, 'default' => 0),
    ),
    'indexes' => array(
      'checked' => array('checked'),
    ),
  );
}
```

**Critical:** this hook MUST live in your `.install` file. Backdrop won't find it in `.module`.

## Step 3 — Widget(s)

A field can have **multiple widgets**; the user picks one when attaching the field. Reference: `field_example` provides three (text, 3text, colorpicker) for the same field type.

```php
function mymodule_field_widget_info() {
  return array(
    'mymodule_thing_widget' => array(
      'label'       => t('Standard widget'),
      'field types' => array('mymodule_thing'),
    ),
    'mymodule_thing_compact_widget' => array(
      'label'       => t('Compact widget'),
      'field types' => array('mymodule_thing'),
    ),
  );
}

function mymodule_field_widget_form(&$form, &$form_state, $field, $instance, $langcode, $items, $delta, $element) {
  $value = isset($items[$delta]) ? $items[$delta] : array();
  $widget = $element;
  $widget['#delta'] = $delta;

  switch ($instance['widget']['type']) {
    case 'mymodule_thing_widget':
      $widget['label'] = array(
        '#type' => 'textfield',
        '#title' => t('Label'),
        '#default_value' => isset($value['label']) ? $value['label'] : '',
      );
      $widget['checked'] = array(
        '#type' => 'checkbox',
        '#default_value' => !empty($value['checked']),
      );
      // weight handled by field_collection-style drag, OR a hidden field
      break;
  }

  // Don't return $widget directly — assemble into $element keyed by column name.
  // Each form key here MUST match a schema column or a #element_validate that
  // converts to one (see the field_example 3text → rgb pattern).
  return $widget;
}
```

**Multi-column widget gotcha:** if your widget collects values that don't 1:1 match schema columns (e.g. three textfields → one `rgb` column), use `#element_validate` to assemble the final value via `form_set_value()`. See `field_example_3text_validate()` for the canonical implementation.

**Required fields gotcha:** Form API doesn't support `#required` on a fieldset. If your widget is a fieldset of multiple inputs, set `#required` on each leaf field individually based on `$instance['required']`.

## Step 4 — Formatter(s)

Multiple formatters allowed; the user picks one in Manage Display. Reference: `field_example` ships two (`field_example_simple_text`, `field_example_color_background`).

```php
function mymodule_field_formatter_info() {
  return array(
    'mymodule_thing_default' => array(
      'label'       => t('Default'),
      'field types' => array('mymodule_thing'),
    ),
  );
}

function mymodule_field_formatter_view($entity_type, $entity, $field, $instance, $langcode, $items, $display) {
  $element = array();
  switch ($display['type']) {
    case 'mymodule_thing_default':
      foreach ($items as $delta => $item) {
        $element[$delta] = array(
          '#type' => 'head_tag',
          '#tag'  => 'div',
          '#value' => check_plain($item['label']),
          '#attributes' => array('class' => array('mymodule-thing')),
          '#attached' => array(
            'css' => array(backdrop_get_path('module', 'mymodule') . '/mymodule.css'),
          ),
        );
      }
      break;
  }
  return $element;
}
```

**Always escape user input** in formatters — `check_plain()` for text, or use a render array element (`#tag`, `#attributes`) so Backdrop escapes for you. Never concatenate raw `$item` values into HTML strings.

## Step 5 — Validation + emptiness

Three hooks work together:

```php
// Tells Backdrop whether a row is "empty" (so required-field checks can fire).
function mymodule_field_is_empty($item, $field) {
  return empty($item['label']);
}

// Validates row content. Errors are added to $errors keyed by field/lang/delta.
function mymodule_field_validate($entity_type, $entity, $field, $instance, $langcode, $items, &$errors) {
  foreach ($items as $delta => $item) {
    if (!empty($item['label']) && strlen($item['label']) > 512) {
      $errors[$field['field_name']][$langcode][$delta][] = array(
        'error'   => 'mymodule_label_too_long',
        'message' => t('Label cannot exceed 512 characters.'),
      );
    }
  }
}

// Maps the error code from hook_field_validate() onto a form element.
function mymodule_field_widget_error($element, $error, $form, &$form_state) {
  switch ($error['error']) {
    case 'mymodule_label_too_long':
      form_error($element['label'], $error['message']);
      break;
  }
}
```

The `error` key is your own machine string — keep it module-prefixed and stable. Reference: `field_example_field_validate()` and `field_example_field_widget_error()`.

## Step 6 — Field-level access (optional)

Use when the field needs view/edit gates **separate from the host entity's permissions**. Reference: `field_permission_example` shows the full pattern.

```php
function mymodule_permission() {
  return array(
    'view own thing'  => array('title' => t('View own thing field')),
    'edit own thing'  => array('title' => t('Edit own thing field')),
    'view any thing'  => array('title' => t('View any thing field')),
    'edit any thing'  => array('title' => t('Edit any thing field')),
  );
}

function mymodule_field_access($op, $field, $entity_type, $entity, $account) {
  // Bail early if not our field — this hook fires for every field type.
  if ($field['type'] !== 'mymodule_thing') {
    return TRUE;
  }
  // Superuser shortcuts.
  if (user_access('bypass node access', $account)) {
    return TRUE;
  }
  // 'own' vs 'any' context — only meaningful for entities with a uid.
  $context = 'any';
  if ($entity && isset($entity->uid) && $entity->uid == $account->uid) {
    $context = 'own';
  }
  return user_access("$op $context thing", $account);
}
```

`hook_field_access()` is called for **every field** on every entity render and edit; the `if ($field['type'] !== ...)` early return is mandatory or you'll grant/deny on fields you don't own.

If the field needs a write path **outside** standard view/edit (e.g. an authenticated user adds a row on a published node without taking edit-lock on the host), `hook_field_access()` is the wrong tool — it only gates the host entity's normal flow. Define explicit AJAX menu paths in `hook_menu()` with their own access checks for the sidecar write path. Use both: hook_field_access for standard view/edit, custom permission for sidecar writes.

## Common pitfalls

- **Schema lives in `.install`.** `hook_field_schema()` in `.module` will be invisible. Backdrop only looks for it in `.install`.
- **Widget keys must match schema columns** — or use `#element_validate` to convert. Submitting a value at `$form_state['values']['my_field']['und'][0]['unknown_key']` silently drops it.
- **Multiple deltas:** for cardinality > 1, the widget runs once per delta. The form is built per-row. Reordering / removing happens via Field API's standard infrastructure — don't reinvent it unless you have a reason.
- **Required fieldsets:** Form API doesn't honor `#required` on a fieldset. Apply to leaf elements.
- **`hook_field_access` runs for every field.** Bail early on `$field['type']`.
- **Render array escaping:** if you build raw HTML strings in a formatter, you've lost. Use `#type`, `#tag`, `check_plain()`, or a theme function. Per project CLAUDE.md, `<input>` and HTML in form `#options` is also a hazard.
- **Audit columns:** if your field will track who-added-what or who-changed-what, include `created_uid`, `created_at`, `changed_uid`, etc. in the schema from v1. Retrofitting these later means a multi-row data migration that's nasty to write idempotently.
- **Field UI inline-form widget for compound types** is well-trodden but tricky. Look at how `link` and `field_collection` widgets are structured before reinventing.

## When to use this skill

Trigger when:
- Building a new field type (matching `<topic>_example/` reference is `field_example` or `field_permission_example`)
- Adding a widget or formatter to an existing field type
- Adding field-level access control (view/edit per user beyond host entity access)
- Reviewing field code for missing validation or emptiness rules

Skip when:
- The user wants to **attach an existing field** to a content type — that's Field UI, no code needed
- The user wants to **define a content type** — that's `node_type_example` and the `backdrop-entity` skill territory
- The user wants a **standalone admin form** — that's `form_example` and FAPI, not the Field API
