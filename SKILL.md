---
name: plugin-builder
description: "WordPress plugin development standards for Miriam Schwab's plugin projects. Every plugin is built repo-ready from day one — assume WP.org submission is possible unless told otherwise. Use this skill whenever building, editing, or reviewing a WordPress plugin — including Lighthouse Scanner, Angie MCP for Yoast, Site Chat, or any new plugin. Triggers on: starting a new plugin, writing plugin PHP, REST endpoints, admin pages, plugin packaging, version bumps, plugin headers, readme.txt, i18n, output escaping, nonces, Plugin Check (PCP) compliance, or any plugin-specific architecture question. Also triggers when reviewing code before packaging a new zip. Do NOT skip this skill just because the task seems simple — version bumps, header reviews, capability checks, and sanitization rules apply even to small changes. MCP and Angie integration patterns are included in this skill but only apply when a plugin explicitly integrates with Angie."
---

# Plugin Builder

Standards and hard-won patterns for WordPress plugin development. Every plugin is built repo-ready from day one — assume it may be submitted to the WordPress.org plugin directory unless told otherwise.

This skill grows with every project — append new discoveries at the end of every session where something worth remembering was learned.

---

## Plugin header

Every plugin's main PHP file must begin with this header block. All fields below are either required by WP.org or strongly recommended. Do not omit fields — they affect review, SEO in the directory, and upgrade compatibility.

```php
<?php
/**
 * Plugin Name:       My Plugin Name
 * Plugin URI:        https://miriamschwab.me/plugins/my-plugin
 * Description:       A clear, one-sentence description. No "This plugin..." opener. Max ~140 chars.
 * Version:           1.0.0
 * Author:            Miriam Schwab
 * Author URI:        https://miriamschwab.me
 * License:           GPL-2.0-or-later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       my-plugin
 * Domain Path:       /languages
 * Requires at least: X.X  ← set based on code (see below)
 * Requires PHP:      X.X  ← set based on code (see below)
 */
```

**Rules:**
- `Author` is always "Miriam Schwab". `Author URI` is always `https://miriamschwab.me`.
- `Plugin URI` should point to `https://miriamschwab.me/plugins/[plugin-slug]`. If the plugin is later listed on WP.org, you can optionally update this to the WP.org listing URL. Keep it on your site until then — you own it and it works before any directory listing exists.
- `Plugin Name` must be unique and descriptive. No "WordPress", "WP", or trademarked terms unless you own them.
- `Version` must follow semantic versioning (`major.minor.patch`).
- `License` must be GPL-2.0-or-later (or compatible). WP.org requires a GPL-compatible license.
- `Text Domain` must exactly match the plugin's folder slug and the slug used in `load_plugin_textdomain()`.
- The header comment block must be in the main plugin file, not in an included file.

**Setting `Requires at least` (WordPress version):**

This should be the version that introduced the *newest* WordPress API used in the plugin. It's deterministic — not a guess. When writing code, track the most recently introduced WP function, hook, or class being used, and set the floor to that version. Add an inline comment so it's traceable:

```php
 * Requires at least: 6.2  // wp_cache_flush_runtime() added in 6.2
```

If no recent WP APIs are used (just core hooks and `$wpdb`, for example), 6.0 is a reasonable conservative floor for plugins being built today. Never set this lower than what you've verified actually works — it's a support commitment, not just metadata.

**Setting `Requires PHP`:**

Set this based on the newest PHP syntax feature used in the code. Common floors:

| Syntax used | Minimum PHP |
|---|---|
| No modern features, basic OOP | 7.4 |
| Arrow functions, typed properties | 7.4 |
| Named arguments, `match`, nullsafe `?->` | 8.0 |
| `readonly` properties, enums, fibers | 8.1 |

When writing a plugin, identify the highest-version PHP feature in use and set `Requires PHP` accordingly. Call this out when first writing the header so it's a deliberate decision, not an afterthought.

---

## File structure

Every plugin follows this structure. Consistency matters — it's what makes plugins reviewable, maintainable, and extensible.

```
my-plugin/
├── my-plugin.php          ← main plugin file (header lives here)
├── readme.txt             ← required for WP.org (see readme.txt section below)
├── uninstall.php          ← cleanup on deletion (alternative to register_uninstall_hook)
├── includes/              ← core PHP classes and logic
│   ├── class-my-plugin.php
│   └── class-my-plugin-admin.php
├── admin/                 ← admin-specific templates and partials
├── assets/
│   ├── css/
│   └── js/
└── languages/             ← .pot file and translations
```

**Every PHP file (except the main plugin file) must start with:**
```php
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
```
This prevents direct URL access. The main plugin file is the only exception — WordPress loads it directly.

---

## Prefixing

All functions, classes, constants, hooks, and option/meta keys must be prefixed with a unique slug derived from the plugin name. Without this, you will collide with other plugins or WordPress core.

```php
// Functions
function myplugin_do_something() {}

// Classes
class MyPlugin_Admin {}

// Constants
define( 'MYPLUGIN_VERSION', '1.0.0' );
define( 'MYPLUGIN_PLUGIN_DIR', plugin_dir_path( __FILE__ ) );
define( 'MYPLUGIN_PLUGIN_URL', plugin_dir_url( __FILE__ ) );

// Options and meta keys
update_option( 'myplugin_settings', $value );
update_post_meta( $post_id, '_myplugin_field', $value );

// Custom hooks
do_action( 'myplugin_after_save' );
apply_filters( 'myplugin_output', $output );
```

Use the same prefix consistently throughout the plugin. Two-word plugins: `myplugin_` not `my_plugin_` — shorter is cleaner.

---

## Internationalization (i18n)

Every user-facing string must be wrapped in a translation function. This is checked by Plugin Check and required for WP.org.

```php
// Simple string
__( 'Settings saved.', 'my-plugin' );

// Echo directly
_e( 'Settings saved.', 'my-plugin' );

// With variable substitution
sprintf( __( 'Hello, %s.', 'my-plugin' ), $name );

// Plurals
sprintf(
    _n( '%d item deleted.', '%d items deleted.', $count, 'my-plugin' ),
    $count
);
```

**Load the text domain on `init`:**
```php
add_action( 'init', 'myplugin_load_textdomain' );
function myplugin_load_textdomain() {
    load_plugin_textdomain( 'my-plugin', false, dirname( plugin_basename( __FILE__ ) ) . '/languages' );
}
```

**Rules:**
- The text domain string must be a literal string, not a variable — translation tools can't parse variables.
- Never concatenate strings to form translatable text. Each logical phrase is one string.
- Admin strings and frontend strings both need translation functions.

---

## Output escaping

Every value output to the browser must be escaped at the point of output. Sanitization on input is not a substitute. Plugin Check will flag unescaped output as a security error.

```php
// Plain text
echo esc_html( $title );

// HTML attribute
echo '<input value="' . esc_attr( $value ) . '">';

// URLs
echo '<a href="' . esc_url( $url ) . '">';

// Translatable strings (safe to output directly — already safe)
esc_html_e( 'Label text', 'my-plugin' );

// Integers
echo absint( $count );

// Allow limited HTML (e.g. for settings fields)
echo wp_kses_post( $content );
```

**Rule of thumb:** escape as late as possible — at the exact point of output, not earlier in the data flow. Use the most specific function (`esc_attr` for attributes, `esc_url` for URLs) rather than the most general.

---

## Nonces

Nonces protect against CSRF (cross-site request forgery). Required on all form submissions and AJAX requests. Plugin Check flags missing nonces on write operations.

**Form submission:**
```php
// In the form
wp_nonce_field( 'myplugin_save_settings', 'myplugin_nonce' );

// In the handler
if ( ! isset( $_POST['myplugin_nonce'] ) || ! wp_verify_nonce( sanitize_text_field( wp_unslash( $_POST['myplugin_nonce'] ) ), 'myplugin_save_settings' ) ) {
    wp_die( esc_html__( 'Security check failed.', 'my-plugin' ) );
}
```

**AJAX:**
```php
// Enqueue with localized nonce
wp_localize_script( 'myplugin-script', 'mypluginData', [
    'nonce' => wp_create_nonce( 'myplugin_ajax' ),
    'ajaxUrl' => admin_url( 'admin-ajax.php' ),
]);

// In the AJAX handler
check_ajax_referer( 'myplugin_ajax', 'nonce' );
```

---

## Prepared SQL

Never write raw SQL with string interpolation. Use `$wpdb->prepare()` for every query that includes a variable. Plugin Check flags raw queries as security errors, and WP.org reviewers will reject on sight.

```php
global $wpdb;

// SELECT with variable
$row = $wpdb->get_row(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}myplugin_items WHERE id = %d AND status = %s",
        $id,
        $status
    )
);

// INSERT
$wpdb->insert(
    $wpdb->prefix . 'myplugin_items',
    [
        'user_id' => $user_id,
        'value'   => $value,
    ],
    [ '%d', '%s' ]
);

// UPDATE
$wpdb->update(
    $wpdb->prefix . 'myplugin_items',
    [ 'status' => 'done' ],
    [ 'id' => $id ],
    [ '%s' ],
    [ '%d' ]
);

// DELETE
$wpdb->delete(
    $wpdb->prefix . 'myplugin_items',
    [ 'id' => $id ],
    [ '%d' ]
);
```

**Format specifiers:**
- `%d` — integer
- `%s` — string (quoted automatically)
- `%f` — float

**Creating plugin tables on activation:**

```php
function myplugin_create_tables() {
    global $wpdb;
    $charset = $wpdb->get_charset_collate();
    $sql = "CREATE TABLE IF NOT EXISTS {$wpdb->prefix}myplugin_items (
        id         BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT,
        user_id    BIGINT(20) UNSIGNED NOT NULL,
        value      VARCHAR(500) NOT NULL DEFAULT '',
        created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id),
        KEY user_id (user_id)
    ) $charset;";
    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql );
}
```

Use `dbDelta()` — never run `CREATE TABLE` directly. `dbDelta()` handles upgrades safely. Call it from `register_activation_hook` and also on version upgrade checks.

---

## readme.txt (required for WP.org)

Every plugin must have a `readme.txt` in its root. WP.org uses this file to populate the plugin directory listing. Missing or malformed readme is a common rejection reason.

```
=== Plugin Name ===
Contributors: authorslug
Tags: tag1, tag2, tag3
Requires at least: X.X
Tested up to: X.X
Requires PHP: X.X
Stable tag: 1.0.0
License: GPLv2 or later
License URI: https://www.gnu.org/licenses/gpl-2.0.html

One-sentence description matching the plugin header.

== Description ==

Full description of what the plugin does. Markdown-like formatting is supported.

== Installation ==

1. Upload the plugin folder to `/wp-content/plugins/`.
2. Activate from Plugins > Installed Plugins.
3. Configure under Settings > My Plugin.

== Frequently Asked Questions ==

= How do I configure X? =

Answer here.

== Changelog ==

= 1.0.0 =
* Initial release.

== Upgrade Notice ==

= 1.0.0 =
Initial release.
```

**Rules:**
- `Stable tag` must match the version in the plugin header exactly.
- `Tested up to` should reflect the latest WP version you've actually tested against.
- Tags are used for directory search — choose 1-5 specific, relevant terms.
- Contributors must be WordPress.org usernames (slugs), not display names.
- Keep the short description (above `== Description ==`) to one sentence under 150 chars.

---

## Plugin Check (PCP) compliance

Run Plugin Check before packaging any zip that might go to WP.org. Install it at `Tools > Plugin Check`. The "Plugin repo" category is the one that matters for directory approval — fix all errors in that category before submission. Warnings are advisory.

**The categories PCP checks:**
- **Plugin repo** — required for WP.org approval. Covers header format, readme.txt, licensing, file structure.
- **Security** — sanitization, escaping, nonces, capability checks, direct access prevention.
- **Performance** — database queries in loops, unoptimized asset loading, missing indexes.
- **Accessibility** — missing `alt` attributes, improper heading hierarchy, missing form labels.
- **i18n** — untranslated strings, wrong text domain, non-literal domain arguments.

**Common PCP failure points to pre-empt:**
- Missing or wrong `Text Domain` in header
- Hardcoded strings not wrapped in `__()`
- Unescaped output (`echo $variable` without `esc_html()`)
- Missing nonce verification on POST handlers
- Direct `$_GET`/`$_POST` access without sanitization
- `readme.txt` missing or `Stable tag` mismatched
- No `Requires PHP` or `Requires at least` in header
- Calling `wp_enqueue_scripts` outside a proper hook
- Using deprecated functions

**WP-CLI check (static only):**
```bash
wp plugin check my-plugin/my-plugin.php
```

**With runtime checks:**
```bash
wp plugin check my-plugin/my-plugin.php --require=./wp-content/plugins/plugin-check/cli.php
```

---

## Non-negotiables

These apply to every plugin, every session, no exceptions.

### Version bumps are mandatory

Any session that produces a new plugin zip must increment the version number before packaging. Never deliver a zip with the same version number as the previous one. If in doubt, bump it.

**Every location where the version must be updated:**
- Plugin header (`Version:` in the main `.php` file)
- Version constant (e.g. `define( 'MY_PLUGIN_VERSION', '1.x.x' )`)
- `package.json` — `version` field (if the plugin has a JS build)
- MCP server strings — version references in `createServer()` or `registerServer()` (if applicable)
- Handoff doc and project context doc headers

At wrap-up, always confirm the version was bumped before the zip was packaged. If it wasn't, flag it explicitly and output the corrected version strings.

### Review the plugin header at wrap-up

At the end of every session where code was written or modified, review the plugin header and check whether `Requires at least` or `Requires PHP` need updating.

**Ask:**
- Did any new code use a WordPress function, hook, or class introduced in a version newer than the current `Requires at least`?
- Did any new code use PHP syntax that requires a version newer than the current `Requires PHP`?

If yes to either, update the header field and document the reason inline:

```php
 * Requires at least: 6.3  // WP_HTML_Tag_Processor added in 6.2; using 6.3 for stability
 * Requires PHP:      8.0  // match expression used in class-my-plugin.php
```

This check happens at the same time as the version bump — it's part of the same wrap-up step, not a separate task.

### Capability checks on all write endpoints

Every REST endpoint that writes, deletes, or modifies data must check the appropriate WordPress capability before executing. Use `manage_options` for site-wide actions, `edit_post` for post-level actions. Never skip this even on internal-only endpoints.

```php
if ( ! current_user_can( 'manage_options' ) ) {
    return new WP_Error( 'forbidden', 'Insufficient permissions.', [ 'status' => 403 ] );
}
```

### Sanitize and length-limit all inputs

`sanitize_text_field()` strips tags and normalises whitespace but does not truncate. Always pair it with `mb_substr()` before storing. A malformed or adversarial tool call can otherwise write very large strings to the database.

```php
$value = mb_substr( sanitize_text_field( $raw ), 0, 500 );
```

### Try/catch all external calls

Any call to the Anthropic API, a REST endpoint, or any external service must be wrapped in try/catch. Return a user-readable error string — never let an unhandled exception propagate.

```typescript
try {
    const data = await restPost( '/endpoint', payload );
    return { content: [{ type: 'text', text: data.result }] };
} catch ( err ) {
    return { content: [{ type: 'text', text: `Error: ${ err instanceof Error ? err.message : 'Unknown error' }` }] };
}
```

### Uninstall hook cleans up all data

Every plugin that writes to the database (options, user meta, transients) must have an uninstall hook that removes all plugin data. Don't leave orphaned rows.

---

## Activation, deactivation, and uninstall hooks

Three separate lifecycle events — each has a distinct purpose and the right function to register it.

```php
// In the main plugin file
register_activation_hook( __FILE__, 'myplugin_activate' );
register_deactivation_hook( __FILE__, 'myplugin_deactivate' );
register_uninstall_hook( __FILE__, 'myplugin_uninstall' );

function myplugin_activate() {
    // Create DB tables, set default options, flush rewrite rules
    myplugin_create_tables();
    add_option( 'myplugin_version', MYPLUGIN_VERSION );
    flush_rewrite_rules();
}

function myplugin_deactivate() {
    // Remove scheduled events, flush rewrite rules
    // Do NOT delete data here — user may reactivate
    wp_clear_scheduled_hook( 'myplugin_daily_event' );
    flush_rewrite_rules();
}

function myplugin_uninstall() {
    // Delete all plugin data — user explicitly chose to remove
    delete_option( 'myplugin_settings' );
    delete_option( 'myplugin_version' );
    global $wpdb;
    $wpdb->query( "DROP TABLE IF EXISTS {$wpdb->prefix}myplugin_items" );
}
```

**Rules:**
- Activation: set up state, don't assume a fresh install (plugin may be reactivating — check if tables/options already exist).
- Deactivation: clean up runtime state only (scheduled hooks, rewrite rules). Never delete user data on deactivation.
- Uninstall: delete everything. This is the point of no return — user said delete.
- Use `uninstall.php` file as an alternative to `register_uninstall_hook` for complex cleanup — WP loads it in a clean context. If both exist, `uninstall.php` takes precedence.

**Schema version upgrades:** On activation, compare the stored version against `MYPLUGIN_VERSION`. If they differ, run `dbDelta()` again and update the stored version. This handles upgrades transparently.

```php
function myplugin_activate() {
    $installed_version = get_option( 'myplugin_version' );
    myplugin_create_tables(); // dbDelta is safe to re-run
    if ( $installed_version !== MYPLUGIN_VERSION ) {
        update_option( 'myplugin_version', MYPLUGIN_VERSION );
    }
    flush_rewrite_rules();
}
```

---

## CSS and JS delivery on Elementor Hosting

**Elementor Hosting's Cloudflare CDN caches static plugin assets by file path and ignores `?ver=` cache-busters.** This means `wp_enqueue_script()` and `wp_enqueue_style()` for plugin assets will silently serve stale files to visitors after an update.

**The workaround for frontend assets:** Output inline via `wp_footer` hook.

```php
add_action( 'wp_footer', function() {
    echo '<style>' . file_get_contents( plugin_dir_path( __FILE__ ) . 'assets/style.css' ) . '</style>';
    echo '<script>' . file_get_contents( plugin_dir_path( __FILE__ ) . 'assets/script.js' ) . '</script>';
} );
```

**Admin assets are different:** Admin pages are served dynamically and are not CDN-cached. Admin CSS and JS can be inlined via `readfile()` on the admin page output, or enqueued normally — both work. Using `readfile()` in the admin page callback is the pattern used in Lighthouse Scanner and is reliable.

**The only exception:** MCP server JS (`angie/dist/mcp-server.js`) can stay enqueued normally — it's loaded in the admin context, not the frontend.

Do not refactor inline output back to `wp_enqueue_*` for frontend assets without testing on the live site first.

---

## Angie MCP integration (only applies when a plugin integrates with Angie)

**This entire section is conditional.** Only apply these patterns if the plugin explicitly includes an Angie MCP integration. Do not reference MCP, TypeScript server patterns, or confirmation enforcement for plugins that have no Angie integration — they are irrelevant and will add unnecessary complexity.

**Adding Angie integration to an existing plugin:** If a plugin starts without Angie support and integration is added later, this section applies from that point forward. The integration doesn't change the plugin's core PHP architecture — it adds a separate MCP server layer (`angie/` folder with TypeScript source) alongside the existing code. Treat it as an additive layer, not a refactor.

### Scope the MCP script to relevant admin screens only

Don't load your MCP server JS on every admin page. Check `get_current_screen()->base` and only enqueue on screens where Angie will actually be used.

```php
add_action( 'admin_enqueue_scripts', function() {
    $screen = get_current_screen();
    if ( ! $screen || ! in_array( $screen->base, [ 'post', 'edit' ], true ) ) return;
    wp_enqueue_script( 'my-mcp-server', plugin_dir_url( __FILE__ ) . 'angie/dist/mcp-server.js', [], MY_PLUGIN_VERSION, true );
} );
```

### Enforcing user confirmation before write actions

Behavioral enforcement (instructions, parameter flags, structural tricks) cannot reliably prevent an agentic AI from executing tool sequences in a single response turn. The only reliable approach is a server-side timestamp check.

**The pattern:**
1. Preview tool stages the action in user meta with a `preview_time` timestamp
2. Apply tool checks elapsed time — rejects with 425 if under 30 seconds
3. On 425 rejection, do NOT delete the pending action — preserve it so the user can confirm without re-running the preview
4. Apply tool takes no parameters — looks up pending action by user ID only

```php
define( 'MY_MIN_CONFIRM_SECS', 30 );

// On preview
update_user_meta( $user_id, '_my_pending_action', [
    'expires'      => time() + 600,
    'preview_time' => time(),
    'data'         => $action_data,
] );

// On apply
$pending = get_user_meta( $user_id, '_my_pending_action', true ) ?: null;
$elapsed = time() - ( $pending['preview_time'] ?? 0 );

if ( $elapsed < MY_MIN_CONFIRM_SECS ) {
    // Do NOT delete pending here
    return new WP_Error(
        'confirmation_required',
        'STOP. Show the user the preview. Ask "Shall I apply these changes?" Wait for their response. Only call apply after they say yes.',
        [ 'status' => 425 ]
    );
}
```

**Why 30 seconds:** Two sequential tool calls in the same Angie response turn complete in milliseconds. A real user reading a preview and typing yes always takes more than 30 seconds. Time cannot be compressed by any instruction or structural trick.

### Store pending state in user meta, not transients

Transients are global. On multi-admin sites, transients can cause one user's pending action to interfere with another's. Always use `update_user_meta()` / `get_user_meta()` for per-user pending state.

### One pending action per user at a time

Multiple pending actions per user require tokens to differentiate — and returning tokens to Angie reintroduces the confirmation bypass problem. One pending entry per user, overwritten on each new preview call, keeps the apply step truly parameterless.

---

## Build and packaging

### MCP server build (Angie integrations only)

Only applies if the plugin has an Angie MCP integration. After any change to TypeScript source files:

```bash
cd angie
npm install
npm run build
```

Output: `angie/dist/mcp-server.js` — this is what WordPress loads. Always build before packaging.

### Plugin zip packaging

```bash
zip -r [plugin-name]-[version].zip [plugin-folder] \
  --exclude "*/node_modules/*" \
  --exclude "*/.DS_Store" \
  --exclude "[plugin-folder]/package-lock.json"
```

Stay in the parent directory when running the zip command. Exclude `node_modules` — never ship it.

---

## WordPress $submenu access validation — never remove entries after reparenting

If you flatten a plugin's submenu items into a new parent (moving them into a custom group or under a different top-level item), **do NOT remove the original `$submenu[$slug]` entries**.

WordPress's `user_can_access_admin_page()` validates page access on every admin page load by walking all of `$submenu` and looking for the current page slug. If the entry is gone, WordPress denies access with "Sorry, you are not allowed to access this page" — even if the user has the right capability and the URL is correct.

Orphaned entries (no corresponding `$menu` top-level entry pointing to them) are invisible in the rendered sidebar, so they won't cause double display. Leave them in place.

```php
// WRONG — causes "not allowed" access errors on sub-pages
if ( isset( $submenu[ $cs ] ) ) {
    foreach ( $submenu[ $cs ] as $s ) $subs[] = $s;
    unset( $submenu[ $cs ] ); // ← never do this after reparenting
}

// CORRECT — copy entries, leave originals in place
if ( isset( $submenu[ $cs ] ) ) {
    foreach ( $submenu[ $cs ] as $s ) $subs[] = $s;
    // Do NOT unset — WP's user_can_access_admin_page() needs these entries to validate access.
    // Orphaned entries (no $menu parent) are invisible in the sidebar.
}
```

---

## Normalize menu URLs when reparenting items

When moving a plugin's menu items under a new parent (e.g. nesting a plugin under Tools instead of its original top-level position), bare page slugs will produce broken URLs.

WordPress builds submenu URLs as `{parent-file}?page={slug}`. Under the plugin's original `admin.php`-based parent, a bare slug like `ai1wm_import` becomes `admin.php?page=ai1wm_import` — correct. Under `tools.php` it becomes `tools.php?page=ai1wm_import` — a frontend 404.

Convert all bare slugs to absolute `admin.php?page=SLUG` URLs before storing them in the new parent's submenu:

```php
private function normalize_menu_url( string $url ): string {
    if ( strpos( $url, '.php' ) !== false || strpos( $url, '://' ) !== false ) {
        return $url; // already absolute or external
    }
    return 'admin.php?page=' . $url;
}
```

Apply this to both the top-level item URL and every sub-item URL when flattening. Any URL already containing `.php` is already absolute and passes through unchanged. This is safe because all `admin.php?page=` plugin registrations either use bare slugs or full `admin.php` URLs — never relative paths to other `.php` files.

---

## Session opener addendum for plugin projects

When setting up a session opener for a plugin project, add these items to the standing instructions block, in addition to the standard work-docs wrap-up steps.

```markdown
## Plugin-specific wrap-up (add to standing instructions)

**Version bump:** Confirm the version number was incremented in every required location:
- Plugin header (`Version:`)
- Version constant (`define( 'MYPLUGIN_VERSION', ... )`)
- `package.json` if the plugin has a JS build
- MCP server strings if applicable
- Handoff doc and project context doc headers

**Header review:** Check whether `Requires at least` or `Requires PHP` need updating based on any new code written this session. If either changed, update the header and add an inline comment documenting why.

**PCP check (before packaging):** If a new zip was produced, confirm Plugin Check was run and all errors in the "Plugin repo" category were resolved before packaging.
```

These three items belong in every plugin project's session opener standing instructions, alongside the standard changelog, handoff, decisions log, and project context updates.

---

## Always add a Settings link in the plugin list

Every plugin that has a settings page must add a Settings link in the Plugins list page via the `plugin_action_links_` filter. This is a standing rule — never skip it.

```php
add_filter( 'plugin_action_links_' . plugin_basename( __FILE__ ), function ( $links ) {
    $settings_link = '<a href="' . esc_url( admin_url( 'options-general.php?page=my-plugin' ) ) . '">' . esc_html__( 'Settings', 'my-plugin' ) . '</a>';
    array_unshift( $links, $settings_link );
    return $links;
} );
```

Replace `options-general.php?page=my-plugin` with the correct admin URL for the plugin's settings page. Use `array_unshift` so the Settings link appears first (leftmost) in the action links row.

---

## `scrollIntoView` is unreliable inside `position:fixed` elements

`el.scrollIntoView()` does not reliably scroll a container that lives inside a `position:fixed` element. The browser may target the page body instead, leaving the scroll container unchanged.

**The fix:** Set `scrollTop` directly using `offsetTop`:

```javascript
requestAnimationFrame(function() {
    container.scrollTop = el.offsetTop - container.offsetTop;
});
```

Both `el.offsetTop` and `container.offsetTop` are relative to the same positioned ancestor (the fixed element), so the subtraction gives the element's position within the scroll container. `requestAnimationFrame` ensures the layout is settled before reading the value.

This came up in the AI Site Chat plugin: the chat messages area (`overflow-y: auto`) is inside the chat panel (`position: fixed`). `scrollIntoView({ block: 'start' })` scrolled the page body instead of the messages container. The `offsetTop` approach worked correctly.

---

## REST API — permission_callback is mandatory

Every REST route must declare a `permission_callback`. Returning `__return_true` is only acceptable for genuinely public, read-only data. For anything write-related or user-specific, check a capability.

```php
register_rest_route( 'myplugin/v1', '/items', [
    // Public read
    [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'myplugin_get_items',
        'permission_callback' => '__return_true',
    ],
    // Write — requires capability
    [
        'methods'             => WP_REST_Server::CREATABLE,
        'callback'            => 'myplugin_create_item',
        'permission_callback' => function() {
            return current_user_can( 'edit_posts' );
        },
        'args' => [
            'title' => [
                'required'          => true,
                'sanitize_callback' => 'sanitize_text_field',
                'validate_callback' => function( $val ) {
                    return is_string( $val ) && strlen( $val ) > 0;
                },
            ],
        ],
    ],
] );
```

**Use `args` for input sanitization on REST routes** — it's cleaner than doing it manually in the callback and PCP will flag missing sanitization on REST input.

Never omit `permission_callback` entirely — WP will throw a `_doing_it_wrong` notice and the REST API may reject the route on newer versions.

---

## Block editor compatibility

If the plugin does anything in the admin, verify it doesn't break the block editor. Common pitfalls:

**Enqueue scripts on the right hook:**
```php
// Block editor assets (only on block editor screens)
add_action( 'enqueue_block_editor_assets', 'myplugin_block_editor_assets' );
function myplugin_block_editor_assets() {
    wp_enqueue_script(
        'myplugin-block-editor',
        plugin_dir_url( __FILE__ ) . 'assets/js/block-editor.js',
        [ 'wp-blocks', 'wp-element', 'wp-editor' ],
        MYPLUGIN_VERSION,
        true
    );
}

// Admin assets — only on the plugin's own screens, not everywhere
add_action( 'admin_enqueue_scripts', function( $hook ) {
    if ( strpos( $hook, 'myplugin' ) === false ) return;
    wp_enqueue_style( 'myplugin-admin', plugin_dir_url( __FILE__ ) . 'assets/css/admin.css', [], MYPLUGIN_VERSION );
} );
```

**Don't load plugin scripts on every admin page.** Check `$hook` or `get_current_screen()->base` before enqueuing. Unnecessary scripts on the block editor will slow it down and may cause JS conflicts.

**Metaboxes in the block editor:** Classic metaboxes added with `add_meta_box()` still render in the block editor via a compatibility panel, but consider whether a sidebar plugin or block is a better fit for new work.

---

## Object caching

Use WordPress's object cache for expensive operations — database queries, remote API calls, computed results.

```php
// Cache a database query
function myplugin_get_items( $user_id ) {
    $cache_key   = 'myplugin_items_' . $user_id;
    $cache_group = 'myplugin';

    $items = wp_cache_get( $cache_key, $cache_group );
    if ( false !== $items ) {
        return $items;
    }

    global $wpdb;
    $items = $wpdb->get_results(
        $wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}myplugin_items WHERE user_id = %d",
            $user_id
        )
    );

    wp_cache_set( $cache_key, $items, $cache_group, HOUR_IN_SECONDS );
    return $items;
}

// Invalidate the cache when data changes
function myplugin_save_item( $user_id, $data ) {
    // ... save logic ...
    wp_cache_delete( 'myplugin_items_' . $user_id, 'myplugin' );
}
```

**Transients vs object cache:**
- `wp_cache_get/set` — in-memory, lives only for the current request (or longer if a persistent cache like Redis/Memcached is installed). Use for things that can be recomputed cheaply if missing.
- `set_transient` / `get_transient` — persisted to the database (unless a persistent cache is active). Use for things that are expensive to compute and should survive across requests.
- Don't use transients for per-user or per-request state — they're global and can collide.

---

## Multisite compatibility

If the plugin might run on a multisite install, a few patterns change.

```php
// Check if running on multisite
if ( is_multisite() ) {
    // Network-level option (all sites share it)
    $value = get_site_option( 'myplugin_network_setting' );
} else {
    $value = get_option( 'myplugin_setting' );
}

// Get the current blog's table prefix (changes per site on multisite)
global $wpdb;
$table = $wpdb->prefix . 'myplugin_items'; // $wpdb->prefix is already site-specific

// Network activation (activates on all sites at once)
register_activation_hook( __FILE__, 'myplugin_activate' );
// To handle network activation separately:
add_action( 'wpmu_new_blog', 'myplugin_new_blog_setup' );
function myplugin_new_blog_setup( $blog_id ) {
    switch_to_blog( $blog_id );
    myplugin_create_tables();
    restore_current_blog();
}
```

**Capability checks on multisite:** `manage_options` checks per-site admin. For network-level actions, use `manage_network_options`. Be explicit — don't assume an admin on one site can touch another.

If you don't intend to support multisite, add this to the plugin header:
```php
 * Network: false
```
This disables network activation in the UI and signals intent clearly.

---

## Privacy (GDPR)

If the plugin stores personal data (user IDs, names, emails, IPs, preferences), register erasure and export handlers. WP's privacy tools at Tools > Erase Personal Data / Export Personal Data call these hooks.

```php
// Register exporter
add_filter( 'wp_privacy_personal_data_exporters', 'myplugin_register_exporter' );
function myplugin_register_exporter( $exporters ) {
    $exporters['myplugin'] = [
        'exporter_friendly_name' => __( 'My Plugin Data', 'my-plugin' ),
        'callback'               => 'myplugin_export_user_data',
    ];
    return $exporters;
}

function myplugin_export_user_data( $email, $page = 1 ) {
    $user  = get_user_by( 'email', $email );
    $items = [];
    if ( $user ) {
        $data = get_user_meta( $user->ID, '_myplugin_data', true );
        if ( $data ) {
            $items[] = [
                'group_id'    => 'myplugin',
                'group_label' => __( 'My Plugin', 'my-plugin' ),
                'item_id'     => 'myplugin-' . $user->ID,
                'data'        => [ [ 'name' => __( 'Stored value', 'my-plugin' ), 'value' => $data ] ],
            ];
        }
    }
    return [ 'data' => $items, 'done' => true ];
}

// Register eraser
add_filter( 'wp_privacy_personal_data_erasers', 'myplugin_register_eraser' );
function myplugin_register_eraser( $erasers ) {
    $erasers['myplugin'] = [
        'eraser_friendly_name' => __( 'My Plugin Data', 'my-plugin' ),
        'callback'             => 'myplugin_erase_user_data',
    ];
    return $erasers;
}

function myplugin_erase_user_data( $email, $page = 1 ) {
    $user    = get_user_by( 'email', $email );
    $removed = false;
    if ( $user ) {
        delete_user_meta( $user->ID, '_myplugin_data' );
        $removed = true;
    }
    return [ 'items_removed' => $removed, 'items_retained' => false, 'messages' => [], 'done' => true ];
}
```

**Rule:** If the plugin stores anything that could identify or be linked to a person, it needs these hooks. WP.org reviewers increasingly check for privacy compliance.

---

## Plugin slug uniqueness

Before naming a plugin, verify the slug isn't already taken on WordPress.org. Search `wordpress.org/plugins/[slug]` directly. The plugin folder name, text domain, and option prefix all derive from the slug — changing it later touches every file. Do this check before writing a single line of code.

If the plugin is installed on a site and an existing WP.org plugin shares the slug, WordPress will show that other plugin's details in the plugin list. Fix this immediately with a `plugin_row_meta` filter to replace "View details" with a "Visit plugin site" link pointing to the author's own site:

```php
add_filter( 'plugin_row_meta', 'myplugin_plugin_row_meta', 10, 2 );
function myplugin_plugin_row_meta( $links, $file ) {
    if ( plugin_basename( MYPLUGIN_PLUGIN_FILE ) !== $file ) {
        return $links;
    }
    foreach ( $links as $key => $link ) {
        if ( strpos( $link, 'plugin-install.php' ) !== false ) {
            unset( $links[ $key ] );
        }
    }
    $links[] = '<a href="' . esc_url( 'https://miriamschwab.me/plugins/my-plugin' ) . '" target="_blank">' . esc_html__( 'Visit plugin site', 'my-plugin' ) . '</a>';
    return $links;
}
```

---

## .gitignore — required from day one

Every plugin repo must have a `.gitignore` created at the start. Never let a session end without one. The standard contents:

```gitignore
.DS_Store
.claude/
*.zip
analysis/
decisions-log.md
handoff.md
project-context.md
session-opener.md
```

**Why:** Pillar docs are development context, not source code. Zip files are build artifacts. Neither belongs in version control. If `.gitignore` is missing, pillar docs and zips can end up committed and pushed, exposing internal planning notes publicly.

---

## Packaging zips

When packaging an installable zip, always exclude:
- `.git/` and `.git/*` — never ship git history in an installable plugin
- `.gitignore`
- All pillar docs (`session-opener.md`, `handoff.md`, `project-context.md`, `decisions-log.md`, `CHANGELOG.md`)
- `README.md` (GitHub-facing; WP.org uses `readme.txt`)
- `analysis/` folder
- `*.zip` files
- `.DS_Store`

Zip from the plugins folder parent, not from inside the plugin folder. Always delete an existing zip before recreating — `zip -r` updates rather than replaces, so stale excluded files can persist.

```bash
cd /path/to/plugins
rm my-plugin/my-plugin-X.X.X.zip  # delete old first
zip -r my-plugin/my-plugin-X.X.X.zip my-plugin/ \
  -x "my-plugin/.git/*" \
  -x "my-plugin/.git" \
  -x "my-plugin/.gitignore" \
  -x "my-plugin/.DS_Store" \
  -x "my-plugin/**/.DS_Store" \
  -x "my-plugin/session-opener.md" \
  -x "my-plugin/handoff.md" \
  -x "my-plugin/project-context.md" \
  -x "my-plugin/decisions-log.md" \
  -x "my-plugin/CHANGELOG.md" \
  -x "my-plugin/README.md" \
  -x "my-plugin/analysis/*" \
  -x "my-plugin/*.zip"
```

**Always run the zip command from inside the plugin project folder** (the folder that *contains* the plugin folder), never from a grandparent directory. If you run zip from a grandparent, the zip gets a double-nested path (e.g. `my-plugin/my-plugin/my-plugin.php`) which causes WordPress to reject the upload with "No valid plugins were found."

Correct:
```bash
cd /path/to/plugins/my-plugin          # the project folder
zip -r my-plugin-X.X.X.zip my-plugin  # zips the plugin subfolder from one level up
```

Wrong:
```bash
cd /path/to/plugins                                      # grandparent — WRONG
zip -r my-plugin/my-plugin-X.X.X.zip my-plugin/my-plugin  # creates double-nested path
```

Store the zip inside the plugin's own folder, not in the parent plugins folder.

---

## phpcs:ignore for intentional unescaped output

When a plugin intentionally echoes unescaped content (serving raw text, not HTML — e.g. markdown output, llms.txt), Plugin Check and PHPCS will flag it as a security error. The correct fix is NOT to escape the content (that would corrupt it). Instead, suppress with an explanatory comment:

```php
// phpcs:ignore WordPress.Security.EscapeOutput.OutputNotEscaped -- Serving raw text/plain content, not HTML.
echo $content;
```

The comment must explain *why* it's safe to suppress. This satisfies Plugin Check and tells the next reviewer there was a deliberate decision here.

---

## Version checks belong in hooks, not global scope

Code that runs at plugin load time should be inside a hook, not at global scope. Running outside hooks can fire before WordPress is fully initialized.

```php
// Wrong — runs at global scope
$stored_version = get_option( 'myplugin_version' );
if ( $stored_version !== MYPLUGIN_VERSION ) {
    delete_transient( 'myplugin_cache' );
    update_option( 'myplugin_version', MYPLUGIN_VERSION );
}

// Correct — deferred to plugins_loaded
add_action( 'plugins_loaded', 'myplugin_check_version' );
function myplugin_check_version() {
    $stored = get_option( 'myplugin_version' );
    if ( $stored !== MYPLUGIN_VERSION ) {
        delete_transient( 'myplugin_cache' );
        update_option( 'myplugin_version', MYPLUGIN_VERSION );
    }
}
```

---

## Bulk operations need a limit filter

Any function that processes all posts (activation bulk generation, reprocessing tools) must have a filter to cap the count. On large sites, uncapped bulk operations can time out or exhaust memory.

```php
/**
 * Filters the maximum number of posts processed during bulk generation.
 * Set to a reasonable limit (e.g. 500) on large sites to avoid timeouts.
 * Default -1 processes all published posts.
 *
 * @param int $limit Posts per page. -1 for all.
 */
$limit = (int) apply_filters( 'myplugin_bulk_generate_limit', -1 );
$posts = get_posts( [
    'post_type'      => $post_types,
    'post_status'    => 'publish',
    'posts_per_page' => $limit,
    'fields'         => 'ids',
] );
```

---

## Growing this skill

At the end of every plugin session, check: did we discover anything worth adding here? This skill is only valuable if it stays current.

**What belongs here:**
- Hosting-specific constraints or gotchas
- WordPress patterns that worked well or caused unexpected problems
- Security patterns and escaping rules
- Performance or caching discoveries
- Any "I wish I'd known this at the start" moment

**What does NOT belong here:**
- Plugin-specific content (post copy, specific version numbers, project-specific decisions)
- One-off decisions unlikely to recur in other plugins

To update: append new findings under a clear `## Heading` at the bottom of this file. Don't edit existing content unless it's wrong — add new sections. Then repackage the skill and install it.

---

## WordPress Abilities API integration

When adding Abilities API support to a plugin, follow these patterns exactly. Learned through building abilities for four plugins (Admin Menu Manager, AI Site Chat, Lighthouse Scanner, LLM Markdown).

### File structure

Add a dedicated `includes/abilities.php` — never inline in the main plugin file.

```
my-plugin/
├── my-plugin.php
└── includes/
    └── abilities.php   ← new
```

Require it from the main plugin file **after all constants are defined**. This is critical — a fatal error occurs if the `require_once` uses a constant that hasn't been defined yet.

```php
define( 'MYPLUGIN_VERSION', '1.0.0' );
define( 'MYPLUGIN_DIR', plugin_dir_path( __FILE__ ) );
// ...all constants first, then:
require_once MYPLUGIN_DIR . 'includes/abilities.php';
```

### abilities.php template

```php
<?php
if ( ! defined( 'ABSPATH' ) ) exit;

// Bail silently on WP < 6.9 (no Abilities API).
if ( ! function_exists( 'wp_register_ability' ) ) {
    return;
}

add_action( 'wp_abilities_api_categories_init', 'myplugin_register_ability_category' );
function myplugin_register_ability_category() {
    wp_register_ability_category( 'my-plugin', array(
        'label'       => __( 'My Plugin', 'my-plugin' ),
        'description' => __( 'What these abilities do.', 'my-plugin' ),
    ) );
}

add_action( 'wp_abilities_api_init', 'myplugin_register_abilities' );
function myplugin_register_abilities() {

    wp_register_ability( 'my-plugin/get-data', array(
        'label'               => __( 'Get Data', 'my-plugin' ),
        'description'         => __( 'Returns plugin data.', 'my-plugin' ),
        'category'            => 'my-plugin',
        'output_schema'       => array( 'type' => 'object' ),
        'permission_callback' => fn() => current_user_can( 'manage_options' ),
        'execute_callback'    => function( $input = null ) {  // ← $input = null is required
            return array( 'key' => get_option( 'myplugin_data' ) );
        },
        'meta' => array(
            'mcp'        => array( 'public' => true ),        // ← mcp.public, not show_in_rest
            'annotations' => array(
                'readonly'    => true,
                'destructive' => false,
                'idempotent'  => true,
            ),
        ),
    ) );

    // Gate write abilities behind a settings checkbox.
    if ( ! get_option( 'myplugin_write_abilities', false ) ) {
        return;
    }

    wp_register_ability( 'my-plugin/update-data', array(
        // ...
        'meta' => array(
            'mcp'        => array( 'public' => true ),
            'annotations' => array( 'readonly' => false, 'destructive' => false, 'idempotent' => false ),
        ),
    ) );
}
```

### Two critical rules

**1. Use `meta.mcp.public: true`, not `show_in_rest`**

The MCP Adapter discovers abilities via `meta.mcp.public = true`. The `show_in_rest` key in `meta` is for the WordPress REST API and does NOT make abilities visible to the MCP Adapter. Using `show_in_rest` causes abilities to register correctly but never appear in the MCP tool list.

**2. Always use `$input = null` in execute_callback**

When a plugin has no `input_schema`, WordPress calls the execute_callback with zero arguments. PHP 8 throws a fatal error if the callback declares `function( $input )` (required argument). Always use `function( $input = null )` — the parameter is optional when no input schema is defined.

### Settings checkbox for write abilities

Register the option and add a checkbox to the settings page. Write abilities should always be off by default.

```php
// In admin_init:
register_setting( 'myplugin_settings', 'myplugin_write_abilities', [
    'sanitize_callback' => 'rest_sanitize_boolean',
] );

// In settings page HTML (within an existing settings_fields form):
<label>
    <input type="checkbox" name="myplugin_write_abilities" value="1"
        <?php checked( 1, get_option( 'myplugin_write_abilities', 0 ) ); ?> />
    <?php esc_html_e( 'Enable write abilities', 'my-plugin' ); ?>
</label>
```

Clean up on uninstall:
```php
delete_option( 'myplugin_write_abilities' );
```

### Async / long-running write abilities

For write abilities that take more than a few seconds (e.g. calling an external API for multiple URLs), return immediately and do the work via WP-Cron. The MCP Adapter has a request timeout that will kill synchronous long-running callbacks.

```php
// execute_callback — returns in milliseconds
'execute_callback' => function( $input = null ) {
    $strategy = $input['strategy'] ?? 'default';
    wp_schedule_single_event( time() - 1, 'myplugin_background_job', array( $strategy ) );
    spawn_cron(); // fires immediately without waiting for next page load
    return array(
        'started' => true,
        'message' => 'Job started. Check results in ~60 seconds.',
    );
},

// The actual work — hooked separately
add_action( 'myplugin_background_job', 'myplugin_do_background_job' );
function myplugin_do_background_job( $strategy ) {
    if ( function_exists( 'set_time_limit' ) ) set_time_limit( 300 );
    // ... do expensive work ...
}
```

Note: `spawn_cron()` makes a non-blocking request to `wp-cron.php`. On hosts where WP-Cron is fully disabled and replaced with a real server cron, `spawn_cron()` may not trigger immediately — but the job will still run on the next cron execution.

### add_query_arg with repeated params

`add_query_arg( array( 'category' => array( 'a', 'b' ) ), $url )` encodes as `category[0]=a&category[1]=b` — PHP array notation. Many external APIs (including Google PageSpeed) require repeated params: `category=a&category=b`. Build the URL manually when you need this:

```php
$url = 'https://api.example.com/endpoint'
    . '?url=' . rawurlencode( $target_url )
    . '&category=performance&category=accessibility'
    . ( $api_key ? '&key=' . rawurlencode( $api_key ) : '' );
```
