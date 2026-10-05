# AnonKit Release Runbook — Plugins & Themes

Canonical process for shipping any AnonKit-family plugin or theme (anonkit-quorum,
anonkit [Bridge], anonkit-meetingmaker, anonkit-beacon, anonkit-bulletin, fellowship theme).
Distribution is via AnonKit Hub, which serves updates from GitHub releases + this registry.

## How the Hub decides an update exists (read this first)

1. Sites report each registered product's **plugin header `Version:`** to `POST /api/updates/check`.
2. Hub looks the slug up **in `registry.json`** (fetched raw from this repo's `main` branch).
   Not in the registry, or entitlement missing → no update offered, silently.
3. Hub fetches the repo's **GitHub *latest release*** (`/releases/latest`) and compares
   `ltrim(tag_name, 'v')` against the site's installed version with `version_compare`.

So an update is offered only when: registry lists the product **and** a GitHub **release**
(a tag alone is NOT enough) exists whose tag version is greater than the installed header version.

## Version sync checklist — every release, no exceptions

The #1 historical failure is these drifting out of sync. All of them must carry the same number:

1. Plugin header `Version:` in the main plugin file
2. The version constant next to it (`AKQ_VERSION`, `ANONKIT_VERSION`, `MMTSML_VERSION`, …)
3. `readme.txt`: `Stable tag:` + a new changelog entry `= X.Y.Z (YYYY-MM-DD) =`
4. Git tag `vX.Y.Z` — **lowercase v**, annotated
5. GitHub **release** created from that tag (`gh release create vX.Y.Z`)
6. `registry.json` → the product's `plugin.version` field (`"vX.Y.Z"`), committed and
   pushed to `main`. No registry tag needed — Hub reads raw `main`.

Themes: same idea; version lives in `style.css` header, registry field is under `theme`.

## Release sequence

```bash
# 0. Hygiene — in EVERY repo you'll touch (plugin repo AND registry):
git fetch && git status -sb        # pull --rebase if behind

# 1. Land the work as granular commits (feat:/fix:), then one bump commit:
#    header + constant + readme stable tag + changelog
git commit -m "chore: bump to vX.Y.Z"

# 2. Tag, push, release:
git tag -a vX.Y.Z -m "Version X.Y.Z"
git push origin main && git push origin vX.Y.Z
gh release create vX.Y.Z --title "vX.Y.Z" --notes "…"

# 3. Registry (this repo):
#    bump the product's version field, then:
git commit -m "<slug>: vX.Y.Z (one-line summary)"
git push origin main
```

## Timing & caches (why "it's not showing up yet")

- Hub caches the GitHub release lookup **5 minutes** and the registry on its own TTL —
  the Hub admin page (`admin.php?page=anonkit`) reflects a release within minutes.
- The native Plugins/Updates screens read WordPress's `update_plugins` **site transient
  (~12 h)**. Force a refresh: `wp-admin/update-core.php?force-check=1`.
- The Hub admin page queries live; plugins.php shows the cached transient. Seeing the
  update on one but not the other is staleness, not a broken release.

## Known gotchas

- **Bridge < 1.0.26 download expiry:** Hub download URLs are signed with a 5-minute TTL,
  but old bridges stored them in the 12-hour transient — manual updates outside the
  window failed with "Download failed. Unauthorized." Bridge ≥ 1.0.26 re-signs at install
  time (`upgrader_pre_download`). On a site running an older bridge, force-check and
  update within 5 minutes (this applies to updating the bridge itself).
- **Bridge 1.0.26–1.0.27 first installs:** those versions re-signed from the update check,
  which only covers installed products, so installing a *new* product from the Feature
  Manager always failed ("Could not obtain a fresh download link"). Fixed in 1.0.28; on an
  affected site, update the bridge first (updates were unaffected).
- **Repo name ≠ slug is fine:** e.g. slug `anonkit-quorum` lives in repo `quorum-connect`.
  The bridge's `fix_directory_name` renames the GitHub zipball directory to the slug.
- **Quorum readme legacy numbering:** pre-2026 changelog/upgrade-notice entries use a
  legacy scheme that never matched git tags. Leave them alone; new entries use
  `= X.Y.Z (YYYY-MM-DD) =` at the TOP of the changelog and may collide with legacy
  numbers below — expected, documented by the note in the file.
- **Update registration:** each plugin must call
  `anonkit_register_for_updates( '<slug>', __FILE__ )` on the `anonkit_loaded` hook,
  where `<slug>` matches the registry slug. Without it, the Hub page still shows updates
  but WordPress's native updater never will.
- **`Requires at least` / `role: content` style compat keys** in block.json etc. are
  ignored harmlessly by older WP — no need to gate releases on minimum-version bumps
  for additive metadata.

## Verifying a release

1. Hub admin page on any site shows `old → new` on the product card (within ~5 min).
2. `update-core.php?force-check=1` → product appears under Plugins with the new version.
3. Run the update; confirm "updated successfully" and the new version on plugins.php.
