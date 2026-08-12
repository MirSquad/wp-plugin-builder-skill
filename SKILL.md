---
name: plugin-builder
description: "WordPress plugin development standards and hard-won patterns. Every plugin is built repo-ready from day one — assume WP.org submission is possible unless told otherwise. Use this skill whenever building, editing, or reviewing a WordPress plugin. Triggers on: starting a new plugin, writing plugin PHP, REST endpoints, admin pages, plugin packaging, version bumps, plugin headers, readme.txt, i18n, output escaping, nonces, Plugin Check (PCP) compliance, or any plugin-specific architecture question. Also triggers when reviewing code before packaging a new zip. Do NOT skip this skill just because the task seems simple — version bumps, header reviews, capability checks, and sanitization rules apply even to small changes."
---

# Plugin Builder

Standards and hard-won patterns for WordPress plugin development. Every plugin is built repo-ready from day one — assume it may be submitted to the WordPress.org plugin directory unless told otherwise.

This skill grows with every project — append new discoveries at the end of every session where something worth remembering was learned.

---

## Standards this skill is based on

The practices here distil the official WordPress and web standards for building secure, compliant, accessible plugins. When a specific rule is ambiguous, defer to the primary sources:

- **WordPress Plugin Check (PCP)** — <https://wordpress.org/plugins/plugin-check/>
- **WordPress Coding Standards (WPCS)** — <https://github.com/WordPress/WordPress-Coding-Standards>
- **PHP_CodeSniffer** — <https://github.com/PHPCSStandards/PHP_CodeSniffer>
- **WCAG 2 AA (accessibility)** — <https://www.w3.org/WAI/WCAG2AA-Conformance>
- **OWASP (application security)** — <https://owasp.org/>
- **WordPress.org Plugin Guidelines** — <https://developer.wordpress.org/plugins/wordpress-org/detailed-plugin-guidelines/>

---

## Before starting a new plugin or major feature

**Blind spot pass.** Before writing code for a new plugin or a substantial new feature (not small fixes), ask: "What are the unknown unknowns here — WP.org compliance traps, architecture decisions that are hard to reverse later, or patterns I should confirm before committing to an approach?" State this explicitly rather than jumping straight to code, especially for anything touching REST endpoints, custom tables, or a third-party integration you have not built against before.

**Implementation notes.** For any multi-session plugin build, keep a scratch `implementation-notes.md` (gitignored, not one of the five pillar docs) logging deviations from the original plan as they happen — e.g. "switched from transients to user meta because pending state needed to be per-user." Fold anything durable into the relevant pillar doc at wrap-up; anything durable enough to help future plugins goes into this skill file instead.

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
- Set `Author` and `Author URI` to your name and site, and keep them consistent across all your plugins.
- `Plugin URI` should point to a page you control for the plugin (e.g. `https://example.com/plugins/[plugin-slug]`). If the plugin is later listed on WP.org, you can optionally update this to the WP.org listing URL. Keep it on your own site until then — you own it and it works before any directory listing exists.
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
- `%i` — identifier (table/column name, auto-quoted with backticks) — **WordPress 6.2+ only**. Do not use `%i` unless `Requires at least` is 6.2 or higher. For older targets, whitelist identifiers manually instead of interpolating user input.

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

## Settings API

Use the WordPress Settings API for plugin options pages. It handles form rendering, saving, and nonce/capability checks when used correctly.

```php
add_action( 'admin_menu', 'myplugin_add_settings_page' );
function myplugin_add_settings_page() {
    add_options_page(
        __( 'My Plugin Settings', 'my-plugin' ),
        __( 'My Plugin', 'my-plugin' ),
        'manage_options',
        'my-plugin',
        'myplugin_render_settings_page'
    );
}

add_action( 'admin_init', 'myplugin_register_settings' );
function myplugin_register_settings() {
    register_setting( 'myplugin_settings_group', 'myplugin_option', [
        'type'              => 'string',
        'sanitize_callback' => 'sanitize_text_field',
        'default'           => '',
    ] );

    add_settings_section(
        'myplugin_main_section',
        __( 'General', 'my-plugin' ),
        '__return_null',
        'my-plugin'
    );

    add_settings_field(
        'myplugin_option_field',
        __( 'Option Label', 'my-plugin' ),
        'myplugin_render_option_field',
        'my-plugin',
        'myplugin_main_section'
    );
}

function myplugin_render_option_field() {
    $value = get_option( 'myplugin_option', '' );
    echo '<input type="text" name="myplugin_option" value="' . esc_attr( $value ) . '" class="regular-text">';
}

function myplugin_render_settings_page() {
    ?>
    <div class="wrap">
        <h1><?php echo esc_html( get_admin_page_title() ); ?></h1>
        <form method="post" action="options.php">
            <?php
            settings_fields( 'myplugin_settings_group' );
            do_settings_sections( 'my-plugin' );
            submit_button();
            ?>
        </form>
    </div>
    <?php
}
```

**Rules:**
- Always use `sanitize_callback` in `register_setting()` — it's your input validation layer.
- The first argument to `settings_fields()` and `register_setting()` must be the same option group string.
- Capability checks are handled by `add_options_page` (which requires `manage_options`), but if you build a custom save handler instead of using `options.php`, you must check capabilities yourself.

---

## Cron and scheduled tasks

If the plugin schedules WP-Cron events, follow these patterns.

```php
// Schedule on activation
function myplugin_activate() {
    if ( ! wp_next_scheduled( 'myplugin_daily_cleanup' ) ) {
        wp_schedule_event( time(), 'daily', 'myplugin_daily_cleanup' );
    }
}

// Unschedule on deactivation
function myplugin_deactivate() {
    wp_clear_scheduled_hook( 'myplugin_daily_cleanup' );
}

// The callback
add_action( 'myplugin_daily_cleanup', 'myplugin_run_cleanup' );
function myplugin_run_cleanup() {
    // ... cleanup logic ...
}
```

**Rules:**
- **Idempotency is mandatory.** Cron callbacks may run late, may run twice, or may overlap if execution is slow. Write the callback so that running it twice produces the same result as running it once.
- **Provide a manual trigger path.** Every cron task should be runnable on demand for debugging — either a WP-CLI command or a hidden admin action. If something goes wrong, you need to reproduce it without waiting for the schedule.
- **Always clear scheduled hooks on deactivation.** An orphaned cron event will fire fatal errors if the plugin is deactivated but the callback no longer exists.
- **Don't rely on exact timing.** WP-Cron is triggered by page visits, not a real system cron. On low-traffic sites, events may fire hours late.

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

## Automated security audit gate — run on every change, enforce it

The most effective way to keep a plugin secure over time is a security audit that runs on **every change** and blocks anything with a finding — so an escaping, nonce, sanitization, SQL, or redirect regression can never ship or be silently re-introduced. Two fast, free static passes cover it; neither needs a running WordPress install.

**1. PHPCS with the WordPress security sniffs** — output escaping, nonce/capability checks, input validation + sanitization, prepared SQL, safe redirects:

```bash
# one-time setup
composer global config allow-plugins.dealerdirect/phpcodesniffer-composer-installer true
composer global require squizlabs/php_codesniffer wp-coding-standards/wpcs \
  dealerdirect/phpcodesniffer-composer-installer

phpcs -p --standard=WordPress \
  --sniffs=WordPress.Security.EscapeOutput,WordPress.Security.NonceVerification,WordPress.Security.ValidatedSanitizedInput,WordPress.Security.SafeRedirect,WordPress.DB.PreparedSQL \
  --extensions=php --ignore='*/vendor/*,*/node_modules/*' path/to/plugin
```

These security sniffs are the non-negotiable floor. A full `--standard=WordPress` run (WordPress-Extra + `phpcompatibility/phpcompatibility-wp`) plus PHPStan with `szepeviktor/phpstan-wordpress` catches quality/compat/correctness bugs too and is well worth adding. Auto-fix the mechanical findings first with `phpcbf` (same `--standard`/`--sniffs`), then hand-fix the rest.

**2. A SAST pass** with Semgrep's PHP rules:

```bash
pip install semgrep
semgrep scan --config p/php --error --exclude vendor path/to/plugin
```

Treat every security finding as blocking — fix to zero before packaging.

### Enforce it so it can't be skipped

**CI gate.** A GitHub Actions workflow that runs on push/PR, and that the release workflow *depends on* so a failing audit blocks the release. `.github/workflows/security-audit.yml`:

```yaml
name: Security audit
on:
  push: { branches: [ main ] }
  pull_request:
  workflow_call:            # so release.yml can gate on it
permissions: { contents: read }
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with: { php-version: '8.2', coverage: none }
      - name: Install PHPCS + WPCS
        run: |
          composer global config allow-plugins.dealerdirect/phpcodesniffer-composer-installer true
          composer global require --no-interaction --no-progress \
            "squizlabs/php_codesniffer:^3.13" "wp-coding-standards/wpcs:^3.2" \
            "dealerdirect/phpcodesniffer-composer-installer:^1.0"
          echo "$(composer global config bin-dir --absolute)" >> "$GITHUB_PATH"
      - name: PHPCS — WordPress security rules
        run: |
          phpcs -p --standard=WordPress \
            --sniffs=WordPress.Security.EscapeOutput,WordPress.Security.NonceVerification,WordPress.Security.ValidatedSanitizedInput,WordPress.Security.SafeRedirect,WordPress.DB.PreparedSQL \
            --extensions=php --ignore='*/vendor/*,*/node_modules/*' .
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - name: Semgrep SAST (PHP)
        run: |
          pip install --quiet semgrep
          semgrep scan --config p/php --error --quiet --metrics off --exclude vendor .
```

Then make the release job depend on it in `release.yml`:

```yaml
jobs:
  audit:
    uses: ./.github/workflows/security-audit.yml
  release:
    needs: audit          # a failing audit blocks the release
    runs-on: ubuntu-latest
    steps: [ ... build + publish ... ]
```

**Local pre-push hook** — catches issues before they leave your machine. `.git/hooks/pre-push` (then `chmod +x`):

```bash
#!/usr/bin/env bash
# Block the push if the security audit fails. Emergency bypass: git push --no-verify
dir="."   # or the plugin subdirectory
phpcs -p --standard=WordPress \
  --sniffs=WordPress.Security.EscapeOutput,WordPress.Security.NonceVerification,WordPress.Security.ValidatedSanitizedInput,WordPress.Security.SafeRedirect,WordPress.DB.PreparedSQL \
  --extensions=php --ignore='*/vendor/*,*/node_modules/*' "$dir" || { echo "phpcs security check failed"; exit 1; }
command -v semgrep >/dev/null && { semgrep scan --config p/php --error --quiet --exclude vendor "$dir" || exit 1; }
```

Git hooks are not committed with the repo, so reinstall the pre-push hook after a fresh clone. The CI gate is the authoritative, always-on backstop; the hook is fast local feedback.

---

## Non-negotiables

These apply to every plugin, every session, no exceptions.

### The security audit gate is mandatory

Every change to plugin code must pass the security audit (PHPCS WordPress security sniffs + Semgrep SAST) **before a zip is built or anything ships** — this is what keeps escaping, nonce/capability, sanitization, prepared-SQL, and safe-redirect issues from being introduced or re-introduced. Security-category findings are always blocking; fix to zero, don't package around them. Automate it with the CI gate + pre-push hook so it can't be skipped. See "Automated security audit gate" above.

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

## Debugging checklist

Quick reference for common "it doesn't work" scenarios.

**Plugin doesn't load / fatal on activation:**
- Check the PHP error log and enable `WP_DEBUG_LOG` in `wp-config.php`: `define( 'WP_DEBUG_LOG', true );` — errors go to `wp-content/debug.log`.
- Confirm the plugin header is valid and in the correct main file.
- Look for syntax errors in any file loaded at boot time (the main file and anything it `require`s).

**Activation hook not firing:**
- The hook must be registered at top-level scope in the main plugin file — not inside another hook callback, not inside a class method called from a hook.
- Verify the file path matches: `register_activation_hook( __FILE__, ... )` only works in the main plugin file.
- On multisite, network activation runs differently — the hook fires once, not per-site.

**Settings not saving:**
- Confirm `register_setting()` is called (on `admin_init`).
- Confirm the option group in `register_setting()` matches the group passed to `settings_fields()`.
- Check capability: does the user have the required capability (usually `manage_options`)?
- Check nonce: is the form using `settings_fields()` which handles the nonce, or is a custom handler missing nonce verification?

**Security regressions (nonce present but still vulnerable):**
- Nonces prevent CSRF but don't check authorization. A valid nonce from a subscriber doesn't mean they should have access. Always pair nonce verification with `current_user_can()`.

**Cron task not running:**
- Confirm the event is scheduled: `wp_next_scheduled( 'myplugin_hook' )` should return a timestamp.
- WP-Cron depends on page visits. On low-traffic or staging sites, nothing triggers it. Set up a real system cron hitting `wp-cron.php` or test manually.
- Check that the callback is hooked: `add_action( 'myplugin_hook', 'myplugin_callback' )` must run on every request (not just admin), otherwise the callback won't exist when cron fires.

---

## CDN caching of plugin assets on managed hosting

**Some managed WordPress hosts front static assets with a CDN that caches by file path and ignores `?ver=` cache-busters.** On those hosts, `wp_enqueue_script()` and `wp_enqueue_style()` for plugin assets can silently serve stale files to visitors after an update.

**The workaround for frontend assets:** Output inline via the `wp_footer` hook.

```php
add_action( 'wp_footer', function() {
    echo '<style>' . file_get_contents( plugin_dir_path( __FILE__ ) . 'assets/style.css' ) . '</style>';
    echo '<script>' . file_get_contents( plugin_dir_path( __FILE__ ) . 'assets/script.js' ) . '</script>';
} );
```

**Admin assets are different:** Admin pages are served dynamically and are typically not CDN-cached. Admin CSS and JS can be inlined via `readfile()` on the admin page output, or enqueued normally — both work.

Do not refactor inline output back to `wp_enqueue_*` for frontend assets without testing on the live site first.

---

## Build and packaging

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

This came up in a chat-widget plugin: the chat messages area (`overflow-y: auto`) is inside the chat panel (`position: fixed`). `scrollIntoView({ block: 'start' })` scrolled the page body instead of the messages container. The `offsetTop` approach worked correctly.

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
    $links[] = '<a href="' . esc_url( 'https://example.com/plugins/my-plugin' ) . '" target="_blank">' . esc_html__( 'Visit plugin site', 'my-plugin' ) . '</a>';
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
archive/
*-decisions-log.md
*-handoff.md
*-project-context.md
*-session-opener.md
*-changelog.md
```

Use wildcard patterns for the pillar docs: some projects prefix them (e.g. `myplugin-handoff.md`), and a bare `handoff.md` pattern would miss those. The leading `*-` catches both prefixed and unprefixed names.

**Why:** Pillar docs are development context, not source code. Zip files are build artifacts. Neither belongs in version control. If `.gitignore` is missing, pillar docs and zips can end up committed and pushed, exposing internal planning notes publicly.

**A `.gitignore` rule does not untrack files already committed.** If the repo's first commits predate the rule, the ignore does nothing and the docs keep riding along on every push. Verify with `git ls-files | grep -E '\-handoff\.md$|\-decisions-log\.md$'` (or the project's actual naming pattern) and `git rm --cached` anything that matches.

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

## Ship a `.gitattributes` with `export-ignore` in every plugin repo

The release workflow's named asset (`<slug>-X.Y.Z.zip`) is clean, but GitHub also hands out the repo through the green "Code → Download ZIP" button and the auto-generated "Source code (zip/tar.gz)" assets attached to every release. All three are produced by `git archive`, which honours `export-ignore` from `.gitattributes` — so without that file, anyone who grabs a source zip gets `.github/`, `README.md`, `CLAUDE.md`, `.gitignore` and other repo cruft dumped into their plugin directory. This caused a real "fatal error on install" report (a user installed a source zip instead of the release asset).

Add a `.gitattributes` at the repo root marking every non-runtime path `export-ignore`:

```gitattributes
/.github        export-ignore
/.gitignore     export-ignore
/.gitattributes export-ignore
/README.md      export-ignore
/CHANGELOG.md   export-ignore
/CLAUDE.md      export-ignore
```

Include only the paths that are actually tracked in that repo. Verify before committing:

```bash
git archive --worktree-attributes --format=zip HEAD | bsdtar -tf - | awk -F/ '{print $1}' | sort -u
# should list only the plugin's own runtime paths
```

Two things to remember:
- **`export-ignore` is not retroactive.** GitHub builds a tag's source archive from that tag's tree, so only tags cut *after* the file exists get clean source archives. Committing to `main` fixes the "Download ZIP" button immediately; to give the latest *release* a clean source archive, cut a packaging-only patch release (version bump with no code change is fine — say so in the changelog).
- **The named `<slug>-X.Y.Z.zip` release asset is always the recommended install.** For repos where the plugin lives in a subdirectory (the workflow archives `HEAD:<subdir>`), the source zip is now cruft-free but still nests the plugin one level deep, so it isn't directly installable — the named asset is.

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

## When in doubt, check the Plugin Handbook

Before inventing a pattern for something WordPress already has an API for, consult the [Plugin Developer Handbook](https://developer.wordpress.org/plugins/). WordPress has built-in solutions for settings pages, cron, privacy exports, custom post types, REST endpoints, and more. Using the canonical approach means fewer surprises on updates and fewer PCP flags on review.

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

When adding Abilities API support to a plugin, follow these patterns exactly. Learned through building abilities for three production plugins.

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
---

## readme.txt FAQs must stay in sync with the code

readme.txt FAQ answers go stale when features change. Any time a feature is added or changed — especially around data storage, context limits, or default behaviour — check the FAQ for contradictions. Common staleness traps:

- "Nothing is stored" when logging was added later
- A character/post limit that was increased in a newer version
- Feature availability that changed

Stale FAQs that contradict the plugin's actual behaviour are a WP.org review flag. Audit the FAQ at the end of every session that changes user-visible behaviour.

---

## Public REST endpoints with manual nonce verification and full-page caching

When using `permission_callback => '__return_true'` for a public endpoint but also verifying a `wp_rest` nonce manually in the callback, document the caching behaviour with an inline comment. The nonce is generated in `wp_footer` and baked into the cached page HTML. For unauthenticated visitors, WordPress nonce validation is not session-specific, so cached nonces work across all visitors — the protection is anti-CSRF, not per-user authentication. Rate limiting is the primary abuse defence in this pattern.

```php
// wp_rest nonce is embedded in the page via wp_footer. For unauthenticated visitors
// this provides anti-CSRF protection but is shared across all page cache copies —
// cached nonces still validate correctly because WP's nonce validation for logged-out
// users is not session-specific. Rate limiting below is the primary abuse defence.
$nonce = isset( $_SERVER['HTTP_X_WP_NONCE'] ) ? sanitize_text_field( wp_unslash( $_SERVER['HTTP_X_WP_NONCE'] ) ) : '';
if ( ! wp_verify_nonce( $nonce, 'wp_rest' ) ) { ... }
```

---

## Source code must live in extracted files, not only inside zips

Plugin source PHP files must exist as unzipped files in the project directory. Only keeping source code inside `.zip` files makes it impossible to edit code without unzip/rezip cycles, search/grep across the codebase, or review diffs. Zips are build artifacts — store them in the plugin folder, exclude them from git via `.gitignore`, and always keep the extracted source alongside them.

---

## Translated strings passed to inline JavaScript

When a plugin's admin page uses inline `<script>` blocks (rather than enqueued files), translated strings needed by JavaScript cannot be passed via `wp_localize_script`. Instead, output a small PHP-generated object before the main script block:

```php
<script>
var myPluginI18n = {
    confirmDelete: '<?php echo esc_js( __( 'Are you sure you want to delete this?', 'my-plugin' ) ); ?>',
    labelRename:   '<?php echo esc_js( __( 'Rename', 'my-plugin' ) ); ?>'
};
</script>
<script>
jQuery(function($) {
    // use myPluginI18n.confirmDelete etc.
});
</script>
```

Use `esc_js()` on every value. Reference the object in JS instead of hardcoding English strings.

---

## readme.txt `Stable tag` must match the plugin header version

`Stable tag` in `readme.txt` tells WordPress.org which zip to serve to users. If it lags behind the plugin header version, the directory will serve an old zip even after a new one is submitted. Update `Stable tag` every time `Version:` changes in the plugin header — they must always be identical.

---

## Hook callback scope — don't set variables in one callback and read them in another

Variables set inside an `add_action()` callback (including closures) are in that callback's local scope only. They cannot be accessed in a separately-registered page-render function.

**Wrong pattern:**
```php
// Sets $scan_url — but this closure's scope ends when it returns.
add_action( 'admin_enqueue_scripts', function() {
    $scan_url = sanitize_url( $_GET['scan_url'] ?? '' );
    // ...
} );

// lhsc_render_page() cannot see $scan_url — it is undefined here.
function lhsc_render_page() {
    if ( $scan_url ) { ... } // PHP notice: undefined variable
}
```

**Correct pattern — re-read the source in the render function:**
```php
function lhsc_render_page() {
    $scan_url = '';
    if ( ! empty( $_GET['lhsc_scan_url'] ) ) { // phpcs:ignore ...
        $scan_url = esc_url_raw( wp_unslash( $_GET['lhsc_scan_url'] ) );
    }
    if ( $scan_url ) { ... }
}
```

Each callback must derive any values it needs from the original source (`$_GET`, `$_POST`, `get_option()`, etc.) — not from variables set in other callbacks.

---

## Uninstall cleanup for wildcard user meta keys

When a plugin stores user meta with a variable suffix (e.g. `_myplugin_pending_{post_id}`), `delete_metadata()` can only match exact keys — not patterns. Use a raw `$wpdb->prepare()` + LIKE query in `uninstall.php`:

```php
global $wpdb;
$wpdb->query( // phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- uninstall context, no alternative.
    $wpdb->prepare(
        "DELETE FROM {$wpdb->usermeta} WHERE meta_key LIKE %s",
        $wpdb->esc_like( '_myplugin_pending_' ) . '%'
    )
);
```

This is the correct pattern in an uninstall context where the alternative (iterating all users) would be far worse. Document the phpcs:ignore with a comment explaining why.

---

## Escape at point of output — not at assignment

Even if a variable was escaped when it was assigned, always re-escape it at the point of `echo` output. Relying on "I already escaped it" is fragile: someone may reassign the variable, refactor the code, or PCP/phpcs will flag the `echo` regardless because the sanitization is not "visible" at the output point.

```php
// Wrong — PCP will flag echo $scanner_url even though it was already escaped.
$scanner_url = esc_url( admin_url( '...' ) );
echo '<a href="' . $scanner_url . '">';

// Correct.
$scanner_url = admin_url( '...' );
echo '<a href="' . esc_url( $scanner_url ) . '">';
```

---

## Security guards must normalize input exactly the way the protected code does

A guard that blocks dangerous input is only as good as its agreement with the code downstream of it. If the guard inspects the raw input but the dispatcher normalizes it first, the two disagree — and every value that normalizes *into* a blocked form slips straight through. This is a general class of bug, and two WordPress specifics make it easy to hit.

**1. `sanitize_key()` lowercases *and* strips characters.** It is `strtolower()` followed by `preg_replace( '/[^a-z0-9_\-]/', '', ... )`. So `Force`, `FORCE`, `force ` (trailing space), `f o r c e` and `fo.rce` all normalize to `force`. A check against raw parameter names misses every one of them:

```php
// WRONG — checks raw keys, dispatches normalized keys.
if ( array_key_exists( 'force', $body ) ) {
    return new WP_Error( 'blocked', 'force is blocked.' );
}
foreach ( $body as $key => $value ) {
    $request->set_param( sanitize_key( $key ), $value ); // "Force" arrives as "force"
}

// CORRECT — normalize once, up front; check and dispatch the same array.
$body = myplugin_normalize_body( $body ); // sanitize_key() on every key
if ( array_key_exists( 'force', $body ) ) {
    return new WP_Error( 'blocked', 'force is blocked.' );
}
foreach ( $body as $key => $value ) {
    $request->set_param( $key, $value );
}
```

**2. WordPress matches REST routes case-insensitively.** `WP_REST_Server::match_request_to_handler()` matches with `preg_match( '@^' . $route . '$@i', $path )` — note the `i`. `/wp/v2/Users/5` resolves to exactly the same handler as `/wp/v2/users/5`. Any route pattern in a guard needs the same flag:

```php
// WRONG — /wp/v2/Users/5 walks past this and still reaches the endpoint.
if ( preg_match( '#^/wp/v2/users(/|$|\?)#', $route ) ) { ... }

// CORRECT — match the way WordPress itself matches.
if ( preg_match( '#^/wp/v2/users(/|$|\?)#i', $route ) ) { ... }
```

Uppercasing the whole namespace works too: the namespace pre-filter in that method uses case-sensitive `str_starts_with()`, so `/WP/V2/USERS` fails it, falls back to scanning every registered route, and the case-insensitive regex matches anyway.

**What does *not* need special handling.** `sanitize_text_field()` strips `%XX` percent-encoded sequences outright, so URL-encoding can't smuggle anything past a check that runs on the sanitized string — provided the guard and the dispatcher both operate on that same sanitized string. And `WP_REST_Request` never splits a query string out of a route, so a `?param=value` suffix in a route doesn't dispatch at all.

**The rule to apply generally:** normalize once, as early as possible, then check and use the same normalized value. Don't apply the transform in two places. A single normalization helper, called before the check, whose output is what actually gets used, makes divergence structurally impossible rather than merely currently-absent.

**Test guards with mutations, not just the canonical spelling.** A guard verified only with the exact input it expects is untested — that is precisely how both bugs above survived an earlier round of live testing that "passed". For every block, assert on casing and padding variants, and assert the allow side too (a `forced` parameter must still be permitted, or the guard is over-blocking). Then **validate the suite by running it against the un-fixed code**: if the tests don't fail there, they aren't testing anything.
