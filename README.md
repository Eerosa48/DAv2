# Disney App 2.0 (DAv2)

DAv2 is an independent Progressive Web App for Disneyland Resort and the starting point for a modular rebuild of the existing Disney trip companion. This public repository preserves the current application data, map artwork, imagery, and working features while development proceeds in small, verifiable steps.

The current application uses static files and separate UI scripts. The planned modular rebuild must preserve existing behavior while the current script order, shared map state, data relationships, and resource paths are deliberately refactored and checked.

## Relationship to V1

- Development repository: [Eerosa48/DAv2](https://github.com/Eerosa48/DAv2).
- Separate V1 production application: [original published URL](https://eerosa48.github.io/disneyland-trip-2026-app/).
- V1 remains a separate application and deployment. Never modify its repository or local folder, and never fetch its assets or data as a DAv2 dependency.
- DAv2 contains its own application files, data, tiles, icons, and vendored libraries. Live wait times use an independently configured external API.
- Relative resource paths support a later move to the original production path. Such a cutover requires a separate, explicitly approved change and upgrade testing.

Read [AGENTS.md](AGENTS.md) before development. It contains the permanent repository safeguards.

## Current features and entry points

| Entry point | Features |
| --- | --- |
| `index.html` | Illustrated resort map; place search; dated itinerary and details; numbered stop and traced walking-route overlays; device location and recenter controls; live attraction waits; a link to the Disney app; automatic offline map downloading. |
| `route-editor.html` | Local walking-route authoring with draw/refine modes, draggable bends, undo/redo, completed-leg tracking, browser autosave, and compatible JSON import/export. |

The route editor loads the place directory, itinerary, ordered stops, and any saved browser draft. It does **not** automatically load existing traces from `walking-routes-2026.json`; use its import control to load a compatible export. Export downloads completed legs as `walking-routes-2026.json` and does not rewrite repository files or publish anything. Traced routes are visual overlays, not turn-by-turn navigation.

## Technology and dependencies

- HTML, CSS, and browser JavaScript; JSON application data.
- Locally vendored Leaflet 1.9.4 and the Leaflet DoubleTouchDragZoom plugin, including local styles and marker/layer images in `vendor/`.
- Web App Manifest, Service Worker API, Cache Storage, localStorage, and the browser Geolocation API.
- Optional live waits from `https://api.themeparks.wiki/v1/entity/`, configured in `wait-times.js`; fresh wait data requires connectivity. GPS depends on device availability, browser permission, and a secure context.
- Python standard-library utilities for local serving and asset maintenance. No Python packages are required for the local server.
- No framework, bundler, package-install step, backend, or configured automated test runner is currently required. Node.js is optional for JavaScript syntax checks below.

The running map uses local tiles and libraries; it does not need a runtime CDN download of Leaflet.

## Project structure

| Path | Role |
| --- | --- |
| `index.html` | Main map entry point and map initialization. |
| `gps-ui.*`, `itinerary-ui.*`, `places-ui.*`, `toolbar-ui.*`, `wait-times.*` | Feature scripts and styles. |
| `ios-double-tap.js` | Mobile touch interaction helper. |
| `route-editor.html`, `route-editor.js`, `route-editor.css` | Route authoring application. |
| `manifest.webmanifest`, `sw.js`, `offline-ui.js` | PWA identity, worker lifecycle, and offline downloading. |
| `places.json`, `itinerary-2026.json`, `route-2026.json`, `walking-routes-2026.json` | Place, schedule, ordered-stop, and walking-trace data. |
| `tile-index.json`, `tiles/16/` through `tiles/20/` | Curated tile inventory and local map images. |
| `icons/`, `disney-castle-toolbar.png` | Runtime icons and the retained root castle-image copy. |
| `vendor/` | Local third-party JavaScript, CSS, and images. |
| `setup_map.py` | Upstream asset acquisition utility; not an exact baseline reproduction command. |
| `build_tile_index.py` | Generator that rewrites the index of existing tiles. |
| `greys_place_calibration_snippet.js` | Manual browser DevTools calibration helper with browser draft storage and JSON export. |
| `archive/legacy-v1-patches/` | Eight preserved historical patch scripts and their execution warning. |
| `AGENTS.md`, `README.md`, `.gitignore`, `.gitattributes` | Development rules, documentation, ignore rules, and text normalization. |

Required JSON, `tile-index.json`, `tiles/`, `vendor/`, `icons/`, and `archive/` belong in the baseline. The ignore rules cover local caches, environment files, partial downloads, temporary files, and backup artifacts; `.env.example` remains eligible for tracking. The current application does not require an `.env` file.

## Local development on Windows

Open PowerShell in the DAv2 repository root. Verify the location and remote before doing work:

```powershell
git rev-parse --show-toplevel
git remote -v
```

The repository must be DAv2 and its remote must point to `https://github.com/Eerosa48/DAv2.git`. Then serve the existing files using Python 3:

```powershell
py -m http.server 8765 --bind 127.0.0.1
```

Open the main application at `http://localhost:8765/` or `http://localhost:8765/index.html`. Open the editor separately at `http://localhost:8765/route-editor.html`. Stop the server with `Ctrl+C`.

Reserve port **8765** for DAv2 and use the same `localhost` hostname consistently. Hostnames and ports affect browser storage and worker registrations; do not alternate V1 and V2 on the same development origin. Serve the app over HTTP rather than opening it with `file://`.

The Windows localhost server is for testing on that computer. A physical iPhone's localhost refers to the phone itself; device/PWA testing requires an appropriate secure test endpoint, such as a separately approved HTTPS deployment.

## Development PWA identity and storage

| Setting | Current development value |
| --- | --- |
| Manifest `name` | `Greyson's Disney Trip (V2 Development)` |
| Manifest `short_name` and Apple web-app title | `Disney V2 Dev` |
| Manifest `id` | `/dav2-development/` |
| Manifest `start_url` and `scope` | `./` and `./` |
| Shell cache | `dav2-shell-001` |
| Tile cache | `dav2-tiles-v1` |
| Shell cleanup prefix | `dav2-shell-` |
| Itinerary UI state | `dav2-map-session-v1` |
| Route-editor draft | `dav2-route-editor-v1` |
| Calibration draft | `dav2-place-calibration-v1` |

The manifest ID identifies the development app; it is not an asset URL or the service-worker scope. The worker URL and scope derive from the current application directory. Storage and cache identifiers use a DAv2 namespace; the `v1` suffixes here are data/cache schema versions.

When V2 eventually replaces V1 at its original published URL, deliberately restore the original manifest ID `./` and production branding as part of that separate cutover. Preserve the published URL and test existing-install upgrades where supported. Do not apply the production identity during development.

## Offline behavior

`offline-ui.js` registers `sw.js` on HTTPS or localhost when the browser supports service workers and Cache Storage. Installation precaches 28 shell entries, including the main application, its JSON data, icons, and libraries. The worker calls `skipWaiting()` and claims in-scope clients on activation; cleanup removes only other caches with the `dav2-shell-` prefix and leaves the tile cache intact.

For same-origin GET requests within the app directory, tiles are cache-first. Other resources are network-first with a cached fallback when the network request fails. Offline navigation can fall back to `index.html`.

The downloader reads `tile-index.json`, requests only missing tiles, and uses up to six concurrent workers. It pauses when the page is hidden, skips starting downloads while offline, and retries missing tiles on later checks or online events. It honors exposed data-saving/cellular hints, requests persistent storage on a best-effort basis, and reports incomplete downloads or quota failures. Safari does not reliably expose connection type, so automatic downloading is not guaranteed to detect cellular use.

Open the main app while connected and allow tile downloading to complete before relying on offline maps. Available storage, browser eviction, and browser/PWA storage contexts affect retention. Live wait times cannot refresh offline. Do not clear origin-wide storage or unrelated worker registrations to troubleshoot DAv2.

## Curated map inventory and tools

The current `tile-index.json` contains **4,602 tiles**, totaling **174,042,995 bytes** (about **166 MiB**):

| Zoom directory | Tile count |
| --- | ---: |
| `tiles/16/` | 16 |
| `tiles/17/` | 64 |
| `tiles/18/` | 224 |
| `tiles/19/` | 896 |
| `tiles/20/` | 3,402 |

Preserve this curated inventory and its coordinate paths. A generated index is still a required deployed asset and must remain tracked with the tiles it describes.

`setup_map.py` downloads a full rectangular grid from its configured upstream sources. Its current constants request 5,456 tiles through zoom 20, potentially adding 854 tiles outside the curated baseline; its introductory tile-count comments are stale. It also does not acquire the touch plugin. **Do not use it as a routine setup or exact baseline reproduction command.** All files needed to serve the existing application are already local.

`build_tile_index.py` inventories existing tile files and **writes** `tile-index.json`. Use it only after intentional, reviewed tile changes, then verify the resulting inventory. It is not a read-only validation command. The calibration helper exports a review draft rather than automatically applying it to `places.json`.

Historical patch scripts are not runtime dependencies. Read [the archive warning](archive/legacy-v1-patches/README.md); do not execute them without a fresh review of their targets and obsolete assumptions.

## Validation before commits or deployments

Explain significant changes and their impact before implementation. Preserve existing behavior, keep changes small, and perform checks appropriate to the affected feature.

From the repository root, JSON parsing can be checked with PowerShell:

```powershell
Get-ChildItem -Path .\*.json, .\manifest.webmanifest -File | ForEach-Object {
    $null = Get-Content -LiteralPath $_.FullName -Raw | ConvertFrom-Json -ErrorAction Stop
}
```

If Node.js is available, check the external JavaScript files without executing them:

```powershell
Get-ChildItem -Path .\*.js, .\vendor\*.js -File | ForEach-Object {
    node --check $_.FullName
    if ($LASTEXITCODE -ne 0) { throw "JavaScript syntax check failed: $($_.Name)" }
}
```

These commands do not replace browser tests or validate the inline JavaScript in `index.html`. Before approving a change:

1. Open both entry points and check the browser console, resource paths, map pan/zoom, search, itinerary, independent overlays, GPS permission/recenter behavior, live waits, and the Disney link as relevant.
2. Use a separate test browser profile for editor draft/import/export tests; export important drafts before importing or resetting. Verify completed-leg export and reload persistence without overwriting authoritative JSON.
3. Check offline behavior after the shell and tiles have downloaded, including reload, interruption/resume, and incomplete-cache handling. Inspect the DAv2 cache names and worker scope.
4. Keep date keys, ordered-stop `timelineIndex` values, walking segments, and the date assumptions in the UI/editor consistent when changing trip data. Confirm every indexed tile and required local asset exists with matching path casing.
5. Test affected behavior on desktop, iPhone Safari, installed iPhone PWA, tablets, and foldable layouts, including rotation, touch interaction, and safe areas. Report which environments were actually tested.
6. Review changed and untracked files, then inspect Git status and ignored files. Do not rely only on a diff when reviewing new files:

```powershell
git diff --check
git status --short
git ls-files --others --ignored --exclude-standard
```

Report validation results and remaining limitations. Commits, pushes, deployments, significant deletions, and Git configuration changes require explicit approval under `AGENTS.md`.

## Known limitations

- The route editor is not included in shell precaching and does not register a worker itself. A fresh offline editor visit is not supported and may fall back to the main application.
- `manifest.webmanifest` declares `icons/icon-512.png` as 512x512, but the existing image is 1254x1254. This metadata mismatch remains for a focused future correction.
- Default Leaflet marker/layer images are local but not explicitly precached; controls using them may require a prior online visit.
- The current schedule/editor assumes the existing dated trip dataset; it is not yet a general-purpose trip configuration system.
- Shared map globals, script-order dependencies, and distributed UI layering remain to be addressed during the modular rebuild.
- Storage persistence, installation, existing-install upgrades, GPS, and responsive behavior still require browser/device validation. Static syntax, JSON, and asset checks do not establish acceptance on every target device.
