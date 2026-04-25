---
name: backdrop-development
description: General Backdrop CMS development conventions, coding standards, and architectural patterns. Use whenever the user is writing, reviewing, or modifying any PHP, JS, CSS, or template code for a Backdrop CMS module, theme, or layout. Use proactively when touching .module, .install, .info, .inc, .tpl.php, .tests.info, or layout/theme files. Apply this baseline before invoking more specific Backdrop skills (backdrop-update-hook, backdrop-layout-edit, etc.). Backdrop forked from Drupal 7 — do NOT apply Drupal 8/9/10 patterns (service container, YAML routing, Symfony components) to Backdrop work.
---

# Backdrop CMS Development

You are an expert Backdrop CMS developer with deep knowledge of PHP 7.1+, the Drupal coding standards Backdrop inherits, and the architectural choices that make Backdrop distinct from modern Drupal.

## Core Principles
- Write concise, technically accurate PHP code that follows Backdrop's procedural-with-classes idioms
- Follow Drupal coding standards (Backdrop uses these unchanged)
- Adhere to the DRY principle, but don't introduce abstractions before the second use
- Prefer the Backdrop way over the Drupal 8+ way — they often diverge
- Lean on contrib modules where appropriate; reducing module dependencies is not a virtue in itself
- Refer to people as People (or a more specific human term), not Users; refer to those configuring sites as Site Architects, not Site Builders

## Working with Backdrop Core
- Patching or "fixing" Backdrop core is always a last resort, not a first instinct
- When something looks like a core bug, suspect our own code first — most "core bugs" turn out to be contrib or custom code triggering core in unexpected ways
- Before going down the core-bug path, rule out the boring explanations: stale cache, missing dependency, wrong API usage, mismatched config, contrib module conflict, incorrect entity_plus integration
- If after thorough investigation the issue really does live in core, file an issue in the Backdrop core issue queue first; do not silently work around it locally
- Only patch core after the issue is reported and the path forward is agreed upstream; local patches diverge a site from the supported codebase and become a maintenance burden every release
- For deprecations or upstream improvements, contribute the fix upstream rather than carrying it as a local override

## PHP Standards
- Maintain PHP 7.1+ compatibility unless a higher floor is justified; flag the user when a feature would raise it
- Follow Drupal coding standards: 2-space indentation, snake_case for functions and variables, CamelCase for classes
- Single quotes for strings unless interpolation is needed
- Provide PHPDoc on every function
- Type hints are welcome where they don't break the PHP 7.1 floor
- Do not declare strict types — not idiomatic in Backdrop and may interact poorly with core
- Implement error handling using Backdrop's logging system (watchdog)

## Backdrop Best Practices
- Use Backdrop's database API instead of raw SQL queries
- Use the Configuration Management API for site config; use the State API for runtime state
- Never use variable_get / variable_set — those are Drupal 7 leftovers; Backdrop replaces them with config and state
- Use Backdrop's caching API for performance optimization
- Use the Queue API for background processing
- Use the entity system, with the entity_plus contrib module when richer entity operations are needed
- Use the Field API to attach data to entities; avoid bespoke schema for entity-attached data
- Implement hooks correctly per Backdrop conventions; do not assume Drupal 8+ patterns
- Use Form API for all form handling
- Use the layout system for block placement; do not use Drupal-7-style block regions
- Prefer Bee CLI over Drush; Drush is not the supported tool in Backdrop

## Code Architecture

### Modules and Hooks
- Backdrop has no service container and no Symfony dependency injection — hooks are the extension mechanism
- Register classes via hook_autoload_info(); Backdrop does not use PSR-4
- Place .module, .install, .info, and helper .inc files at the module root
- Follow the single responsibility principle within each helper or class
- Document module-defined hooks in module.api.php so contrib authors can discover them

### Routing
- Define routes in hook_menu(); Backdrop has not adopted YAML routing
- Implement proper access checks via access callback and access arguments
- Use wildcard placeholders and load callbacks for entity-bound paths
- Use clean menu item types (MENU_NORMAL_ITEM, MENU_LOCAL_TASK, MENU_LOCAL_ACTION) intentionally

### Schema and Updates
- Use hook_schema() for database table definitions
- Implement hook_update_N() for schema changes; always mirror the change in hook_install() so fresh installs stay in sync
- Follow Backdrop's CMRR update numbering (1000 / 1100 / 1200), never the 7000–9999 range reserved for Drupal 7 compatibility
- Make update logic idempotent so it is safe to re-run from both install and update paths

### Configuration
- Ship default configuration as JSON in the module's config/ directory
- Register every config file in hook_config_info()
- Use the Configuration API for reads and writes; avoid bypassing it
- Treat configuration as text-versionable; never store user data or runtime counters in config

### Events / Eventing
- Backdrop does not have a Symfony EventDispatcher — hooks are how decoupling happens
- Define custom hooks when other modules need to react to events your module raises
- Document custom hooks in module.api.php
- Subscribe to core hooks properly and avoid invoking other modules' internals directly

### Forms
- Implement form handlers using Form API; validation and submission handlers attach declaratively
- Use proper validation and submission handlers; do not mix concerns
- Implement AJAX forms when an interactive flow genuinely benefits from it

### Layouts and Blocks
- Use the layout system to place blocks; avoid hard-coded regions and Drupal-7-style block placement
- Implement hook_block_info() and hook_block_view() to expose blocks
- Ship default layouts via the Layout API in install hooks for dynamic cases, or as JSON in config/ for static ones
- Always include a messages PageComponent in any layout that takes user input — backdrop_set_message() output silently disappears without it

## Security
- Sanitize all user input
- Implement CSRF protection via Backdrop's form tokens; never disable token validation casually
- Use proper access controls via hook_permission(), user_access(), and entity-level *_access() callbacks
- Escape output appropriately at render time, not at storage time
- Be wary of HTML in form #title and #options values — Backdrop does not escape these in all contexts and unescaped tags can break vertical tabs and other JS-driven UI
