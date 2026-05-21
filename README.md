# WordPress Plugin Builder — Claude Code Skill

A Claude Code skill that enforces WordPress plugin development standards. Built from real plugin projects submitted to WP.org — every pattern in this skill was learned the hard way.

## What this is

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) — a markdown file that teaches Claude how to build WordPress plugins correctly. When installed, Claude automatically follows these standards whenever you're working on a WordPress plugin.

## What it covers

- **Plugin header** — every required and recommended field, with rules for setting `Requires at least` and `Requires PHP` based on actual code
- **File structure** — standard layout for WP.org-ready plugins
- **Prefixing** — consistent namespacing for functions, classes, constants, hooks, and option keys
- **Internationalization (i18n)** — translation function patterns and text domain rules
- **Output escaping** — `esc_html`, `esc_attr`, `esc_url`, `wp_kses_post` at the point of output
- **Nonces** — CSRF protection for forms and AJAX
- **Prepared SQL** — `$wpdb->prepare()` patterns for SELECT, INSERT, UPDATE, DELETE, and table creation with `dbDelta()`
- **readme.txt** — WP.org directory listing format with all required fields
- **Plugin Check (PCP)** — how to run it and the most common failure points to pre-empt
- **Activation, deactivation, uninstall hooks** — lifecycle management with schema version upgrades
- **REST API** — `permission_callback`, input sanitization via `args`, capability checks
- **Block editor compatibility** — enqueue on the right hooks, don't break Gutenberg
- **Object caching** — `wp_cache_get/set` vs transients, cache invalidation patterns
- **Multisite** — `get_site_option`, network capability checks, per-site table prefixes
- **Privacy (GDPR)** — personal data export and erasure hooks
- **CDN caching gotchas** — workarounds for managed hosts that ignore `?ver=` cache busters

## Installation

### Option 1: Copy the skill file

Copy `SKILL.md` into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp SKILL.md .claude/skills/wp-plugin-builder.md
```

### Option 2: Reference from your CLAUDE.md

Add a pointer in your project's `CLAUDE.md`:

```markdown
## Skills
- See `.claude/skills/wp-plugin-builder.md` for WordPress plugin development standards
```

Claude Code will automatically load the skill when working in your project directory.

## Usage

Once installed, Claude will automatically apply these standards when:
- You ask it to build a new WordPress plugin
- You're editing plugin PHP files
- You ask about REST endpoints, admin pages, or plugin packaging
- You're preparing a plugin for WP.org submission
- You're doing version bumps, security reviews, or PCP compliance checks

## Origin

Built and maintained by [Miriam Schwab](https://miriamschwab.me), refined across multiple WordPress plugin projects including plugins published on WordPress.org. Every rule and pattern in this skill was discovered through real development — bugs found, reviews failed, workarounds proven.

## License

MIT
