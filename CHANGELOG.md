# Changelog

All notable changes to this skill are recorded here. Entries are newest-first.

## 2026-08-12

- Changed: removed the last named plugins from the skill. The Abilities API section cited four plugins as provenance — one of which no longer exists — and the `scrollIntoView` section named a specific plugin. Both now describe the source generically ("three production plugins", "a chat-widget plugin"); the technical guidance is unchanged, since the names were sourcing rather than instruction.
- Added: **Settings API** — registering options pages with `register_setting()`, `add_settings_section()`, `add_settings_field()`, and the rules that make the sanitize callback and option group line up.
- Added: **Cron and scheduled tasks** — scheduling and clearing events, and why cron callbacks must be idempotent and manually triggerable.
- Added: **Debugging checklist** — quick reference for plugin-won't-load, activation-hook-not-firing, settings-not-saving, and cron-not-running.
- Added: **Before starting a new plugin or major feature** — the blind-spot pass, and keeping scratch implementation notes across a multi-session build.
- Added: **Ship a `.gitattributes` with `export-ignore` in every plugin repo** — `git archive` powers the "Download ZIP" button and every release's auto-generated source assets, so without it a downloaded source zip carries CI config and dev docs into the user's plugin directory. Includes the caveat that `export-ignore` is not retroactive.
- Added: **When in doubt, check the Plugin Handbook** — prefer the canonical WordPress API over inventing a pattern.
- Added: the `%i` identifier placeholder caveat to Prepared SQL (WordPress 6.2+ only).
- Changed: `.gitignore` section now notes that an ignore rule does **not** untrack files already committed, with the `git ls-files` check to catch it, and adds `archive/` and `*-changelog.md` to the standard contents.
- Added: **"Security guards must normalize input exactly the way the protected code does"** — a new section covering a class of bug where a guard inspects raw input while the code it protects normalizes first, so any value that normalizes *into* a blocked form slips through. Covers the two WordPress specifics that make it easy to hit: `sanitize_key()` lowercases *and* strips characters outside `[a-z0-9_-]`, and `WP_REST_Server::match_request_to_handler()` matches routes case-insensitively (`'@^' . $route . '$@i'`), so `/wp/v2/Users/5` reaches the same handler as `/wp/v2/users/5`. Includes wrong/right code for both, notes on what needs no special handling (`sanitize_text_field()` strips `%XX` sequences; `WP_REST_Request` never splits a query string out of a route), and the testing rule that guards must be exercised with mutations — and the suite itself validated by running it against the un-fixed code.

## 2026-08-06

- Added: "Standards this skill is based on" section, citing the primary sources the guidance defers to (Plugin Check, WordPress Coding Standards, PHP_CodeSniffer, WCAG 2 AA, OWASP, WP.org plugin guidelines).

## 2026-08-05

- Changed: sanitized the skill to be vendor-neutral so it is useful outside its original projects — generic author/plugin URI placeholders in the header template, and host-specific guidance generalized to "managed hosting".
- Added: the automated security audit gate (run it on every change and enforce it), plus generalized lessons on packaging, `.gitignore` from day one, `phpcs:ignore` for intentional unescaped output, version checks belonging in hooks rather than global scope, bulk operations needing a limit filter, and WordPress Abilities API integration.

## 2026-05-21

- Initial release — WordPress Plugin Builder skill for Claude Code, covering the plugin header, file structure, prefixing, i18n, output escaping, nonces, prepared SQL, readme.txt, Plugin Check compliance, lifecycle hooks, REST `permission_callback`, block editor compatibility, object caching, multisite, and privacy/GDPR hooks.
