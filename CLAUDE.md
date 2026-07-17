# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

UrBizia — PlanSan Mobile is a single-file Progressive Web App (PWA) for surveying public toilets in the field ("constat terrain des toilettes publiques"). It's the mobile companion to a desktop app called "PlanSan" — mobile confirms/edits/reports toilets on a map; desktop is the authoritative source for the base dataset. There is no build step, no package.json, and no test suite: the entire app is `index.html`, served as a static file.

## Files

- `index.html` — the whole application: CSS, embedded seed data, and all JS in inline `<style>`/`<script>` tags. ~1100 lines but one line (~2.9MB) is `MAP_DATA_TOI`, a giant embedded array literal of toilet records, and another (~35KB) is a base64-encoded logo (`LOGO_B64`). Don't try to eyeball those lines in an editor — use `grep`/`sed`/targeted `Read` offsets instead of opening the whole file.
- `sw.js` — service worker: cache-first for same-origin app-shell files, network-first (with cache fallback) for everything else (map tiles, CDN assets).
- `manifest.json` — PWA manifest (name, icons, theme colors, `display: standalone`).
- `icon-*.png` — app icons.

## Running / testing locally

There's no dev server or build tooling. Serve the directory with any static file server and open `index.html`:

```bash
npx serve .
# or
python -m http.server 8000
```

Opening `index.html` directly via `file://` mostly works but the service worker registration will silently fail (this is handled gracefully — see the `.catch()` around `navigator.serviceWorker.register`). Geolocation and camera capture require a real device or browser devtools device emulation to test meaningfully.

There is no lint/test/build command — verify changes by loading the page in a browser (see the `run` skill) and exercising the flow you changed.

## Architecture

Everything lives in one global script scope in `index.html`. Key pieces, in the order they appear in the file:

**Data model** (three sources of toilet points, merged at render time):
- `MAP_DATA_TOI` — the base dataset seeded from PlanSan desktop. Each row is a positional array: `[nom, adresse, ville, dep, region, operateur, verif, auto, pmr, lat, lon, refId]`. `verif` is one of `'v'` (Vérifiée/Gouv+OSM), `'g'` (Gouv only), `'o'` (OSM only) — see `V_COLORS`/`V_LABELS`.
- `ANNOTATIONS` — a `{ [refId]: {...} }` map of local edits/overrides layered on top of a `MAP_DATA_TOI` row (position override, type, rating, photos, deletion state, etc). Never mutates the base row directly.
- `NEW_POINTS` — an array of user-reported toilets that don't exist in the base dataset at all (`id` starts with `NEW-`).

`findToiByRefId`, `findNewPointById`, `getAnnotation`, `isDeleted` are the accessors used everywhere instead of reaching into the arrays directly.

**Map rendering** (`buildLayers`): Leaflet + `leaflet.markercluster`, loaded from CDN (`cdnjs`). Five cluster groups keyed `v/g/o/supprimees/nouvelles` toggled by the chip filters (`activeChips`). `buildLayers()` is the single re-render entry point — call it after any mutation to `MAP_DATA_TOI`, `ANNOTATIONS`, or `NEW_POINTS` rather than patching the DOM/markers directly. Map tiles switch dark/light CARTO basemaps automatically via `prefers-color-scheme`.

**Theming**: CSS custom properties in `:root` with a `@media (prefers-color-scheme: light)` override block — the whole UI (not just the map) follows the OS theme, no manual toggle.

**Sheet UI** (`openSheet` / `closeSheet` / `openForm` / `openRatingForm` / `saveForm`): a bottom-sheet panel is populated by string-concatenated `innerHTML` (no templating engine, no framework) and wired up with `addEventListener` calls made right after the HTML is injected. This pattern repeats throughout the file — when editing sheet/form markup, keep the corresponding `document.getElementById(...).addEventListener` calls in sync since they're re-attached every time the sheet content is rebuilt.

**Rating/equipment model**: `openRatingForm` builds sliders (`sliderRow`, -5..+5 range inputs) for access/cleanliness/odors/overall, plus per-equipment OK/HS toggles with a fill-level (`equipRow`: lave-mains, savon, séchage, papier, poubelle). Stored under `ann.rating` / `ann.equipements` and summarized read-only via `ratingSummaryHtml`.

**Photos**: captured via `<input type="file" capture="environment">`, compressed client-side to a JPEG data URI (`compressImage`, canvas-based, no libraries), tagged with a category (`environnement`/`portrait`/`interieur`). Stored embedded (`data:` URI) until synced, at which point `uploadPhotoIfNeeded` uploads the blob to Supabase Storage and swaps the field for a public URL (`hosted: true`).

**Sync (Supabase)**: `SUPABASE_URL`/`SUPABASE_ANON_KEY` (anon/public key, safe to be client-side) point at a Supabase project used as the shared backend.
- `dirtyRefIds` / `dirtyNewIds` (in-memory `Set`s) track local unsynced changes; `markDirty`/`markNewDirty` add to them.
- `pushDirty()` uploads photos then upserts rows into `toilette_annotations` / `new_points` tables (`Prefer: resolution=merge-duplicates`).
- `pullAll()` fetches all rows from those tables, skipping any ref that's still locally dirty (so an unsent local edit is never clobbered by a stale server read).
- `syncNow(silent)` runs push then pull and re-renders; called once silently on startup and again from the menu's "Synchroniser maintenant" button.
- `refreshBaseFromServer()` is a separate, one-way operation: paginates through a `toilettes_base` table published by PlanSan desktop and replaces `MAP_DATA_TOI` wholesale — this is how the mobile app picks up new base data without an app update.
- There is no auth/login — sync is anonymous and shared across all installs of the app pointing at the same Supabase project.

**Local export/import** (`exportAnnotations`/`importAnnotations`): a JSON file (`{ type, version, annotations, newPoints }`) downloaded/uploaded as an offline fallback and as the interchange format with PlanSan desktop. Import skips annotations whose `refId` isn't found in the current `MAP_DATA_TOI` and marks everything else dirty so it also gets pushed to Supabase on next sync.

**Startup flow**: welcome overlay → `locateOnStartup()` requests geolocation and zooms to a 500m bbox around the user (falls back to the default France-wide view if denied/unavailable) → `buildLayers()` → silent `syncNow(true)`.

## Conventions specific to this codebase

- French throughout: UI strings, variable-adjacent domain terms (`ann`, `verif`, `refId`, `etat`, `supprimee`), and code comments. Keep new user-facing strings and comments in French to match.
- No framework, no bundler, no modules — everything is a global `const`/`function` in one script tag, and DOM updates are done via `innerHTML` string building rather than a virtual DOM. Match this style rather than introducing a build step or component library.
- `CHANGELOG` at the top of the script is a manually maintained, user-visible version history (shown in the app's "about" panel) — bump `APP_VERSION`/`SYNCED_WITH` and add an entry here for any user-facing change.
- Everything degrades gracefully offline/without permissions (geolocation denied, service worker unregisterable, sync unreachable) rather than erroring — preserve that pattern (silent `catch`, toast messages, fallback to last-known-good state) when touching these paths.
