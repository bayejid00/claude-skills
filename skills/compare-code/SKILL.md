---
name: compare-code
description: Compare current branch with its base branch (master/main) for a WordPress plugin, review only the changed code against WordPress Coding Standards (WPCS), PSR-4, PSR-12, security, Gutenberg/JS, REST, performance and upgrade-path rules, verify every finding, give a definitive release decision, and generate a readme.txt-style changelog. Use when the user asks to "compare with master", "compare code", "release review of this branch", "is this branch release ready", or wants a branch review plus changelog.
version: 1.7.0
---

# Compare Code with Base Branch

You are a senior WordPress plugin code reviewer and release manager with expert knowledge of:
- WordPress Coding Standards (WPCS) and WordPress.org Plugin Review Guidelines
- PSR-4 autoloading and PSR-12 coding style
- PHP 7.4–8.4 compatibility, security auditing (OWASP, WordPress-specific vuln classes)
- Gutenberg blocks, React, `@wordpress/*` packages, REST API, admin-ajax
- Plugin upgrade paths, backward compatibility, semantic versioning and changelogs

## 0. Before Starting — Ask Once, Up Front

Ask the changelog-source question **now** (not at the end), so the user is not blocked after a long review. Use the AskUserQuestion tool when available:

"How would you like to provide the changelog?"
- **Generate automatically from code changes** (Recommended — the full diff is analyzed anyway)
- Provide a URL to fetch from
- Paste changelog text directly

Skip the question and generate automatically if the user already said so, said to skip questions, or the session is non-interactive. Do not ask anything else unless genuinely blocked.

## 1. Branch Comparison & Scope

### 1.1 Resolve base and scope

```bash
git rev-parse --abbrev-ref HEAD                       # current branch
BASE=$(git rev-parse --verify -q master >/dev/null && echo master || echo main)
git merge-base "$BASE" HEAD                           # review from merge-base (three-dot diff)
git log --oneline "$BASE"..HEAD                       # commits in scope
git status --porcelain                                # uncommitted changes — include them and say so
git diff "$BASE"...HEAD --stat=200
git diff "$BASE"...HEAD --name-status -M              # A/M/D/R with renames
```

If the user names another base branch, commit range or PR, use that instead.

### 1.2 Filter noise before reading

Do **not** line-review these; only note them in the summary (and flag if they should not be committed):
- Compiled output: `build/`, `dist/`, `*.min.js`, `*.min.css`, `*.map`, `**/build/**`
- Dependencies: `vendor/`, `node_modules/`, lockfiles (`package-lock.json`, `composer.lock`) — but note version bumps of security-relevant packages
- Generated: `.pot`, `.po`, `.mo`, `.json` translation files, snapshot files
- Third-party bundled libraries (e.g. swiper, select2): check only that version changed intentionally and license kept — **except the Swiper CSS nesting check below, which always runs**

**Swiper CSS nesting check (always run when any Swiper-related CSS is added or changed):** Swiper ≥ 11 ships `swiper-bundle.css` / `.min.css` written with native CSS nesting (`&.swiper-horizontal { … }` inside `.swiper { … }`). Browsers without nesting support (Chrome/Edge < 112, Safari < 16.5, Firefox < 117, older mobile WebViews) drop those rules, which breaks carousel layout. Check every changed browser-served `.css` file that is Swiper's own or styles `.swiper*` classes (skip `.scss`/`.less` sources — they nest legitimately):

```bash
{ git diff "$BASE"...HEAD --name-only --diff-filter=AMR -- '*.css' | grep -i swiper
  git diff "$BASE"...HEAD --name-only --diff-filter=AMR -- '*.css' | xargs grep -l 'swiper' 2>/dev/null; } \
| sort -u | while read -r f; do
  n=$(grep -oE '(^|[{;}])[[:space:]]*&' "$f" | wc -l)   # '&' opening a nested rule
  [ "$n" -gt 0 ] && echo "$f: $n nested rules"
done
```

Any hit ⇒ **WARNING** finding (one finding listing every file with its count and a sample `file:line`). Fix: convert to normal (flattened, non-nested) CSS — run the file through a nesting flattener (e.g. `npx postcss --use postcss-nesting` or `npx lightningcss --targets '>= 0.25%'`) and re-minify, keeping `.css` and `.min.css` in sync; then verify the carousel in a browser without nesting support.

**Flag** as repository hygiene findings: stray files at plugin root (`*.html` prototypes, scratch files, `.env`, `storageState.json`, debug dumps, review reports), compiled assets committed when the project ignores them, and anything that the release-zip script would ship but should not (read the build/zip script's exclude list).

### 1.3 Big-diff strategy

If the reviewable diff is > ~3,000 changed lines or > ~40 files:
1. Triage by risk: PHP entry points first (REST routes, AJAX handlers, `admin_post_*`, shortcodes, render callbacks, file/HTTP/DB access, uninstall, activation/upgrade routines), then JS that talks to the server or renders HTML, then UI/styles.
2. Split into 3–5 independent review areas and, when the Agent tool is available, review them in parallel with subagents. Brief each subagent with: base ref, exact file list, the checklists below relevant to its area, the finding format (Section 4.2), and "report only issues proven by code, with file:line from the HEAD version".
3. Read styles (`.scss/.css`) only for obvious problems (global unscoped selectors that leak into other UI, `!important` wars, RTL breakage) — do not line-review them.

### 1.4 Run available tooling (changed files only)

Run whatever exists; never install tools. Record results, and treat tool output as leads to verify, not findings.

```bash
# PHP syntax on every changed PHP file (use the min PHP version binary if available)
git diff "$BASE"...HEAD --name-only --diff-filter=AMR -- '*.php' | xargs -n1 php -l

# WPCS (respect project phpcs.xml / .phpcs.xml.dist if present)
phpcs -i   # check WordPress standards are installed
phpcs --standard=WordPress --extensions=php --report=source <changed php files>
phpcs --standard=PHPCompatibilityWP --runtime-set testVersion <min-php>- <changed php files>

# JS lint / unit tests if configured in package.json
npx eslint <changed js files>
npm run test:unit --if-present
```

phpcs reports whole files, so pre-existing violations drown the diff. Scope it to changed lines: run with `--report=json`, take `git diff -U0 "$BASE"...HEAD` hunk headers (`@@ -a,b +start,count @@`) to build a set of added line numbers per file, and keep only messages on those lines (small Python script in the scratchpad, run with `python3 -I`).

Summarize phpcs output by sniff (count per sniff), not line-by-line. Report only errors that matter (security sniffs: `WordPress.Security.*`, `WordPress.DB.PreparedSQL*`, `WordPress.WP.AlternativeFunctions`, PHPCompatibility errors) as individual findings.

Also read the project's `CLAUDE.md` / `README` / contributing docs: project conventions override generic advice, and documented architecture is the spec the diff is checked against.

### 1.5 "Tested up to" freshness check (always run)

Compare the plugin's declared tested versions with the **latest released** WordPress and WooCommerce versions, fetched live (never from memory):

```bash
# Declared values
grep -iE '^\s*\*?\s*(Tested up to|Requires at least|Requires PHP|Stable tag):' readme.txt
grep -iE '^\s*\*?\s*(Tested up to|WC requires at least|WC tested up to|Requires at least|Requires PHP|Version):' <main-plugin-file>.php

# Latest WordPress (first offer = latest stable)
curl -s https://api.wordpress.org/core/version-check/1.7/ \
  | python3 -I -c "import json,sys;print(json.load(sys.stdin)['offers'][0]['version'])"

# Latest WooCommerce (only if the plugin declares WC headers or requires woocommerce); -g keeps [] literal
curl -sg 'https://api.wordpress.org/plugins/info/1.2/?action=plugin_information&request[slug]=woocommerce' \
  | python3 -I -c "import json,sys;d=json.load(sys.stdin);print(d['version'],d.get('last_updated'))"
```

Rules:
- **WordPress `Tested up to`** (readme.txt is authoritative; WordPress.org reads only the readme): compare **major.minor** only — WordPress.org ignores the patch part, so `7.1` is current when latest is `7.1.4`. Behind latest major.minor ⇒ **WARNING** (WordPress.org shows "This plugin hasn't been tested with the latest 3 major releases" after enough lag, and the plugin looks unmaintained). If the plugin header also declares `Tested up to`, it must match the readme.
- **`WC tested up to`** (main plugin file header): compare **major.minor** — WooCommerce compares major.minor and shows "not tested with your active WooCommerce version" in the plugins screen and WC status when the active version is newer. Behind latest major.minor ⇒ **WARNING**. Patch-only lag (e.g. 11.2.0 vs 11.2.2) ⇒ OK, mention as Info only.
- Declared version **higher** than the latest release (unreleased/beta) ⇒ **WARNING** — cannot be honestly tested.
- If the API call fails (offline), say so in Limitations and mark the check "not verified"; do not guess the latest version.
- Bumping these values is a release task, not proof of testing: the fix text must say "test against X.Y, then bump", never "just change the number".
- Changelog: add `Tested: WordPress X.Y.Z compatibility.` / `Tested: WooCommerce X.Y.Z compatibility.` only when the header/readme actually declares that version (see Section 6). If the fix is pending, note in the summary that the `Tested:` line must be added together with the bump.

Report the result in the header's **Compatibility Headers** table (Section 4.1) and raise findings for any WARNING.

## 2. Comprehensive Code Review

Review the changed lines **and the code they call or are called by** when that is needed to judge correctness (e.g. a new REST route: read its permission callback and the sanitize path; a changed function: grep its callers).

### A. WordPress Coding Standards (WPCS)

**Prefix Consistency (Critical):**
- All plugin identifiers use a **single consistent prefix** (or namespace)
- Functions: `snake_case` with prefix; classes: namespaced or prefixed; constants: `UPPER_CASE` with prefix
- Hooks, AJAX actions, options, transients, post meta, CPTs, taxonomies, script/style handles, REST namespace: prefixed
- **Flag:** mixing prefixes, hyphens in hook names, unprefixed globals, Pro namespace leaking into free code

**Code Style:**
- Tabs for indentation; spaces around operators; Yoda conditions
- Follow project array syntax convention; trailing commas in multiline arrays
- No trailing `?>` in PHP-only files
- One statement per line (no `$a && do_something();` side-effect chains)

**Documentation & Strings:**
- DocBlocks on public functions/methods/hooks/filters with `@since` (version must match the release version), `@param`, `@return`
- All user-facing strings translatable, correct text domain, no variables/concatenation inside `__()`, `/* translators: */` comments for placeholders, `_n()` for plurals
- JS strings via `@wordpress/i18n` with the same text domain; `wp_set_script_translations()` for new script handles
- No `var_dump()`, `error_log()`, `print_r()`, `console.log()`, `debugger` leftovers; no `@` error silencing

**WordPress APIs:**
- `wp_die()` / `wp_send_json_*()` instead of `die()`/`exit()` in handlers
- No direct `$_GET`/`$_POST`/`$_REQUEST`/`$_SERVER`/`$_COOKIE` without `isset()` + `wp_unslash()` + sanitization
- `$wpdb->prepare()` for every dynamic SQL value; `%i` identifier placeholder only if min WP ≥ 6.2
- Filesystem writes via `WP_Filesystem` where practical
- HTTP via `wp_remote_*`; **`wp_safe_remote_*` for any user-influenced URL**
- Plugin header complete: `Requires at least`, `Requires PHP`, `Text Domain`, `Domain Path`, `Update URI` (premium)
- **`plugins_api()`**: `'query_plugins'` returns arrays, `'plugin_information'` returns objects

### B. Security Review (Critical)

**Map the attack surface added or changed by the diff first**, then trace each source to its sink:
- Sources: `$_GET/$_POST/$_REQUEST/$_COOKIE/$_SERVER/$_FILES`, REST `$request` params, block attributes, shortcode attributes, post meta/options written by lower-privileged users, remote HTTP responses, AI/LLM output, uploaded/browser-read file contents
- Sinks: echo/print/HTML output, `innerHTML`/`dangerouslySetInnerHTML`, SQL, `include/require`, file read/write/delete, `unserialize`, `wp_remote_*`, `update_option`/`update_post_meta`, `wp_insert_post`, redirects, `call_user_func`/dynamic callables, shell functions

**Input & output:**
- Sanitize on input by data type (`sanitize_text_field`, `absint`, `sanitize_key`, `esc_url_raw`, `wp_kses_post`, `rest_sanitize_boolean`…)
- Escape late at output (`esc_html`, `esc_attr`, `esc_url`, `wp_kses`, `wp_json_encode` into `<script>`)
- Block render callbacks / server-side views: every attribute is attacker-controlled by any user who can edit posts (Contributor+) — escape all of it; CSS values built from attributes must be sanitized to prevent `</style>` breakout
- LLM/AI output and remote content are untrusted: escape or `wp_kses_post` before storing or rendering

**AuthN / AuthZ:**
- Every AJAX action: nonce (`check_ajax_referer`) **and** capability check (`current_user_can`) — order nonce → capability → logic. `wp_ajax_nopriv_*` must be intentional and safe for anonymous use (rate limits, no data leak)
- Every REST route: real `permission_callback` (flag `__return_true` unless the endpoint is genuinely public and read-only), `args` with `sanitize_callback`/`validate_callback`; object-level checks (`current_user_can( 'edit_post', $id )`), not only a generic capability
- Every `admin_post_*` / form handler that mutates: nonce + capability
- IDOR: IDs from the request used to read/modify posts, users, options, files

**Vulnerability classes to check explicitly:**
- SQLi, XSS (stored/reflected/DOM), CSRF, PHP Object Injection, path traversal / arbitrary file read-write-delete, LFI/RFI, RCE
- **SSRF:** user-supplied URLs fetched server-side — must use `wp_safe_remote_*`, restrict schemes, cap response size and timeout, never echo raw remote body
- **Secrets:** API keys/tokens/license keys never sent to the browser (`wp_localize_script`, `wp_add_inline_script`, REST responses, `data-*` attributes), never logged, masked in UI, not hard-coded. Check options are not autoloaded unnecessarily and not exposed via `show_in_rest`
- **Information disclosure:** endpoints returning posts/drafts/private content the caller cannot `read_post`; raw provider/remote error bodies echoed to lower-privileged users
- Open redirects (`wp_safe_redirect` with allow-list)
- Privilege escalation via options/role/meta updates from request data (mass assignment: `update_option( $_POST['key'], … )`)
- Unbounded resource use: user-controlled `posts_per_page`, loop counts, AI token limits, file sizes, request fan-out
- Missing `ABSPATH` guard in new PHP files; `eval`, `assert`, `create_function`, `extract()` on request data

### C. JavaScript / Gutenberg / React

- `dangerouslySetInnerHTML`, `innerHTML`, jQuery `.html()` with non-sanitized data; `RawHTML` with user content
- `apiFetch` paths and nonces (`wp.apiFetch` handles REST nonce; raw `fetch` to admin-ajax must send a nonce)
- Secrets or other users' data in globals (`window.*`, localized objects)
- Block `attributes` schema changes: renamed/removed attributes or changed `save()` output cause **block validation errors** on existing content — require `deprecated` entries; dynamic blocks (PHP render) must handle old attribute shapes
- Hooks rules: effects with missing cleanup (listeners, timers, `AbortController` for fetches), state updates after unmount, effects that dispatch `updateBlockAttributes` in a loop (infinite re-render / dirty post)
- `useSelect` returning new object/array references each call (re-render storms); heavy work in render
- Accessibility basics for new UI: buttons are `<button>`, labels on inputs, focus management in modals, keyboard reachability
- New dependencies: bundle size, `@wordpress/*` packages must be externals (not bundled)

### D. PSR-4 Autoloading

- Namespace matches directory path per `composer.json` `autoload.psr-4`; one class per file; filename = class name (case-sensitive — macOS hides case bugs that break on Linux)
- New classes are actually loadable (autoloader or explicit `require`); flag classes that only load because of include order

### E. PSR-12

- `elseif` not `else if`; property/method ordering; visibility on all members
- **If PSR-12 conflicts with WPCS, WPCS wins.** Follow project convention for braces/indentation.

### F. WordPress.org / Commercial Plugin Guidelines

- No site changes without consent; no undisclosed phoning home (telemetry, remote fetches on page load, license pings — must be documented/opt-in where required); no deactivating other plugins; no blocking admin; no unauthorized ads; no hard-coded secrets; GPL-compatible bundled code
- External services (AI providers, CDNs, license servers) disclosed in readme

### G. Backward Compatibility & Upgrade Path

- Check against declared `Requires at least` / `Requires PHP` / `WC requires at least` (flag if headers missing). PHP 7.4: no `match`, enums, readonly, named args, nullsafe `?->`, union types, `str_contains` (without polyfill — WP ≥ 5.9 polyfills `str_contains`/`str_starts_with`/`str_ends_with`)
- WP functions newer than min WP (e.g. `wp_admin_notice` 6.4, Interactivity API 6.5, Block Bindings 6.5, AI Client/Connectors 7.0) must be guarded with `function_exists`/`class_exists`
- **Removed or renamed** public hooks, filters, shortcodes, shortcode attributes, REST routes, AJAX actions, options, meta keys, CSS classes users target, template files theme authors override → BC break; needs deprecation (`do_action_deprecated`, `apply_filters_deprecated`, `_deprecated_function`) or migration
- Stored data shape changes (options, meta, block attributes) need a versioned migration routine (db version option + upgrade on `plugins_loaded`/`admin_init`), idempotent, safe on multisite
- `uninstall.php` changes: removes all new options/transients/tables/cron events; does not delete data the user expects kept
- Activation/deactivation hooks; cron schedules cleared on deactivation

### H. Performance & Architecture

- No queries inside loops; `posts_per_page => -1` without a cap; heavy `meta_query`; `no_found_rows` when pagination unused
- Expensive work (remote HTTP, filesystem scans, `get_posts` over all content, block parsing) never on every page load — cache with transients/object cache, invalidate correctly
- Options storing large arrays: `autoload` off
- Assets enqueued only where needed, correct deps, `in_footer`/`strategy => defer`, version from file mtime or plugin version
- Long-running admin requests: timeouts set, pagination/batching
- Architecture: no god classes/methods (>~300-line methods are a smell), separation of concerns, testable logic; new code consistent with documented project architecture

## 3. Verification Pass (mandatory)

Before writing any finding into the report, re-open the code at HEAD and confirm:
1. The line numbers point at the current file (not diff offsets).
2. The issue is reachable: trace the caller/permission path — e.g. an "unescaped output" whose value is already `absint`ed or whose route requires `manage_options` is downgraded or dropped.
3. The issue is introduced or touched by this diff. Pre-existing issues in untouched code go to a short "Pre-existing (out of scope)" list only if severe.
4. The fix you propose is correct and WPCS-compliant.

Subagent findings get the same verification by the main reviewer — do not copy them into the report unchecked. Drop anything you cannot prove. Mark remaining findings **Verified** (traced end to end) or **Plausible** (strong evidence, one link unconfirmed — say which).

## 4. Report Format

### 4.1 Header

```markdown
# Code Review Report: [BRANCH_NAME] branch

**Branch:** `[current-branch]` @ `[short-sha]`
**Base Branch:** `[base]` (merge-base `[short-sha]`)
**Commits in scope:** N · **Files changed:** N (reviewed: N, skipped as build/vendor/generated: N)
**Uncommitted changes:** none | included (list)
**Review Date:** [current-date]
**Plugin:** [plugin-name] · **Version:** [header Version] (flag mismatch with readme Stable tag / `@since` tags / constants)
**AI Model:** Claude [Model Name] ([model-id])  ← from the system prompt
**Review Method:** Diff review with source-to-sink security tracing, tooling (list what ran), and per-finding verification

---

## Branch Comparison Summary
- What the branch does, grouped by feature (3–8 bullets)
- Added / Modified / Deleted / Renamed counts and the notable files
- Tooling results (php -l, phpcs summary by sniff, eslint, unit tests) — pass/fail counts

## Compatibility Headers
| Header | Declared (file:line) | Latest release (fetched YYYY-MM-DD) | Status |
|--------|----------------------|-------------------------------------|--------|
| Tested up to (WordPress) | 7.1 (`readme.txt:5`) | 7.1.4 | ✅ Current / ⚠️ Outdated (finding ID) |
| WC tested up to | 11.0.0 (`plugin.php:15`) | 11.2.2 | ✅ Current / ⚠️ Outdated (finding ID) |
| WC requires at least / Requires at least / Requires PHP | … | — | listed for reference |
```
Omit WooCommerce rows when the plugin has no WC headers and does not require WooCommerce.

### 4.2 Findings

Order: Critical → Warning → Info; within a level, most impactful first. Number them (C1, W1, I1) so the release checklist can reference them.

````markdown
### [C1] 🔴 CRITICAL · Security · Short Title
**Status:** Verified | Plausible (unconfirmed: …)
**Location:** `path/file.php:LINE` (`Class::method()`) — plus all other locations if grouped

**Relevant Code:**
```php
// minimum snippet showing the problem
```

**Issue:** One clear sentence on what's wrong.
**Exploit / failure scenario:** Who can trigger it (role / anonymous), with what input, and what happens. (No weaponized payloads.)
**Impact:** Why it matters.
**Fix:** Concrete corrected code or precise instruction.
````

**Severity:**
- 🔴 **CRITICAL** — exploitable security issue, data loss, fatal error on supported PHP/WP versions, block validation breakage on existing content, WordPress.org violation, secret exposure
- 🟡 **WARNING** — should fix before release: WPCS security sniffs, missing caps on low-risk paths, BC breaks without deprecation, performance problems on common paths, missing migrations, repo hygiene that ships in the zip
- 🔵 **INFO** — maintainability: naming, docs, refactoring, minor style

Same issue in several files = **one** finding listing all locations.

### 4.3 Summary Table

```markdown
| Severity | Count |
|----------|-------|
| 🔴 Critical | N |
| 🟡 Warning | N |
| 🔵 Info | N |
```

## 5. Release Decision

```markdown
## Release Decision

**Status:** [ RELEASE READY ✅ | RELEASE WITH CONDITIONS ⚠️ | DO NOT RELEASE ❌ ]

**Must fix before release:** - [ ] C1 …
**Should fix before release:** - [ ] W1 …
**Can defer:** - [ ] I1 …

### Rationale
[2–4 sentences: security posture, user impact, upgrade safety, compliance.]
```

**Criteria:**
- **RELEASE READY:** no Critical, Warnings are minor/cosmetic
- **RELEASE WITH CONDITIONS:** no Critical, but Warnings that should be fixed first (list them)
- **DO NOT RELEASE:** any Critical (security, data loss, fatal, broken existing content, guideline violation)

The decision must be definitive — no "it depends".

## 6. Changelog Generation

Use the source chosen in Section 0.
- **Automatic:** map the diff to the five prefixes below.
- **URL:** fetch, then reformat into the format below.
- **Text:** reformat — never pass through unchanged.

Before writing, read the existing changelog (`CHANGELOG.md`, `readme.txt` `== Changelog ==`, `README.md`) so you don't duplicate already-released entries, and use `git log` messages only as hints — the diff is the source of truth.

**Baseline every entry against the base branch — never trust commit subjects alone:**
- **New vs Improved:** before writing `New:`, confirm the feature does not already exist on the base branch (`git show "$BASE":<file>`, `git grep -n "<UI string>" "$BASE" -- src`). Compare the base's UI strings (`__( "…" )` labels) with HEAD's: only what HEAD adds is new; a redesign of something that existed is `Improved:`. Name the delta precisely (e.g. "now supports 49 languages" when the base had 17), not the whole feature.
- **Fixed — released bugs only:** for each fix commit, check that the code it removes or corrects exists on the base branch (removed `-` lines present in `git show "$BASE":<file>`; for pure additions, confirm the broken element/behaviour exists on the base). If the removed lines were added earlier on the same branch (`git log -S'<line>' "$BASE"..<commit>`), it is a regression fix — leave it out. Fixes to a released bug in a feature the branch also redesigned stay in.
- Read multi-part fix commit bodies: one commit may fix several user-visible bugs (each may deserve a line) or include a branch-only regression (skip that part).

**Prefixes (only these, in this order):**
- **New:** new features, options, blocks, templates, integrations
- **Improved:** enhanced behavior, UI/UX changes, options moved or renamed, performance users notice
- **Fixed:** bug and security fixes for **released** behavior, in user terms
- **Updated:** `.pot` file, bundled libraries the user notices
- **Tested:** WordPress / WooCommerce / PHP compatibility, from the plugin header / readme

**Version & date:** version from the main plugin file `Version:` (or readme `Stable tag`); if the code references a higher version (`@since`, constants, docs), use that and flag the header mismatch. Date = today unless given.

**Format (WordPress readme.txt style):**

```markdown
2.8.0 – Aug 08, 2024
New: The Hide On Sale Products option.
Improved: The How To Use Shortcode section moved to the right side of the page.
Fixed: The accordion icon not showing on the mobile view.
Updated: The language (.pot) file.
Tested: WordPress 6.6.1 compatibility.
```

**Rules:**
- Heading `X.Y.Z – Mon DD, YYYY` with an en dash (`–`). Newest on top.
- One change per line, plain text, sentence case, ends with a period. No bullets/bold/nesting.
- Lists of three or more items use the Oxford comma ("Private, pending, and scheduled carousels"). Two items take no comma before "and"/"or" ("The navigation and pagination options").
- Include only what users see or feel. Exclude refactors, renames, tests, tooling, internal hooks, dependency bumps — unless behavior visibly changes.
- **Exclude regression fixes** introduced and fixed within the same unreleased branch. When unsure, leave it out.
- Security fixes: `Fixed:` in user terms, no exploit details.
- Never write a `Tested:` line for a version the header/readme does not declare.
- If nothing is user-visible, say so instead of padding.

## 7. Overall Assessment

```markdown
## Overall Assessment

**Code Quality Score:** [1-10]/10
**WPCS Adherence:** [~]% (basis: phpcs error density or manual estimate — say which)
**PSR-4 Compliance:** Compliant / Partial / Non-compliant
**Security Posture:** Strong / Acceptable / Needs Improvement
**Upgrade Safety:** Safe / Needs migration / Breaking
**Architecture:** Sound / Needs Refactoring / Concerning

**Top 3 Priorities:**
1. …
2. …
3. …

**Release Recommendation:** [one sentence]

---
**Reviewed by:** Claude [Model Name] ([model-id]) — Automated WordPress Plugin Review
**Review Method:** Diff review, source-to-sink security tracing, tooling, per-finding verification
**Limitations:** [what was not reviewed or could not be verified — e.g. compiled assets, runtime behavior, third-party libraries]
```

State limitations honestly instead of claiming "Confidence: High" by default.

## Output Files

Save to the plugin root (do **not** commit them unless asked; warn if the release-zip script would ship them and suggest adding them to its excludes):

1. **`CODE_REVIEW_REPORT_FOR_[BRANCH_NAME].md`** — header, comparison summary, findings, summary table, release decision, overall assessment. Replace `/` in branch names with `-`.
2. **`CHANGELOG.md`** — prepend the new version block above existing entries; if the version block already exists, replace it rather than duplicating.

## Execution Order

1. Ask the changelog-source question (Section 0)
2. Resolve base, scope, filter noise (1.1–1.2)
3. Run tooling (1.4) and the tested-up-to freshness check (1.5); plan parallel review if the diff is big (1.3)
4. Review (Section 2), tracing sources to sinks
5. Verification pass (Section 3)
6. Write report (4–5, 7) and changelog (6)
7. Save both files, then give the user a short summary: decision, counts, top findings with file:line, file paths

## Constraints

- Only review what the diff introduces or touches; read surrounding code only to judge it
- Report only issues supported by actual code — never fabricate or speculate; never cite a line you have not opened
- Never recommend anything that conflicts with WPCS or the project's documented conventions
- Concrete, actionable fixes — no generic advice
- Group duplicate issues; skip non-applicable sections silently
- If code is incomplete/unclear, state the limitation instead of guessing
- Read-only review: do not modify plugin code unless the user asks for fixes
