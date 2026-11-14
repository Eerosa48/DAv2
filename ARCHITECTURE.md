# DAv2 architecture: current baseline and proposed modular design

This is a design document, not an implementation or deployment approval. **CURRENT** describes behavior confirmed by static inspection of the committed application. **PROPOSED** describes future interfaces, responsibilities, and migration work; none of those modules has been implemented by this document.

The governing constraints are [AGENTS.md](AGENTS.md) and [README.md](README.md): preserve all existing features, application data, artwork, browser drafts, and the established desktop/iPhone experience. DAv2 must remain independent of V1. Do not read or write V1 files, fetch V1 resources, change production identity, or deploy as part of this architecture work.

Inspection covers `index.html`, every first-party root JavaScript/CSS file, the editor, worker, manifest, and JSON relationships. Leaflet CSS is referenced only where its pane/control rules explain application stacking. Binary tiles were not individually enumerated. Static inspection establishes source behavior and dependencies, not computed browser geometry, gesture quality, accessibility acceptance, or device compatibility.

## 1. CURRENT ARCHITECTURE

### CURRENT: entry points and startup

The application is a static HTML/CSS/JavaScript PWA with no configured framework, bundler, backend, or automated test runner. Feature scripts are mostly immediately invoked functions with private closure state. They are separate files, but their startup and integration are not independent modules.

`index.html` loads styles in this order: Leaflet, DoubleTouchDragZoom, `places-ui.css`, `wait-times.css`, `gps-ui.css`, `itinerary-ui.css`, then `toolbar-ui.css`, followed by its inline styles. Later toolbar rules intentionally override earlier feature styles; stylesheet order is part of the current contract.

Classic scripts execute in the following source order, without `async`, `defer`, or `type="module"`:

| Order | Source / relevant symbols | Confirmed responsibility and dependency |
| --- | --- | --- |
| 1 | `vendor/leaflet.js`; `vendor/leaflet-double-touch-drag-zoom.js` | Establish `window.L` and register the touch-drag zoom handler before map creation. |
| 2 | `index.html`: `coverage`, `xyzToLatLng`, `tileBounds`, `artworkBounds`, `streetBounds`, `cameraViews`, `desktopCameraViews`, `map` | Create the main Leaflet map, local tile layer, camera presets, bounds policy, and resize/zoom handlers. |
| 3 | `ios-double-tap.js`: `onStart`, `onMove`, `onEnd`, `reset` | Attach the iOS touch-only two-tap fallback to the existing map. |
| 4 | `places-ui.js`: `buildControl`, `searchPlaces`, `showPlace` | Construct search controls synchronously; asynchronously load and index `places.json`. |
| 5 | `wait-times.js`: `makeControl`, `refresh`, `paint`, `window.greysonWaitFeed` | Construct wait UI and expose the shared feed before asynchronously loading places and live park feeds. |
| 6 | `gps-ui.js`: `GPSControl`, `locateOrRecenter`, `startWatch`, `onPosition` | Add location/recenter controls and GPS pane; do not request tracking until the user acts. |
| 7 | `itinerary-ui.js`: `loadItinerary`, `selectDay`, `render`, `renderRoute`, `renderWalking` | Construct itinerary/day-banner DOM synchronously, start data loading, and own schedule/session/overlay state. |
| 8 | `offline-ui.js`: `start`, `ensureManifest`, `syncMissingTiles` | Create the offline status host, register the worker, and begin eligible missing-tile checks independently of itinerary completion. |
| 9 | `toolbar-ui.js`: `toggleTool`, `closeAll`, `updateActive`, `positionConnector` | Adapt already-created search/itinerary DOM, create waits panel and toolbar, and coordinate presentation. |

Script execution order does not imply data-request completion order. The toolbar requires synchronous control construction, not completion of the JSON/API fetches. Search, waits, and itinerary currently load the place directory separately. Failed data requests have feature-specific fallback/retry behavior.

`route-editor.html` is a separate entry point: Leaflet CSS, `route-editor.css`, Leaflet JS, then `route-editor.js`. Its closure creates its own local `map`, editing layers, event handlers, and draft/history state. Startup loads places, ordered stops, and itinerary data, joins them, then calls `restoreDraft()`, `redraw()`, and `fitLeg()`. It does not automatically load existing walking traces; compatible JSON must be imported through the editor UI.

`greys_place_calibration_snippet.js` is an optional manual DevTools helper, not part of either normal startup. `setup_map.py`, `build_tile_index.py`, and archived patch scripts are not browser dependencies.

### CURRENT: map initialization and shared globals

The main `const map` in `index.html` is a **global lexical binding shared by subsequent classic scripts**; the source does not assign the Leaflet instance to `window.map`. Moving it into a module would remove that implicit binding and cause existing `typeof map` guards to exit. Shared lexical values also include `artworkBounds` and `streetBounds`; global helper functions include `focusInitialView`, `focusResort`, `focusPark`, and `configureNavigationBounds`.

- `index.html`: `L.map` starts with `minZoom:14`, `maxZoom:20`, `zoomSnap:0.05`, `zoomDelta:0.5`; `updateMinZoom()` then sets the effective minimum to 15. Zoom controls are disabled. The touch-drag plugin is enabled with its existing inversion/scale options.
- `L.tileLayer('tiles/{z}/{x}/{y}.jpg', ...)` uses tile size 256, native zooms 16 through 20, no wrapping, existing coverage bounds, `keepBuffer:3`, and `updateWhenIdle:true`.
- `focusInitialView()` and `focusPark()` choose different mobile/desktop camera presets at the 900px breakpoint. Preserve the exact existing presets rather than deriving new ones during extraction.
- `configureNavigationBounds()` expands `streetBounds` by half the current viewport in projected coordinates, allowing boundary positions to be centered. It runs after initial setup, on `zoomend`, and during the animation-frame-coalesced resize handler. Resize calls `invalidateSize({pan:true,animate:false})`; it does not call initial-view focusing.
- `gps-ui.js` uses `artworkBounds` and calls `focusInitialView(true)` from its separate resort recenter control. `itinerary-ui.js:restoreMapView` uses `streetBounds`; `refreshDayIfNeeded` calls `focusInitialView()` at day rollover.
- Editor configuration is deliberately different: `route-editor.js` uses `minZoom:15`, enabled zoom controls, disabled double-click zoom, a padded artwork max-bound rectangle, a distinct initial view, and its own drawing interactions. Sharing a factory must not make the two map profiles identical.

The CSS/HTML for older Focus controls and `parkViews`/camera helpers remain, although the former Locations control is not constructed by the current main startup. Do not remove this scaffolding as an incidental part of architectural extraction.

### CURRENT: feature ownership and integration seams

| Owner | Private state / behavior | Current coupling |
| --- | --- | --- |
| `places-ui.js` | Place list/index; search text and park/type filters; ranked results; search highlight. `normalize`, `phraseScore`, `tokenScore`, and `editDistance` implement existing search semantics. | Reads `L`/`map`; owns `.place-search-trigger` and `.place-search-panel`; document pointer handling closes the original control. |
| `itinerary-ui.js` | Loaded trip/route/place data; `dayIndex`, `currentTab`, `routeOn`, `walkingOn`; separate stop/walking layer groups; itinerary focus highlight; day menu; session persistence. | Reads `map`, `streetBounds`, and `focusInitialView`; toolbar drives its original launch control. |
| `wait-times.js` | `latest`, `waitItems`, `lastCheck`, `pending`, `errors`; API matching, badges, map markers, visibility and refresh timer. | Exposes `window.greysonWaitFeed`; toolbar subscribes to the same feed. `window.greysonWaitDebug` is diagnostics, not a stable feature API. |
| `gps-ui.js` | Tracking intent, watch ID, latest position, center-on-next-fix flag, bounds notices, graphics, GPS toast. | Reads the shared map/bounds; owns `greyson-gps-pane` and a Leaflet control. |
| `toolbar-ui.js` | Active presentation inferred from `.hidden`; toolbar buttons; waits panel; independent wait-selection highlight; connector geometry. | Selects feature DOM by CSS class; calls hidden triggers with `.click()`; reparents search panel into `document.body`. |
| `offline-ui.js` / `sw.js` | Page-side download task, abort/resume/quota/status state; worker-side shell/tile caching. | Share `TILE_CACHE` and the bulk-request header contract; otherwise separate execution contexts. |
| `route-editor.js` | Separate selected day/leg, edit mode, `via` points, completion flags, undo/redo stacks, browser draft, import/export. | Reads static data and uses the editor's own map. Exports a download; does not rewrite repository data. |
| Calibration helper | Manual place-coordinate edits and exported review draft. | Reads the main lexical map and local places; exposes only `window.greysonCalibration.close` for replacing/closing the helper. |

`toolbar-ui.js` requires `.place-search-trigger`, `.place-search-panel`, `.greyson-itinerary-launch`, and `.greyson-itinerary-panel`. It moves search out of the Leaflet control context while retaining its listeners. `closeAll()` clicks original triggers, and `activeTool()` resolves waits, then itinerary, then search if more than one panel happens to be visible. Toolbar state is resynchronized through document click/pointer handlers and animation frames, not a shared state store.

Itinerary and Wait Times declare `role="dialog"`; Search is a plain `div` with a results `role="region"`. The primary panels are not implemented as blocking, focus-trapped modals. Close buttons/Escape, focus restoration, outside-pointer dismissal, propagation suppression, and capture-phase highlight clearing differ by feature. Search, itinerary, and wait selections have separate highlights; a refactor must not silently replace them with one mutually exclusive selection.

### CURRENT: state persistence, lifecycle, and external services

| Store / service | Confirmed current contract |
| --- | --- |
| `dav2-map-session-v1` | `itinerary-ui.js:saveSession/readSavedSession`: `{savedOn,selectedDate,center:[lat,lon],zoom,stopsOn,walkingOn}`. Restore only for the same Anaheim day and supported date; actual supported trip day overrides a saved different day. Validate view against street bounds and zoom limits. |
| `dav2-route-editor-v1` | `route-editor.js:saveDraft/restoreDraft`: `{version:1,selectedDay,selectedLeg,legs:{[date]:[{via,done}]}}`. Draft validation checks geometry and leg counts; selected leg is clamped. Undo/redo history is in memory. |
| `dav2-place-calibration-v1` | Manual helper's stable, separate localStorage draft key. Export does not automatically apply edits to `places.json`. |
| Cache Storage | `sw.js`: `dav2-shell-001`, `dav2-tiles-v1`, cleanup prefix `dav2-shell-`; downloader agrees on the tile name. |
| ThemeParks.wiki | `wait-times.js:API/PARK_IDS/refresh`: two park `/live` requests below `https://api.themeparks.wiki/v1/entity/`, without an API credential. `Promise.allSettled` permits partial results. |
| Disney app link | `toolbar-ui.js:disneyLaunch` navigates to `https://disneyland.app.link/DisneylandApp`. It is not another panel or a V1 dependency. Actual native-app/fallback handling belongs to the device and external provider. |

The main app does not use sessionStorage, IndexedDB, a server-side state service, or a general event bus. localStorage failures are caught. GPS watch/fix state is not explicitly serialized, but GPS-driven map movement can feed `saveSession()` through map `moveend`/`zoomend`; do not claim that no location-related map coordinates are persisted.

`todayInAnaheim()` and `anaheimMinutes()` use `America/Los_Angeles`, independently of device timezone and JSON timezone metadata. `schedulePeriods()` interprets the existing display strings for current/upcoming highlighting without rewriting their text. `refreshDayIfNeeded()` resets overlays, tab, itinerary highlight, and map overview at Anaheim day rollover; outside supported dates it uses preview `dayIndex=1`. The six JSON days are not all selectable UI days.

The wait feed exposes `subscribe(callback)` with an immediate snapshot and unsubscribe return value, plus `refreshIfStale()`. Snapshot shape is `{items,lastCheck,pending,errors}`; items expose `{place,badge:{label,style}}`. Name/alias matching uses the feed's own normalization, which differs from place search. Map bubbles appear only online at zoom 18 or above. Timer/visibility refresh uses a one-minute threshold; going offline clears items/markers. A failed park is not kept as an automatically retained stale result on the next refresh.

GPS releases its watch/graphics on hiding or page exit, resumes only for existing tracking intent, rejects stale fixes, and avoids focusing outside artwork bounds. Its recenter preserves a manually closer zoom. The iOS fallback disables Leaflet double-click zoom only on detected iOS touch devices and preserves desktop double-click, pinch, panning, and double-touch-drag behavior. These are source-confirmed policies; physical-device results remain a regression gate.

## 2. TARGET ARCHITECTURE

### PROPOSED: composition and module boundaries

Use small browser-JavaScript modules with explicit dependencies. Native ES modules are the preferred eventual format, subject to the migration/installed-PWA decision in section 9. A framework, bundler, dependency-injection container, reactive state library, or generic message broker is not justified by this baseline. Existing vendored Leaflet remains local; a small integration boundary accepts `L` explicitly.

The composition root creates services once, mounts views, connects controllers, starts independent loading tasks, and owns teardown. Feature modules do not search for another feature's DOM or register uncontrolled window globals. Domain calculations should remain usable without a DOM or Leaflet instance. Leaflet-specific renderers may depend on Leaflet through an injected map/rendering capability; avoid building an oversized wrapper around the entire Leaflet API.

### PROPOSED: transitional module strategy

The safest first extraction keeps the parser-ordered classic scripts, synchronous control construction, and `index.html`'s lexical `const map` active. Do not add `type="module"`, `defer`, dynamic imports, or top-level asynchronous waits to existing entry scripts as part of extracting a utility.

1. Extract one side-effect-free utility into a synchronous classic-script IIFE. It registers a small named API on one temporary, application-owned `window.DAv2Compat` facade. This is a bounded compatibility exception, not a general global state store: register each API once, keep its public surface explicit, and do not put map instances, feature state, DOM nodes or mutable datasets on it. Registration performs no requests, storage writes, timers or UI mounting.
2. Load the utility before its first consumer, retaining all existing feature-script order. Replace one original call site at a time and compare outputs against the original function. Preserve separate search/wait normalization policies. No framework or bundler is needed.
3. Extract components as explicit factories, without automatic mounting on file load. The existing classic entry point invokes a factory at its original initialization point and passes `{map,L,...dependencies}` from its current scope. Keep the original lexical map binding; do not replace it with `window.map` or assume that moving code into an IIFE/module preserves access. Only one implementation owns each listener, watcher, timer, control and layer.
4. Keep compatibility adapters narrow and reversible. Record their consumers; remove an adapter only after its last caller is migrated. Module-format conversion, path movement, and behavior changes remain separate reviewed slices.

Before native ES module adoption, approve a small, isolated proof-of-concept using one pure utility and one explicitly mounted component. Use a separately authorized harness, not an unreviewed startup change in either live entry point. Demonstrate deterministic initialization relative to Leaflet and the classic map binding, single mounting/disposal, application-root URL resolution from nested modules, correct JavaScript MIME delivery, and complete offline caching of transitive imports. Compare cold/warm startup and an existing installed-PWA update, including actual iPhone Safari/Home Screen checks when device access exists. Module loading is deferred/asynchronous relative to parser execution; successful desktop parsing alone does not prove startup parity. Only a passed, reviewed proof permits a later entry-point/module-format switch; retain the classic path if the proof is incomplete.

### PROPOSED: module responsibilities

All names/signatures below are **PROPOSED interfaces**, not files or exports that already exist:

| Proposed module | Responsibility | Proposed public contract / boundary |
| --- | --- | --- |
| Application bootstrap | Separate main/editor composition, startup ordering, ready/error states and teardown. | `bootstrapMain(dependencies)`, `bootstrapEditor(dependencies)` return `{dispose}`. Create map/UI before awaiting optional network data. |
| Map engine and profiles | Create exactly one map per entry point; preserve distinct camera/bounds/gesture configurations; own map lifecycle. | `createMap(host,profile,{L,layers})`; expose view operations, view subscription, and scoped renderer access. Only this boundary creates/destroys the map. |
| Resource paths | Resolve deployment-relative assets from an explicit app root. | `createPaths(appRoot).asset(relativePath)`. Features receive this service instead of deriving paths from their future module location. |
| Data repositories/contracts | Load/index immutable planning data; expose per-resource readiness, retry and cross-reference diagnostics. | `getPlaces()`, `getItinerary()`, `getOrderedStops(date)`, `getWalking(date)`, plus validation diagnostics. Preserve consumer-specific failure policies during adoption. |
| Shared state | Small observable snapshots and narrowly scoped writer capabilities for domain owners. | `getSnapshot()`, `subscribe(selector,listener)` returning unsubscribe. User actions go through owning controllers; no arbitrary global patch API for features. |
| Park clock and persistence | Injectable Anaheim calendar/highlight helpers, elapsed freshness timing, device-local formatting, and separate draft/session adapters. | `getParkDate()`, `getParkMinutes()`, `now()`, `formatLocalTime()`; `session.load/save`, `editorDraft.load/save`. Keep current keys and validation. |
| UI shell and viewport | Stable hosts, safe-area/layout measurements and lifecycle coordination. | `mountShell(host)`, `getHost(name)`, `subscribeLayout(listener)`, `dispose()`. Reuse existing positions/parents first. |
| Toolbar and panel controller | Own active primary panel, toggle/close commands, focus restoration and connector presentation. | `registerPanel(id,adapter)`, `open(id)`, `close(id)`, `toggle(id)`, `getActivePanel()`. External Disney navigation stays a link. |
| Places/search | Own search query/filters, ranking, result view and search-selection channel. | `search(query,filters)`, `setQuery`, `setFilters`, `selectPlace(id)`, `open/close`. Extract current ranking unchanged. |
| Itinerary | Own selected day/tab, schedule rendering/highlighting, supported-day policy and session orchestration. | `selectDay(date)`, `selectTab(tab)`, `getDay()`, `open/close`; issue overlay commands through injected services. |
| Numbered stops overlay | Join stop records to places/timeline; render destination pins/popups. | `setDay(date)`, `setVisible(boolean)`, `focusBounds()`, `dispose()`. Never implicitly toggle walking lines. |
| Walking overlay | Load/render validated traced geometry and review/error/loading state. | `setDay(date)`, `setVisible(boolean)`, `retry()`, `dispose()`. Geometry remains a visual overlay, not navigation. |
| Selection rendering | Keep feature-specific highlights and explicit dismissal policy. | `show(source,placeId)`, `clear(source)`, `clearForPointer(event)`. Preserve independent search/itinerary/waits channels. |
| GPS | Own tracking intent, watch lifecycle, fix freshness/bounds checks, graphics and status. | `locateOrRecenter()`, `suspend/resume()`, `subscribe(listener)`, `dispose()`. Inject map/profile and notification channel; keep raw watch state out of shared preferences. |
| Live waits | Separate API transport/matching/status from map bubbles and toolbar list rendering. | Retain `subscribe()` and `refreshIfStale()` semantics; add explicit `start/dispose` lifecycle. Both views use one feed, never separate polling loops. |
| Notifications | Central host/layer policy with independent GPS/offline channels and accessible announcements. | `notify({channel,message,dismissible,duration})`, `dismiss(channel)`. Preserve current simultaneous-channel behavior and timers first. |
| Offline/PWA client | Register the root worker, manage tile inventory/download lifecycle and publish status. | `start()`, `checkMissing()`, `suspend()`, `subscribeStatus()`, `dispose()`. Worker lifecycle/cache strategy remains separately owned in `sw.js`. |
| Route editing | Separate editor model/history, geometry, persistence, import/export, view and map renderer. | `selectDay/Leg`, `setMode`, `undo/redo`, `finishLeg`, `importDraft`, `exportCompleted`, `dispose`. Share data/math, not the main app's map or session. |

Dependencies flow from composition into services/controllers/views. Repositories do not import views; the map engine does not import itinerary or GPS; the panel controller does not own trip data; the offline worker cannot import DOM-dependent application modules. Start data tasks independently so a failed wait API or itinerary request does not prevent the map, GPS controls, or offline registration from initializing.

### PROPOSED: centralized visual design tokens

Introduce a future `styles/tokens.css` for shared **visual design values**, separate from the context-qualified layer registry in section 3. Use semantic `--dav2-*` custom properties, retaining the current resolved values, units, fallbacks and responsive variants during initial extraction. This is consolidation, not a redesign or a new theme. Similar-looking literals need not be merged if their roles or current computed values differ.

| Token family | Proposed ownership and extraction evidence |
| --- | --- |
| Colors | Semantic map background, surfaces/glass gradients, text, borders, focus, selection, route and wait-status colors. Inventory `index.html`, feature styles, and Leaflet renderer options; preserve status meaning and opacity. |
| Typography | Existing system font stacks, sizes, weights and line heights for body, controls, panels, labels and notices. Preserve search input sizing and text wrapping; do not introduce fonts or normalize deliberately different roles. |
| Spacing | Existing control padding, panel gaps, offsets and connector clearances. Derive names from actual uses in `toolbar-ui.css`, `itinerary-ui.css`, `places-ui.css`, `gps-ui.css` and `route-editor.css`, rather than imposing a new spacing scale. |
| Dimensions | Control/touch-target sizes, panel widths, icon sizes, map-label dimensions and editor sidebar sizes. Retain current `min`, `max`, `calc`, percentage and viewport expressions. |
| Border radii | Surface, control, badge and connector shapes, retaining their distinct current values. |
| Shadows and surface effects | Existing shadow lists, glass fills, blur/saturation and reduced-transparency alternatives; preserve effects that influence stacking/compositing. |
| Motion | Existing transition durations/easing, map animation options and notice durations, with their separate meanings and reduced-motion rules. A shared token does not authorize changing a gesture threshold or timing policy. |
| Responsive/safe-area measurements | Existing safe-area aliases, panel/toolbar offsets, measured banner gap and viewport formulas. Keep dynamic measurements such as `--greyson-expanded-panel-top` owned by the viewport service, rather than replacing them with a fixed token. |

CSS is the canonical source for visual values. When JavaScript renderers need a shared visual value, resolve a documented CSS custom property after styles are available and pass the resolved value to the renderer; do not copy its numeric/color literal into a competing JavaScript palette. Avoid repeated computed-style reads inside render/gesture loops. Keep current legacy properties as temporary aliases where needed, without changing stylesheet precedence. Media-query breakpoints remain explicit existing conditions: custom properties cannot simply be substituted into ordinary media-query conditions. Preserve the present breakpoints and document their relationship to tokens; a future generated breakpoint source is an optional decision, not a bundler requirement.

Do not add z-index values to `styles/tokens.css`; section 3 owns those separately. Tokenization proceeds one family at a time with computed-style, screenshot, hit-target and motion comparisons. Any deliberate restyling or accessibility improvement beyond equivalent extraction requires its own review.

## 3. CENTRALIZED UI LAYERING

### CURRENT: stacking contexts and effective ordering

There is no single global z-index scale. `index.html:#map` is fixed and establishes a stacking context even with `z-index:auto`. Leaflet's translated/zoomed map pane introduces additional contexts. Leaflet control corners remain descendants of `#map`. Body-mounted panels, banner, toolbar, connectors, and notifications belong to document-level contexts instead.

```text
document
  #map (fixed; stacking context)
    Leaflet map pane
      tile / overlay / marker / GPS / wait-focus / tooltip / popup panes
    Leaflet control corners
      original controls, including GPS/recenter
  selected-day banner
    day menu (local child layer)
  search panel (reparented to body), itinerary wrapper, waits panel
  connector, seam, toolbar
  GPS notice, offline notice
```

The following are final declared values after the main-page stylesheet order, not browser-captured computed-style results. Each number is meaningful only within the stated context:

| Context | CURRENT layer / value | Evidence and interaction |
| --- | --- | --- |
| Map pane | Tiles 200; normal overlays 400; shadows 500; markers 600; tooltips 650; popups 700 | Targeted `vendor/leaflet.css` pane rules. Walking paths and search/itinerary focus circles use the default overlay pane; numbered stops and wait bubbles use marker pane. Preserve Leaflet-managed marker position/hover ordering. |
| Map pane | GPS 625 | `gps-ui.js`: `greyson-gps-pane`; pane ignores pointer events and graphics use `interactive:false`. |
| Map pane | Wait focus 640 | `toolbar-ui.js`: `greyson-wait-focus-pane`; noninteractive highlight and separate tooltip. |
| Map context / corners | Map pane default 400; zoom box 800; control corners 1000; controls 800 within their corner | `vendor/leaflet.css`. Corner containers pass pointers through; controls restore interaction. GPS remains here even though `toolbar-ui.css` makes its control fixed. |
| Document | Day banner 1350; day menu 5 inside that banner | `itinerary-ui.css`: `.greyson-active-day`, `.greyson-active-day-menu`. Child 5 cannot escape parent 1350. |
| Document | Connector 2700 | `toolbar-ui.css:#greyson-panel-connector`; decorative, `pointer-events:none`. |
| Document | Primary panel roots 2800 | Final search/waits rules and `.greyson-itinerary-app` override in `toolbar-ui.css`. Itinerary wrapper passes pointers through; its panel restores `auto`. Search is a body child after toolbar startup. |
| Document | Connector seam 2850; toolbar 2900 | Final `toolbar-ui.css` overrides. Seam is decorative; toolbar is interactive. Initial toolbar 2600 is superseded. |
| Document | GPS toast 3000; offline toast 3100 | `gps-ui.css:.greyson-gps-toast` and `offline-ui.js` injected style. GPS toast is click-dismissable; offline status passes pointers through. |
| Developer helper only | Calibration panel 9999 | `greys_place_calibration_snippet.js:#greyson-cal-tool`. Not a production modal allocation. |
| Separate editor page | Editor panel 1100; map help 950 | `route-editor.css`. Editor panel is a flex item, becoming absolute below 760px; help is noninteractive. Do not treat this page as sharing the main shell's context tree. |

`index.html:.map-info` declares 1000 but is hidden by `itinerary-ui.css`. Older Focus/Search control 2000 and `:has` top-left-corner 1700 rules are legacy conditional rules, not root panel levels. The retired Focus control is not constructed; the relocated search panel no longer matches its former parent's `:has` condition. A high nested value cannot promote a control above a body-level banner.

The comment in `toolbar-ui.css` describing panel/connector ordering does not match the numeric ordering: connector 2700 is behind panels 2800. Preserve actual ordering initially rather than implementing the comment as a visual change. Transforms and glass/backdrop filters also create or affect contexts/containing blocks; do not reparent elements, add `isolation`/`contain`, or remove transforms as incidental cleanup.

### PROPOSED: one registry, context-qualified tokens

Create one small `app/ui/layers.js` registry as the sole first-party source of numeric layer assignments and interaction metadata. During the classic-script transition, load this proposed registry synchronously in the document head before consuming application stylesheets and body content, without `async`, `defer`, or module loading. It publishes CSS custom properties through a small generated style block before those styles or feature mounting can use them, and exposes the same registry's pane values through the bounded compatibility facade. CSS consumes the published variables with no competing hand-copied numeric fallbacks; Leaflet pane assignments use that same registry. Do not add z-index literals to the visual tokens stylesheet or introduce an asynchronous layer-configuration fetch. Keep vendor defaults intact initially and document them as external contracts.

This first-paint mechanism is PROPOSED, not present today. Retain existing literal rules until a reviewed extraction slice supplies their variables synchronously; do not ship partially wired styles that depend on a later module import to become visible or correctly layered. Later ES module adoption must preserve the early publisher, or separately prove an equivalent single-source first-paint mechanism. A generated CSS alternative would have to derive from the same registry with a verified regeneration path, not become an independently edited numerical source. Such generation and any CSP implications require review; no build tool or runtime change is introduced here.

Proposed names are `map.tiles`, `map.paths`, `map.markers`, `map.gps`, `map.waitFocus`, `map.labels`, `map.popups`, `ui.dayBanner`, `day.menu`, `ui.connector`, `ui.panel`, `ui.seam`, `ui.toolbar`, `ui.gpsNotice`, `ui.offlineNotice`, and editor-specific panel/help categories. Seed them only with the CURRENT values above. Store expected host/context and pointer policy alongside category names. CSS variables could be named `--dav2-layer-panel`, `--dav2-layer-toolbar`, and so on; these do not exist yet.

Rules for the proposed registry and shell:

1. Keep existing parents, numeric values, stylesheet precedence and hit testing during token extraction. GPS control promotion to a body host is a separate decision and visual change.
2. Preserve one active primary panel. Keep the day menu as an anchored popover with its existing relationship to itinerary closing and toolbar switching; review any stronger exclusivity policy before implementing it.
3. Decorative connectors, seams, noninteractive map geometry, accuracy graphics and passive offline notices must pass pointers through. Interactive panels/controls must stop map drag/zoom propagation without swallowing unrelated capture-phase selection dismissal.
4. Preserve `[hidden]`, focus return, Escape handling, and the current absence of a modal backdrop for ordinary panels. A `role="dialog"` label alone is not permission to make a panel blocking.
5. A future blocking modal must take interaction priority, provide focus containment/return and background inertness, and place its status messages inside its own accessible context. Its implementation and numeric/top-layer policy require review; do not invent another large z-index or reuse the calibration tool's 9999.
6. Keep notification channels separate initially: GPS uses a 3500ms, click-dismissable status; offline notices use a 3300ms passive status. Consolidating hosts must not silently queue away or make these notices block map gestures.

### CURRENT and PROPOSED: viewport geometry

CURRENT `toolbar-ui.js:updateExpandedPanelTop()` sets `--greyson-expanded-panel-top` to `ceil(dayBanner.bottom + 6)px`, updating on window resize, `visualViewport.resize`, and a banner `ResizeObserver`. CSS fallback is `104px + safe-area-inset-top`; final panel bottom is `88px + safe-area-inset-bottom`. Later expanded-window rules override earlier `dvh` caps: search/waits have automatic height with no maximum, and itinerary's inner panel fills its bounded wrapper.

CURRENT toolbar bottom is `12px + safe-bottom`; GPS bottom is `101px + safe-bottom`; banner top is 8px plus safe-top, becoming 6px at the 540px breakpoint. `positionConnector()` measures real panel/button rectangles. These measurements, glass surfaces and connector alignment are part of the visual baseline.

PROPOSED viewport/layout service centralizes those existing values and coalesces measurements without changing geometry. A transparent full-screen host must use `pointer-events:none` with explicit interactive children and must not create a new trapping stacking context accidentally. Preserve 900px camera, 540px panel/banner, 420/390/350px toolbar, and 760px editor breakpoints first. Horizontal safe-area handling, keyboard/visual-viewport offsets, hinge occlusion and new breakpoint policy require captured-device review rather than guessed values.

## 4. STATE AND COMMUNICATION

### PROPOSED: explicit ownership

| State | Sole owning service/controller | Read/command contract |
| --- | --- | --- |
| Map instance, current view and bounds profile | Map engine per entry point | Features request view changes; subscribe to view snapshots. Renderers receive scoped map access; other modules cannot replace the map. |
| Selected day, active itinerary tab, Anaheim rollover policy | Itinerary controller with park clock | `selectDay(date)` validates supported dates; views consume projected state. Preserve restore/rollover rules before making supported dates data-driven. |
| Active primary panel, opener/focus target, day-menu visibility | Panel controller / shell adapter | Commands `open/close/toggle`; toolbar selection derives from controller state, not a second mutable flag. Preserve existing day-menu policy during adoption. |
| Stops and walking visibility | Separate overlay controllers; itinerary coordinates selected day | Two independent booleans. Closing a panel must not clear overlays. Preserve explicit-toggle fit behavior versus restore-without-fit behavior. |
| Selected place / focus | Selection controller with search, itinerary and waits channels | Stable place IDs plus source; preserve independent current highlights and capture-phase clearing. Any exclusive-selection redesign is an open decision. |
| GPS intent, watch/fix/error status | GPS controller | Own watch handles/coordinates locally; expose a small display snapshot. Do not serialize watch IDs or add new GPS persistence. Map-view persistence remains separate. |
| Live feed items/freshness/pending/errors | One wait-feed service | Immediate subscribe snapshot and unsubscribe; both map/list views observe it. No polling in the toolbar. |
| Session preferences / editor draft | Separate persistence adapters, invoked by their owners | Retain exact existing keys and schemas; no V1 reads, automatic migration, storage clearing, or sharing editor/main sessions. |
| Download progress / availability | Offline client, with worker-owned caches | Subscribe to page status; do not copy Cache objects or worker mutable state into the UI store. |

PROPOSED shared state contains small serializable snapshots, not Leaflet instances, DOM nodes, geolocation watch handles, timers, AbortControllers, or full duplicate copies of every dataset. Read-only repositories and owned lifecycle resources remain outside the store. Search input/filter state and editor history can remain local until another feature genuinely needs them. Do not persist new preferences simply because a store exists.

Prefer direct calls to injected controller APIs for commands and scoped subscriptions for observations. If DOM custom events are needed at a compatibility boundary, emit them on that application root with named payload contracts, not on unrestricted `window`/`document`. Existing browser/Leaflet events still need explicit ownership and teardown.

Example PROPOSED flow: a day-menu action calls `itinerary.selectDay(date)`; that owner validates the date, resets the appropriate tab, updates its state once, tells each visible overlay to render for that day, then saves the existing session shape. Toolbar clicks call only the panel controller. A wait result calls the waits selection channel and map focus operation after panel closure.

Each factory should own a `dispose()` that removes its event listeners, subscriptions, controls, markers/layers, timers, observers and in-flight work as appropriate. Maintain pagehide/visibility/pageshow behavior; cancellation and late-response guards must prevent obsolete data from rendering for a previous day or disposed feature. Disposal must not delete stored drafts/caches or clear another owner's graphics. Do not run legacy and replacement event handlers for the same interaction simultaneously.

## 5. DATA CONTRACTS

### CURRENT: files and joins

| File | Current shape / meaningful fields | Consumers and invariants |
| --- | --- | --- |
| `places.json` | Metadata plus `places[]`; stable `id`, `name`, `park`, `area`, `type`, `lat`, `lon`, `zoom`, `aliases`, `accuracy`; optional source/status metadata. | Search, itinerary, waits and editor. IDs are identities, not display names. Coordinates are latitude then longitude; alias/name matching is not an external API ID. |
| `itinerary-2026.json` | Metadata plus `days[]`; ISO `date`, day labels/summary, `timeline[]`, `lightningLane[]`, optional detail arrays/notes. Entries use display `time`, `activity`, `notes`, optional `mapIds[]`. | Itinerary view/highlighting and editor. `mapIds` resolve to place IDs and may contain several targets. Top-level planning metadata is not a substitute for consumed day fields. |
| `route-2026.json` | Metadata plus `routes[date][]` of `{placeId,timelineIndex}` in stop order. | `itinerary-ui.js:routeStopsForDay` and editor startup join place IDs to `places[]` and zero-based `timelineIndex` to that day's timeline. Timing is read from the timeline, not copied into this file. |
| `walking-routes-2026.json` | Metadata plus `routes[date].{status,segments[]}`; segment `name`, `kind`, `legFrom`, `legTo`, `points:[[lat,lon],...]`. | Main walking overlay renders geometry; `kind==='park-hop'` changes dashing. Editor imports/exports one-based consecutive stop legs, with `legTo===legFrom+1`. |
| `tile-index.json` | `{version,count,bytes,files:string[]}`. | Offline downloader uses relative `files`; counts/bytes describe the inventory. Generator writes it; worker precaches it. |

Inspection confirmed 317 unique place IDs, six itinerary days, three ordered-route dates with 40 stops, and three walking dates with 37 segments/1,295 points. All 54 itinerary `mapIds` and all ordered-stop references resolve. Walking leg indices are valid consecutive stop positions. The index contains 4,602 unique tile paths and reports 174,042,995 bytes. These are baseline observations, not permanent limits.

Important CURRENT contracts:

- `routeStopsForDay(day)` uses the original route-array position plus one as the displayed stop number and silently filters missing place/timeline joins. Do not silently renumber or reorder surviving stops while extracting this function.
- `route-editor.js:ptsForLeg(day,index)` builds geometry from current stop coordinates plus stored interior `via` points. Import checks `legFrom/legTo`, converts to zero-based leg index, strips imported endpoints with `points.slice(1,-1)`, and reanchors endpoints to the current stops. Main walking rendering does not perform that reanchoring.
- `exportRoute()` exports only finished legs, omits unfinished legs, and downloads JSON without rewriting authoritative data. It preserves the visual-overlay/non-navigation distinction.
- `validDates` permits four park/rest dates; `WALKING_DATES`, editor `DAY_LABELS`/`DATES`, and editor HTML options cover three park dates. Six data days do not mean six selectable days. Current `routeable` metadata does not drive this policy.
- Reordering a timeline invalidates `timelineIndex`; reordering stops changes walking-leg identity. Changing a place coordinate changes the editor's reanchored endpoints. Dates, row order, stable IDs and geometry must therefore be reviewed together.

### CURRENT: validation and failure behavior

`places-ui.js` validates the top-level list and filters unsupported parks, missing names and nonfinite coordinates. `loadItinerary()` checks top-level days/routes/places; many nested fields are assumed. Its `fetchJsonFile()` retries once after 650ms with reload caching, and `loadingPromise` deduplicates in-flight initialization while allowing later retries. Failures restore an operable retry control.

`loadWalkingData()` validates all supported walking dates have nonempty segments and finite two-number points. It does not enforce review status, geographic bounds, complete leg coverage or endpoint alignment. Editor `isValidPoint()` adds geographic bounds; draft restoration checks per-day leg counts and import individually rejects invalid/nonadjacent legs. These are different current policies, not one shared validator.

`offline-ui.js:ensureManifest()` checks a nonempty string array matching the local `tiles/(16|17|18|19|20)/x/y.jpg` layout. It does not itself verify count/bytes metadata, uniqueness or physical presence. The existing `build_tile_index.py` is a writing generator, not a read-only check. `setup_map.py` requests a full rectangle and is not an exact reproduction of the curated set.

### PROPOSED: reusable contracts without behavioral changes

Add pure, small validation/join helpers with diagnostics for duplicate/missing IDs, invalid date joins, out-of-range timeline indices, malformed points/legs and index consistency. Initially run stronger checks as development diagnostics; do not make a currently permissive production consumer reject data without explicit review. Keep editor import constraints separate from display-only walking validation.

Repositories should share immutable raw/indexed data and explicit loading state, not feature UI state. Preserve current query ranking, alias matching, fetch/cache/retry policies and error presentation before considering deduplication or normalization unification. A failed in-flight request must remain retryable rather than being cached forever as a rejected promise. Keep exact user-facing time strings and metadata; contracts do not authorize content rewriting.

## 6. RESPONSIVE AND PWA REQUIREMENTS

### CURRENT requirements to preserve

- Desktop camera/zoom behavior and iPhone-specific touch behavior must remain distinct. `ios-double-tap.js` detects iOS touch, uses passive listeners, ignores controls/markers/form elements, and yields moved gestures to the touch-drag plugin. Preserve its thresholds and exclusions before any gesture redesign.
- Keep `viewport-fit=cover`, full-viewport fixed main map, body overflow policy, safe-area offsets, expanded-panel measurements, search input sizing, glass/connector appearance and editor collapse behavior. Main map rotation/resize must preserve a deliberately chosen view; itinerary day rollover has a different intentional reset policy.
- Keep the root entry points and portable relative asset paths. A future module must resolve data through the application-root path service; `new URL('places.json', import.meta.url)` inside a nested feature directory would target the wrong location.
- Preserve desktop, iPhone Safari, installed iPhone PWA, tablet and foldable use. Current code has width-based rules, not a dedicated foldable/hinge or standalone-PWA layout branch. Treat the latter as required test cases, not already proven implementations.
- Retain all existing application data/artwork, the curated tile set, local vendor files and editor behavior. V1 is neither a source of fallback assets nor a deployment target for architecture work.

### CURRENT offline and identity contracts

`offline-ui.js:start` registers application-directory-relative `sw.js` through `new URL('sw.js', root)` with `root.pathname` as scope. Worker `APP_ROOT` derives from registration scope. `sw.js` handles only same-origin, in-scope GET requests; cross-origin live waits bypass it. Installation precaches 28 entries and then calls `skipWaiting()`. Activation deletes only obsolete `dav2-shell-*` caches and then calls `clients.claim()`; it does not delete the persistent tile cache.

Tiles are cache-first. `X-Disney-Bulk:'1'` bypasses worker tile writes because the downloader writes successful responses itself. Other resources are network-first with cached fallback when fetch throws; ordinary HTTP errors do not automatically invoke offline fallback. Navigation may fall back to cached `index.html`, otherwise the worker can return 503.

The page downloader uses six workers, pauses/aborts for page hiding, skips starting while offline, and retries missing tiles on later visible/online checks. It does not guarantee that an already-active download loop aborts immediately when connectivity changes. Persistence is best effort; storage quota, eviction and Safari connection hints remain platform-dependent.

The editor is absent from shell precaching and does not register a worker itself. Fresh offline editor use is not established functionality. Supporting it intentionally is a separate decision, not a silent refactor assumption. The existing manifest declares the 1254x1254 `icon-512.png` as 512x512; correcting that mismatch is separate work.

Development manifest ID stays `/dav2-development/`, short/Apple title stays `Disney V2 Dev`, and `start_url`/`scope` stay `./`. Original production ID `./` and production branding are reserved for a deliberate, separately approved V1-path cutover with upgrade testing. Preserve the current DAv2 storage keys and cache family; never migrate/delete V1 browser data.

### PROPOSED migration gates

Any module/path change must update the worker's complete required asset graph, including transitive imports and styles, in the same reviewed slice. Keep `sw.js` and the manifest at the root so scope does not narrow accidentally. Do not change the worker to module format merely because page code adopts modules.

Test online cold start, fully cached offline start, interrupted downloads, re-entry after backgrounding, unavailable storage, API partial/offline failure and an existing installed application's update path. Shell version changes must stay within the DAv2 prefix and be deliberate; do not rename the tile cache or flush origin-wide storage to make tests pass. Same-origin quota and simultaneous DAv2 deployments are not isolated by cache names alone.

Use the dedicated DAv2 localhost origin described in README for desktop checks. A physical iPhone needs an appropriate secure test endpoint; a computer's localhost does not serve the phone. No test deployment is authorized by this document.

## 7. PROPOSED DIRECTORY STRUCTURE

This is a future layout, not an instruction to move files now. Root data/assets and PWA entry points can remain stable throughout the initial migration. Files under `app/` are proposed modules; existing first-party scripts remain the compatibility implementation until individually replaced.

```text
DAv2/
  index.html
  route-editor.html
  manifest.webmanifest
  sw.js
  app/
    main.js
    editor.js
    bootstrap/             main-bootstrap.js, editor-bootstrap.js
    config/                map-profiles.js, resource-paths.js
    core/                  state.js, lifecycle.js, clock.js, storage.js, compat.js (transitional)
    data/                  repositories.js, contracts.js
    map/
      map-engine.js
      gestures.js
      overlays/            selections.js, stops.js, walking.js
    ui/                    shell.js, layers.js, panels.js, toolbar.js, notifications.js
    features/
      places/              controller.js, search.js, view.js
      itinerary/           controller.js, schedule.js, view.js
      gps/                 controller.js, leaflet-renderer.js
      waits/               feed.js, view.js, leaflet-renderer.js
    offline/               client.js, tile-inventory.js
    editor/                controller.js, draft.js, geometry.js, history.js, view.js
  styles/                  tokens.css, base.css, shell.css, feature styles, editor.css
  places.json
  itinerary-2026.json
  route-2026.json
  walking-routes-2026.json
  tile-index.json
  tiles/                   existing curated coordinate hierarchy
  icons/                   existing imagery
  vendor/                  existing local dependencies
  archive/legacy-v1-patches/
  setup_map.py
  build_tile_index.py
  greys_place_calibration_snippet.js
  disney-castle-toolbar.png
  AGENTS.md
  README.md
  ARCHITECTURE.md
  .gitignore
  .gitattributes
```

Start with fewer files where responsibilities are small; a folder does not require a class hierarchy or separate controller/view for trivial code. Root utilities remain in place because moving scripts that derive paths from their own directory changes their write targets. Archived scripts stay historical and are not reused as migration automation.

## 8. INCREMENTAL MIGRATION PLAN

Everything below is **PROPOSED** future work requiring review. Each row is a boundary, not a mandate to combine all its substeps into one patch. Migrate one consumer or feature at a time, with only one active implementation for each interaction. No whole-application rewrite, dependency installation or directory move is part of this document.

| Phase | Focused scope | Acceptance criteria and regression checks | Rollback strategy |
| --- | --- | --- | --- |
| 0. Characterize baseline | Use commit `4de6524820c9285480c4ee26306b549c5b76fa44` as the immutable reference and execute the characterization protocol below for both entry points. | Record repeatable desktop, screenshot/viewport, gesture, panel, data, offline/failure and performance evidence. Real iPhone Safari/PWA results require actual device execution; mark unavailable checks NOT RUN or BLOCKED. | No runtime replacement; retain the exact baseline, run manifest and comparison evidence. Never reset the active worktree or clear production/V1 storage to reproduce a run. |
| 1. Pure foundations | Extract resource-root resolution, data joins/diagnostics, clock helpers and existing-key storage adapters in separate small slices. Initially preserve callers/policies. | Pure-function cases reproduce current ranking/join/time/date behavior; same stored shapes/keys, no legacy reads or new writes, no data changes. Test malformed/unavailable input and retry paths. | Remove that slice's adapter/call-site change; existing files/data/drafts remain usable. |
| 2. Map/bootstrap seam | Extract map creation/configuration while retaining the classic lexical compatibility seam. Keep main/editor profiles separate; do not switch all scripts to modules at once. | Identical presets, bounds, zoom settings, render/touch options and resize behavior; one map per entry point; no silent feature exits from lost `map`/`L` bindings. | Restore the original bootstrap call path and script ordering; do not touch stored state or assets. |
| 3. Layer registry | Replace one family of first-party layer literals with tokens from one registry; retain current DOM parents and CSS ordering. | Same computed values, context ancestry, pointer targets, connector geometry and screenshots at key widths. Test GPS/control nesting and map gestures under transparent UI. | Revert token wiring for that family; retain original selectors/hosts. No simultaneous context promotion. |
| 4. Panel/toolbar contract | Introduce panel adapters for current controls, then make one controller own switching/focus/connector state. Remove competing handlers only as ownership transfers. | Search/itinerary/waits switching, reopen/close, Escape, outside/capture taps, day menu, ARIA state, input focus and selected-tab connector remain consistent. | Restore legacy toolbar/feature event ownership together for the affected panel; never run both handlers. |
| 5. Small feature slices | Extract search, wait feed/renderers, GPS, then notification channels as separate reviewed changes. | Preserve search ranking/filters, wait matching/status/refresh and partial failure, zoom-18 bubbles, independent selection clearing, GPS freshness/bounds/recenter/background lifecycle, and notice duration/hit testing. No duplicate polling/watchers. | Restore only that feature's legacy entry and dispose replacement resources. Keep existing keys and drafts. |
| 6. Itinerary and overlays | First move day/session orchestration; then views/highlighting; then numbered stops; then walking loading/rendering. Keep these separate patches. | Same supported dates/default preview, Anaheim rollover, highlight timing/scrolling, restored view without auto-fit, independent toggles, asynchronous day-switch safety, retry and popup content. Cross-file joins still pass. | Restore the affected controller/view/overlay adapter; keep source JSON and persisted schema unchanged. |
| 7. Editor reuse | Share only proven data/coordinate helpers; extract editor model/history/view while keeping its distinct map configuration and storage. | Draft restore, import confirmation/reanchoring, draw/refine/drag, undo/redo, leg completion, completed-only export, collapse behavior and reload persistence match baseline. No automatic main-session coupling or new offline promises. | Restore editor's standalone implementation; preserve/export test drafts before switching implementations. |
| 8. Module/path consolidation | After dependency seams are stable, adopt the reviewed module format and move one small file family at a time; remove obsolete globals/selectors only after last consumer migrates. | Explicit startup/lifecycle, valid relative URLs/CSS images, complete transitive offline asset list, no duplicate initialization, cold/installed/offline startup and portable-path tests pass. | Restore the previous entry scripts and matching asset/precache graph as one compatible set. Keep root worker/scope and tile cache. |
| 9. Compatibility qualification | Re-run the full baseline matrix, document residual gaps, and separately propose any deliberate improvement such as offline editor support or icon correction. | Functional/visual parity on available target devices, accepted exceptions documented, cache update/rollback tested, no V1 requests or scope/storage interference. | Return to the last accepted feature/asset set; do not bundle a production identity change or deployment with qualification. |

### PROPOSED: reproducible Phase 0 characterization protocol

**Reference:** commit `4de6524820c9285480c4ee26306b549c5b76fa44`, before architecture-driven runtime changes. Its identity can be verified read-only with `git show --no-patch --format=fuller 4de6524820c9285480c4ee26306b549c5b76fa44`. The checks below are a plan, not a test-results record. No desktop-browser, performance, installed-PWA or physical-device pass is asserted by adding this plan.

For a future approved run, serve a separate, verified export of that commit and the candidate on distinct DAv2-only origins, for example baseline `localhost:8765` and candidate `localhost:8766`. Do not checkout/reset the active worktree to recreate the baseline, use V1 as the reference, or reuse a production browser profile. Export/serving, harnesses and evidence artifacts require a separate authorized task; they are not created here.

Each run must record reference/candidate commit or source fingerprint, entry URL, browser/version, OS, viewport in CSS pixels, device-pixel ratio, browser zoom, font environment, locale/timezone, Anaheim date/time, selected day/tab, map center/zoom, overlay flags, storage seed, worker controller/version, cache completeness and network mode. Fix browser zoom at 100% and keep these inputs identical for each comparison. Use a recorded supported day and deterministic API/geolocation fixtures where a browser harness can intercept them. Clock-controlled cases must be confined to the harness, not changes to baseline source; otherwise record actual time and avoid comparing time-sensitive states across different park days. Preserve seeded drafts and use a dedicated test profile.

| Check group | Repeatable procedure and observations |
| --- | --- |
| Desktop startup and map | Load `index.html` from an empty test profile, then repeat a warm load. Record console/resource errors, initial map center/zoom and visible controls. Pan to each edge, zoom to minimum/maximum, wheel zoom, double-click a fixed map location, then use resort recenter. Repeat resize across the 900px camera boundary without clicking recenter: preserve the current user-selected view rather than assuming resize should apply another initial preset. Verify main and editor retain their distinct map profiles. |
| Search and panels | Select and record a fixed public POI ID/query/alias and typo query from the existing directory. Record result ordering and park/type filtering, clear-input behavior, selected marker/label and next-pointer clearing. Open Search, switch to Itinerary, switch to Wait Times, close/reopen the active panel, use close buttons/Escape/outside taps, then operate the day menu. Compare focus targets, ARIA state, hidden states, connector/seam alignment and whether inside-panel actions reach the map. Include capture-phase dismissal, not only click handlers. |
| Itinerary and overlays | Select each currently supported day; toggle Stops and Route independently, then both together, and close panels without clearing overlays. Open stop popups and itinerary map links. Reload with seeded same-day view/overlay state and verify restoration does not auto-fit; compare new-day reset and unsupported-day preview using recorded clock inputs. Check NOW/UP NEXT and opening-only auto-scroll immediately before/during/after a recorded schedule period. |
| Route editor | Open `route-editor.html` separately with an isolated draft. Select the same recorded day/leg, draw/refine/drag bends, undo/redo, complete a leg and export. Confirm unfinished legs are omitted; reimport a compatible fixture and compare endpoint reanchoring, draft replacement confirmation, selected leg and reload persistence. Exercise collapse at the 760px boundary. Downloads stay test artifacts; do not overwrite authoritative JSON. |
| Screenshots and geometry | Capture identical named states: initial map; each primary panel; day menu; both overlays; a selected place/pop-up; GPS/offline notices; editor expanded/collapsed. Use representative CSS viewports 1440x900, 768x1024, 390x844 and 844x390; probe immediately below/at/above current 900, 760, 540, 420, 390 and 350px breakpoints as applicable. Record CSS rectangles, computed layer values, ancestor contexts and pointer targets alongside images. Keep DPR/browser/fonts identical, wait for settled map animation, and use fixed data/timestamps where possible. |
| Visual comparison criteria | Accept no unexplained movement, clipping, wrapping, touch-target loss, changed overlap, safe-area intrusion or connector gap. Check map-pane and body-root ordering separately. Annotate narrow, justified masks for live timestamps/feed text or raster antialiasing only; do not mask entire panels/maps or use a broad pixel-difference percentage to hide regressions. Cross-platform font/raster differences require separate references, not an automatic pass. |
| Data joins | Automate unique place IDs, all `mapIds`, route date/place/timeline joins, zero-based timeline indices, one-based consecutive walking legs, finite coordinate pairs, segment/stop alignment diagnostics and unique tile inventory metadata. Record baseline counts already established in section 5 and compare candidate results. Stronger diagnostics must not silently change current consumer acceptance rules. Index/file existence checks report aggregate results, not thousands of paths. |
| Offline/PWA | In isolated contexts, record first visit without a worker, installation/controller readiness, 28 shell entries, and complete versus incomplete tile caches. Test warm offline reload, cached routes, a missing tile, hide/show interruption and missing-tile retry, persistence denial/quota failure where reproducible, and return online. Record the known first-offline-editor limitation rather than requiring new coverage. Test an existing installed baseline upgrading to the candidate separately from a fresh install; verify the DAv2 identity, scope, cache names and retained drafts/tiles. Never clear unrelated origin storage. |
| Network/API failures | Compare one park feed failing, both failing, offline transition and recovery, slow/delayed responses, malformed/static JSON failure and retry, walking-data failure, and switching day while walking data loads. Distinguish a rejected network request from HTTP 404/500: current worker fallback is not identical for those cases. Use deterministic intercepted responses when tooling permits, record fixture identity and retain a separate live-service smoke check. A cached response must not accidentally conceal the failure being tested. |
| Gesture/viewport checks | Desktop drag/wheel/double-click checks can be repeated locally. Touch emulation can probe exclusions and multipointer handling, but real iPhone checks must cover pinch, two-tap zoom, double-touch-and-drag, panning, tapping markers/controls, keyboard open/close, portrait/landscape, browser chrome changes and background/resume in Safari and installed mode. Include tablet split view and foldable/hinge layouts when actual supported hardware or an identified emulator is available; label emulation explicitly. |
| Basic performance | On the same machine/profile conditions, record at least three cold and three warm runs separately. Observe navigation timing, first visible map/usable controls, JSON/request counts, tile transfer/cache hits, live-feed request cadence, main-thread stalls during pan/panel use, and responsiveness while six-worker downloading runs. Note CPU/network throttling and cache state; compare median and range to the reference rather than inventing a new performance budget. Repeated open/close/background cycles must not introduce extra polling, watchers, controls or growing listener/layer counts. Record unavailable metrics as unmeasured. |

**Execution boundary:** Codex can automate read-only Git identity/content checks, JSON/data-join checks, syntax checks, local asset-path checks and aggregate inventory comparisons with the existing local tools. Browser interaction, screenshots, request interception, clock/geolocation simulation and performance capture can be automated only if an appropriate browser/driver is already available and its use is authorized; no such runner is currently configured by this repository. Otherwise those desktop checks are manual, not presumed passes. This documentation task installs nothing and executes none of the proposed browser checks.

Physical iPhone Safari, installed iPhone PWA, OS permission dialogs, real GPS, actual safe-area/browser-chrome behavior and physical touch quality require actual device access and executed checks. Desktop emulation cannot certify them. Device testing also needs an appropriate secure endpoint; creating or deploying one is separate work. Record device model, OS/browser, installation state, steps and evidence for every claimed pass.

Use a run manifest and case IDs with **PASS, FAIL, NOT RUN or BLOCKED**, actual observations, artifact references and approved deviations. Screenshots/recordings/traces belong in a separately approved evidence location, not silently in the source tree. Phase 0 completes only when the agreed required cases have evidence and remaining gaps are explicitly reviewed; unavailable devices or tooling remain unresolved coverage, never inferred success.

For every phase, record intended scope/impact before editing, checks actually performed, and the exact source/asset set needed for reversal. Focused tests should check contracts and observable behavior, not merely mirror implementation. A lightweight future read-only contract checker is useful; a new test framework is not a prerequisite for documentation or the first extraction.

Rollback is a reviewed source change, not an instruction to reset history, delete files or clear browser data. Worker/browser caches may outlive a source rollback: use a tested, coherent DAv2 shell version/asset graph while retaining tile data. Do not assume reverting a Git change automatically reverses installed-PWA state. Schema migrations, cache-name changes, and production cutover require their own plans and approvals.

## 9. OPEN DECISIONS AND RISKS

| Decision / risk | What is established and what needs review |
| --- | --- |
| Module format and transitional bootstrap | Use the proposed bounded classic-script compatibility facade while the lexical map and current startup order remain active. Review the isolated utility/component proof before selecting the ES module entry-point transition; offline/installed-PWA and actual-device evidence remain required gates. |
| Visual baseline ownership | Source values are known, but computed stacking, browser compositing, keyboard avoidance, focus return and real hit areas have not been captured by this inspection. Approve desktop/iPhone reference states before changing containers or contexts. |
| Panel/day-menu/selection policy | Current toolbar intends one primary panel; day menu and feature highlights have additional independent rules. Review any simplification rather than making exclusivity or modality a hidden behavior change. |
| Central state size | Keep domain ownership and local transient state. Do not duplicate all datasets, move every keystroke into a global store, or merge editor and main sessions for architectural symmetry. |
| Supported dates and time parsing | Current controls/date lists and 12-hour inference are code-driven. Decide separately whether to make them data-driven; preserve current Anaheim versus device-local clock distinctions first. |
| Shared validation and loading | Current guards and failure tolerances differ; stronger validation, request deduplication, caching and retry consolidation can change visible behavior. Introduce diagnostics and compare before enforcing new rules. |
| Shared selection normalizers | Place-search scoring and wait matching intentionally differ today. A single normalization helper is not automatically equivalent. Protect known alias/typo/status cases. |
| Offline module graph and update atomicity | Current worker is classic, root-scoped and network-first for code. Module moves and eager takeover can mix old pages/new resources; define tested shell-version and recovery behavior without touching V1 caches. |
| Offline editor | Current first-offline editor coverage is incomplete. Decide whether to preserve/document that limit or approve intentional precaching/registration changes later. |
| Storage contexts and quotas | Namespaces provide logical separation, not independent same-origin quotas. Browser tabs and installed PWAs can have different storage contexts. Test coexistence, retention and draft export/recovery on target devices. |
| Performance and disposal | No measured performance budget or feature teardown framework exists. Compare startup/network requests, frame behavior, listeners, polling/watch counts and tile-download responsiveness before/after each slice; no unmeasured optimization mandates. |
| Accessibility and motion | Labels/roles/Escape behavior exist, but keyboard-only, screen-reader, focus containment, contrast and motion acceptance are not established by static inspection. Review improvements explicitly while preserving existing touch behavior. |
| Foldables and safe areas | Width-based layouts exist; hinge segments, horizontal inset coverage, split view and keyboard/standalone viewport behavior need device/emulator evidence. Do not claim they are already fully supported. |
| External services | API availability, name matching, partial outages, geolocation freshness/permission, and Disney native-link fallback cannot be guaranteed from local sources. Keep failures isolated and test controlled cases. |
| Production replacement | V1 remains separate. Production identity restoration, original-URL hosting, installed-app compatibility, deployment settings and approval are outside this architectural migration's implementation authority. |

The next review should select only the first narrow implementation slice and its acceptance evidence. This document itself authorizes no code edits, file moves, installations, commits, pushes, deployments, Git configuration changes, or V1 access.
