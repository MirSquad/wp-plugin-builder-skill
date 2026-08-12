# Changelog

All notable changes to this skill are recorded here. Entries are newest-first.

## 2026-08-12

- Added: **"Security guards must normalize input exactly the way the protected code does"** — a new section covering a class of bug where a guard inspects raw input while the code it protects normalizes first, so any value that normalizes *into* a blocked form slips through. Covers the two WordPress specifics that make it easy to hit: `sanitize_key()` lowercases *and* strips characters outside `[a-z0-9_-]`, and `WP_REST_Server::match_request_to_handler()` matches routes case-insensitively (`'@^' . $route . '$@i'`), so `/wp/v2/Users/5` reaches the same handler as `/wp/v2/users/5`. Includes wrong/right code for both, notes on what needs no special handling (`sanitize_text_field()` strips `%XX` sequences; `WP_REST_Request` never splits a query string out of a route), and the testing rule that guards must be exercised with mutations — and the suite itself validated by running it against the un-fixed code.

## 2026-08-06

- Added: "Standards this skill is based on" section, citing the primary sources the guidance defers to (Plugin Check, WordPress Coding Standards, PHP_CodeSniffer, WCAG 2 AA, OWASP, WP.org plugin guidelines).

## 2026-08-05

- Changed: sanitized the skill to be vendor-neutral so it is useful outside its original projects — generic author/plugin URI placeholders in the header template, and host-specific guidance generalized to "managed hosting".
- Added: the automated security audit gate (run it on every change and enforce it), plus generalized lessons on packaging, `.gitignore` from day one, `phpcs:ignore` for intentional unescaped output, version checks belonging in hooks rather than global scope, bulk operations needing a limit filter, and WordPress Abilities API integration.

## 2026-05-21

- Initial release — WordPress Plugin Builder skill for Claude Code, covering the plugin header, file structure, prefixing, i18n, output escaping, nonces, prepared SQL, readme.txt, Plugin Check compliance, lifecycle hooks, REST `permission_callback`, block editor compatibility, object caching, multisite, and privacy/GDPR hooks.
