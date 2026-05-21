---
name: wp-plugin-builder
description: >
  WordPress plugin development standards. Every plugin is built repo-ready from day one — assume WP.org submission is possible unless told otherwise. Use this skill whenever building, editing, or reviewing a WordPress plugin. Triggers on: starting a new plugin, writing plugin PHP, REST endpoints, admin pages, plugin packaging, version bumps, plugin headers, readme.txt, i18n, output escaping, nonces, Plugin Check (PCP) compliance, or any plugin-specific architecture question. Do NOT skip this skill just because the task seems simple — version bumps, header reviews, capability checks, and sanitization rules apply even to small changes.
---

# WordPress Plugin Builder

Standards and hard-won patterns for WordPress plugin development. Every plugin is built repo-ready from day one — assume it may be submitted to the WordPress.org plugin directory unless told otherwise.

---

## Plugin header

Every plugin's main PHP file must begin with this header block. All fields below are either required by WP.org or strongly recommended. Do not omit fields — they affect review, SEO in the directory, and upgrade compatibility.

```php
<?php
/**
 * Plugin Name:       My Plugin Name
 * Plugin URI:        https://example.com/plugins/my-plugin
 * Description:       A clear, one-sentence description. No "This plugin..." opener. Max ~140 chars.
 * Version:           1.0.0
 * Author:            Your Name
 * Author URI:        https://example.com
 * License:           GPL-2.0-or-later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       my-plugin
 * Domain Path:       /languages
 * Requires at least: X.X  ← set based on code (see below)
 * Requires PHP:      X.X  ← set based on code (see below)
 */
```

**Rules:**
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

If no recent WP APIs are used (just core hooks and `$wpdb`, for example), 6.0 is a reasonable conservative floor for plugins being built today.

**Setting `Requires PHP`:**

Set this based on the newest PHP syntax feature used in the code. Common floors:

| Syntax used | Minimum PHP |
|---|---|
| No modern features, basic OOP | 7.4 |
| Arrow functions, typed properties | 7.4 |
| Named arguments, `match`, nullsafe `?->` | 8.0 |
| `readonly` properties, enums, fibers | 8.1 |

---

## File structure

Every plugin follows this structure:

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

Run Plugin Check before packaging any zip that might go to WP.org. Install it at `Tools > Plugin Check`. The "Plugin repo" category is the one that matters for directory approval — fix all errors in that category before submission.

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

Any session that produces a new plugin zip must increment the version number before packaging. Never deliver a zip with the same version number as the previous one.

**Every location where the version must be updated:**
- Plugin header (`Version:` in the main `.php` file)
- Version constant (e.g. `define( 'MY_PLUGIN_VERSION', '1.x.x' )`)
- `Stable tag:` in `readme.txt`
- `package.json` — `version` field (if the plugin has a JS build)

### Capability checks on all write endpoints

Every REST endpoint that writes, deletes, or modifies data must check the appropriate WordPress capability before executing. Use `manage_options` for site-wide actions, `edit_post` for post-level actions. Never skip this even on internal-only endpoints.

```php
if ( ! current_user_can( 'manage_options' ) ) {
    return new WP_Error( 'forbidden', 'Insufficient permissions.', [ 'status' => 403 ] );
}
```

### Sanitize and length-limit all inputs

`sanitize_text_field()` strips tags and normalises whitespace but does not truncate. Always pair it with `mb_substr()` before storing.

```php
$value = mb_substr( sanitize_text_field( $raw ), 0, 500 );
```

### Try/catch all external calls

Any call to an external API or service must be wrapped in try/catch. Return a user-readable error string — never let an unhandled exception propagate.

### Uninstall hook cleans up all data

Every plugin that writes to the database (options, user meta, transients) must have an uninstall hook that removes all plugin data. Don't leave orphaned rows.

---

## Activation, deactivation, and uninstall hooks

Three separate lifecycle events — each has a distinct purpose.

```php
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
- Uninstall: delete everything. This is the point of no return.
- Use `uninstall.php` file as an alternative to `register_uninstall_hook` for complex cleanup — WP loads it in a clean context.

**Schema version upgrades:** On activation, compare the stored version against `MYPLUGIN_VERSION`. If they differ, run `dbDelta()` again and update the stored version.

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

## REST API — permission_callback is mandatory

Every REST route must declare a `permission_callback`. Returning `__return_true` is only acceptable for genuinely public, read-only data.

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

Never omit `permission_callback` entirely — WP will throw a `_doing_it_wrong` notice.

---

## Always add a Settings link in the plugin list

Every plugin that has a settings page must add a Settings link in the Plugins list page via the `plugin_action_links_` filter.

```php
add_filter( 'plugin_action_links_' . plugin_basename( __FILE__ ), function ( $links ) {
    $settings_link = '<a href="' . esc_url( admin_url( 'options-general.php?page=my-plugin' ) ) . '">' . esc_html__( 'Settings', 'my-plugin' ) . '</a>';
    array_unshift( $links, $settings_link );
    return $links;
} );
```

---

## Block editor compatibility

If the plugin does anything in the admin, verify it doesn't break the block editor.

**Enqueue scripts on the right hook:**
```php
// Block editor assets (only on block editor screens)
add_action( 'enqueue_block_editor_assets', 'myplugin_block_editor_assets' );

// Admin assets — only on the plugin's own screens, not everywhere
add_action( 'admin_enqueue_scripts', function( $hook ) {
    if ( strpos( $hook, 'myplugin' ) === false ) return;
    wp_enqueue_style( 'myplugin-admin', plugin_dir_url( __FILE__ ) . 'assets/css/admin.css', [], MYPLUGIN_VERSION );
} );
```

**Don't load plugin scripts on every admin page.** Check `$hook` or `get_current_screen()->base` before enqueuing.

---

## Object caching

Use WordPress's object cache for expensive operations.

```php
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

---

## Multisite compatibility

If the plugin might run on a multisite install, a few patterns change.

```php
if ( is_multisite() ) {
    $value = get_site_option( 'myplugin_network_setting' );
} else {
    $value = get_option( 'myplugin_setting' );
}
```

**Capability checks on multisite:** `manage_options` checks per-site admin. For network-level actions, use `manage_network_options`.

If you don't intend to support multisite, add this to the plugin header:
```php
 * Network: false
```

---

## Privacy (GDPR)

If the plugin stores personal data, register erasure and export handlers. WP's privacy tools at Tools > Erase Personal Data / Export Personal Data call these hooks.

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

// Register eraser
add_filter( 'wp_privacy_personal_data_erasers', 'myplugin_register_eraser' );
function myplugin_register_eraser( $erasers ) {
    $erasers['myplugin'] = [
        'eraser_friendly_name' => __( 'My Plugin Data', 'my-plugin' ),
        'callback'             => 'myplugin_erase_user_data',
    ];
    return $erasers;
}
```

**Rule:** If the plugin stores anything that could identify or be linked to a person, it needs these hooks. WP.org reviewers increasingly check for privacy compliance.

---

## CDN caching gotcha on managed WordPress hosting

Some managed WordPress hosts configure their CDN to cache static assets by file path, **ignoring the `?ver=` query string** that WordPress uses for cache busting. This breaks the standard update mechanism for plugin CSS/JS files.

**Symptoms:** Plugin updates appear to install correctly but changes never take effect for visitors. The server has the new file; the CDN serves the old one.

**Workarounds when you can't access the CDN config:**

1. **CSS/JS for admin pages** — inline via `readfile()` in the admin page callback. Admin pages are served dynamically and never CDN-cached.
2. **Frontend CSS/JS** — output inline via `wp_footer` hook.
3. **File the bug** with your hosting provider. The correct fix is for them to respect query strings in cache keys.

---

## Build and packaging

### Plugin zip packaging

```bash
zip -r my-plugin-1.0.0.zip my-plugin \
  --exclude "*/node_modules/*" \
  --exclude "*/.DS_Store" \
  --exclude "*/package-lock.json"
```

Stay in the parent directory when running the zip command. Exclude `node_modules` — never ship it.

---

## WordPress $submenu access validation — never remove entries after reparenting

If you flatten a plugin's submenu items into a new parent, **do NOT remove the original `$submenu[$slug]` entries**.

WordPress's `user_can_access_admin_page()` validates page access on every admin page load by walking all of `$submenu`. If the entry is gone, WordPress denies access — even if the user has the right capability and the URL is correct.

Orphaned entries (no corresponding `$menu` top-level entry) are invisible in the rendered sidebar, so they won't cause double display. Leave them in place.

---

## Normalize menu URLs when reparenting items

When moving a plugin's menu items under a new parent, bare page slugs will produce broken URLs. Convert all bare slugs to absolute `admin.php?page=SLUG` URLs before storing them in the new parent's submenu:

```php
private function normalize_menu_url( string $url ): string {
    if ( strpos( $url, '.php' ) !== false || strpos( $url, '://' ) !== false ) {
        return $url;
    }
    return 'admin.php?page=' . $url;
}
```
