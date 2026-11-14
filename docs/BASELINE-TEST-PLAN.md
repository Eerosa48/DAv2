# Phase 0: DAv2 baseline regression test plan

## Scope, reference, and execution status

Reference application commit: **`4de6524820c9285480c4ee26306b549c5b76fa44`**. This plan supports [ARCHITECTURE.md](../ARCHITECTURE.md), follows [AGENTS.md](../AGENTS.md), and uses the local-serving guidance in [README.md](../README.md). It characterizes existing behavior before modular refactoring; it does not authorize feature changes or corrective cleanup.

**This is a test specification, not a completed test report. All cases start as NOT RUN.** Source-derived expectations below are not evidence that a browser, iPhone, installed PWA, or performance test passed. No test runner is configured by this document. No dependencies, fixtures, screenshots, browser profiles, servers, or deployments are created by writing it.

Keep application data, artwork, tile paths, local dependencies, storage identifiers, and both entry points intact. Never access V1 or use its live application as the test reference. A pre-existing limitation is recorded as a baseline issue, not silently fixed or misrepresented as successful feature support.

## Priorities and execution methods

- **P0:** required characterization before the first code refactor. Execute applicable local/static and desktop cases, capture the visual references, and obtain actual iPhone Safari/PWA evidence for the device-critical cases. If access is unavailable, record BLOCKED/NOT RUN and obtain an explicit coverage-gap review before changing the affected behavior; emulation is not a substitute pass.
- **P1:** deeper resilience/platform coverage. Complete before refactoring the affected feature or approving compatibility qualification. P1 does not mean an observed failure can be ignored.
- **S — local static automation:** read-only Git, Node.js or PowerShell checks using available tools; no application execution required.
- **B — browser automation, conditional:** browser interactions, screenshots and assertions using an already available, separately authorized browser/driver. There is no existing repository browser-test command to invoke. Without tooling, execute the desktop steps manually (**M**) and label them accordingly.
- **H — controlled browser harness, conditional:** intercepted HTTP/API responses, clock/geolocation fixtures or storage fault simulation. Record instrumentation and fixtures; do not modify application source, contact V1, or assume all browser APIs can be overridden reliably.
- **D — physical/manual device:** actual device/OS interaction, Safari/Home Screen installation, permission prompts, real GPS and touch quality. Record device access and execution evidence. Desktop touch emulation is supplementary only.

Every row supplies steps, an observable oracle, method, and source basis. Compare candidate and reference under identical inputs. Record a FAIL for unexpected behavior; distinguish a new regression from a confirmed reference defect. Only explicitly reviewed exceptions may qualify a candidate with an existing issue.

## Reproducible setup and fixtures

1. Verify DAv2 root/origin, branch/source fingerprint, working-tree status and the reference object with read-only Git commands. Obtain a separately approved export of the reference commit if comparison needs two trees; do not reset/checkout over the active worktree. Serve only DAv2 sources. Use separate dedicated origins, such as reference `http://localhost:8765/` and candidate `http://localhost:8766/`, with consistent hostnames. Server/export creation is a future execution task, not part of creating this document.
2. Open both `/index.html` and `/route-editor.html`. Use isolated test profiles/contexts, not a production profile or an existing personal draft. Never clear unrelated origin storage, uninstall unrelated workers, overwrite repository JSON with a download, or use maintenance scripts to reset the reference.
3. Record browser/version, OS, CPU/network throttling, viewport CSS dimensions, DPR, browser zoom (100%), locale/timezone, font environment, Anaheim date/time, selected date/tab, view center/zoom, overlay flags, storage seed and worker/cache state. Baseline and candidate must use the same inputs; normalize only the different test origin when comparing resource URLs.
4. Build a recorded fixture manifest from the reference data: one supported route day, the supported rest day, an unsupported-date clock case, a public POI with an alias, a known typo query, an empty-result query, a selected editor leg, and compatible/invalid editor imports. Record exact inputs and expected ordered IDs in approved test evidence, without changing authoritative data. Generate invalid copies in harness memory or an approved test-artifact location only.
5. For deterministic screenshots, intercept both park feeds with fixed timestamps/names matching the recorded POIs, hold the clock at recorded values where tooling permits, and wait for map animations and layout measurements to settle. Keep a separate live-service smoke check. If clock interception is unavailable, record actual time and do not compare schedule highlights across different times/days as though deterministic.
6. Use cold (empty DAv2 test caches/drafts) and warm (controlled complete cache and recorded session) profiles separately. Application JSON is precached: arrange failures before caching, or intercept the intended request so a prior cached response cannot conceal the test. A browser offline switch and an HTTP 500 response are different fixtures.
7. Store evidence only in a separately approved location. Screenshots/traces can contain trip details; do not automatically add them to the public repository. Record each artifact's run/case identity and reference/candidate association.

### Environment and visual matrix

| Environment | Required observations |
| --- | --- |
| Desktop reference browser | P0 full main/editor flow at 1440x900; repeat representative flow in another available engine before compatibility qualification. Record OS fonts and DPR. |
| Narrow desktop/touch emulation | 390x844 and 844x390; test panel/keyboard assumptions without calling this an iPhone pass. |
| Tablet | 768x1024 and landscape; actual tablet touch/split-view checks where available. |
| Physical iPhone Safari | Actual model, OS/browser, portrait/landscape, browser chrome, keyboard, location permission and gestures. Requires an appropriate secure endpoint; computer localhost is not the phone's localhost. No deployment is authorized here. |
| Installed iPhone PWA | Actual Home Screen launch, standalone viewport, background/resume, offline reopening and update behavior, separately from Safari. Do not assume shared storage. |
| Foldable | Recorded folded/unfolded widths, orientation and hinge posture; label real hardware versus identified emulation. Current code has no dedicated hinge layout branch. |

Probe immediately below/at/above the current 900px camera, 760px editor, 540px panel/banner, and 420/390/350px toolbar breakpoints. Representative screenshot sizes are test inputs, not proposed new design dimensions. Capture initial map, each open panel, day menu, both overlays, selection/pop-up, notifications, and expanded/collapsed editor. Compare exact CSS rectangles/context ancestry and narrow image differences in the same browser/DPR/fonts. No unexplained clipping, wrapping, overlap, connector gap or lost hit target is acceptable. Mask only recorded dynamic timestamps/feed text or justified antialiasing differences, never entire panels or maps.

## 1. Main map, cameras, gestures, and resizing

Source: [index.html](../index.html), `L.map`, `cameraViews`, `desktopCameraViews`, `updateMinZoom`, `configureNavigationBounds`, `focusInitialView`, `focusPark`; [ios-double-tap.js](../ios-double-tap.js), `onStart/onMove/onEnd`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| MAP-01 P0 | Cold-load main at widths below and above 900px, with no saved session. Wait for itinerary startup and tiles. | One map and one tile layer initialize; view matches `cameraViews.initialMobile` (zoom 16) or `initialDesktop` (zoom 17), using the existing center. Effective minimum becomes 15; maximum is 20. Controls/data loading do not silently disappear because of missing globals. Record center/zoom and console. | B/M; S checks options/startup order. |
| MAP-02 P0 | Pan toward all coverage edges; use wheel zoom and repeated desktop double-clicks, including at minimum/maximum. | Local XYZ tiles, no wrap; existing bounds policy and maximum-bound viscosity match reference after motion settles. No unbounded world wrapping, duplicate zoom handling, or changed zoom limits. Compare projected bounds from `configureNavigationBounds`, not an invented rule that center must always remain inside the unexpanded street rectangle. | B/M. |
| MAP-03 P0 | Pan/zoom away, click resort recenter, then repeat on each side of 900px. In a harness/DevTools only, invoke each existing `focusPark` preset at identical widths. | Recenter matches `focusInitialView(true)` and preset values match the appropriate camera object. Do not invent a visible Locations dropdown: it is retired. Capture view after animation and compare reference/candidate. | B/H or M with DevTools. |
| MAP-04 P0 | Set a nondefault view, resize across 899/900/901 and rotate narrow/wide viewport without recentering. | `invalidateSize` and bounds/minimum updates occur without deliberately resetting to another initial preset. Compare center/view using the same resize sequence; no blank tile gaps or map-size mismatch. | B/M; D for actual rotation. |
| MAP-05 P0 | On iPhone, perform two stationary taps, pinch, pan and double-touch-and-drag; repeat on controls, markers and input fields. | One discrete two-tap zoom, not duplicate Leaflet/fallback zoom; moved gestures remain available to drag-zoom/pan. Controls/markers/forms are excluded; desktop double-click still works. Source thresholds are 300ms tap, 350ms gap, 12px movement and 40px separation; exercise comfortably inside/outside thresholds, recording real-device variability. | D; H/B supplementary, not certification. |

## 2. Toolbar, panels, connectors, layering, and close behavior

Source: [toolbar-ui.js](../toolbar-ui.js), `toggleTool`, `closeAll`, `activeTool`, `updateActive`, `positionConnector`, `updateExpandedPanelTop`; `toolbar-ui.css`, `itinerary-ui.css`, `places-ui.css`, `gps-ui.css`, and `offline-ui.js` style injection. Context inventory is in ARCHITECTURE section 3.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| UI-01 P0 | Open Search, switch to Itinerary, then Wait Times; click the active tab again and reopen. | Primary panel switching closes the others; active button style/`aria-expanded` and visible panel agree. The external Disney link is not a fourth panel. No duplicate panels/listeners. Record DOM visibility and selected toolbar item after every action. | B/M. |
| UI-02 P0 | For each panel use its close button, Escape and outside-map pointer; tap inside scroll/filter/content areas and use toolbar switches. | Match each original close/focus-return policy. Inside-panel interactions do not unintentionally close the panel or drag/zoom the map. Capture-phase selection removal still occurs as in reference; propagation handling must not create double toggles. Search is not a focus-trapped modal. | B/M; D for touch propagation. |
| UI-03 P0 | Open day menu with itinerary open; choose a day; open menu again and switch toolbar tabs. | Opening the menu closes itinerary; selecting closes the menu; toolbar switching closes it. Compare other panel interactions to reference rather than imposing new global menu exclusivity. Banner/button ARIA state follows visibility. | B/M. |
| UI-04 P0 | Capture each panel at representative widths and boundary probes; resize with it open and inspect rectangles/stacking ancestors. | Preserve final document layers: connector 2700, panels 2800, seam 2850, toolbar 2900; day banner 1350 with local menu 5. Search remains body-mounted; GPS control remains nested under Leaflet. Compare pane-local values separately. Panel top follows measured banner bottom plus 6px; bottom is 88px plus safe-bottom. No connector separation or content obstruction. | B/M; D for real keyboard/safe-area geometry. |
| UI-05 P0 | Drag/zoom map through connector/decorative areas; attempt taps through panel content; show GPS and offline notices together. | Connectors/seam and offline notice pass pointers through; panels and controls accept intended input. GPS notice is dismissable, unlike passive offline notice. Itinerary wrapper transparency does not block otherwise exposed map regions. Record pointer target and map movement. | B/H/M; D touch confirmation. |
| UI-06 P1 | Activate the external Disney link in desktop, Safari and installed mode without completing transactions. | Exact configured link target remains unchanged; no panel state is incorrectly added. Record actual provider/device fallback; a native-app launch cannot be inferred from desktop navigation. | M/D; B can inspect href only. |

## 3. Place search

Source: [places-ui.js](../places-ui.js), `normalize`, `searchPlaces`, `phraseScore`, `tokenScore`, `editDistance`, `showPlace`, `closePanel`, `buildControl`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| SEARCH-01 P0 | Run recorded exact-name, alias, prefix, word-order, typo and empty-result queries; record ordered result IDs before/after extraction. | Preserve exact/prefix/contains/token/fuzzy scoring tiers, alias penalties and conservative typo eligibility. Area fallback must not outrank direct matches. No new remote search dependency; candidate IDs/order equal reference for fixed inputs. | B; S/H for pure-function comparison without changing source. |
| SEARCH-02 P0 | Apply each park and type filter, combined filters, empty query and no-match query; clear text using clear control. | Every result satisfies filters; count/empty message matches results; clear control restores the expected listing and input focus. Compare alphabetical/tie behavior under the recorded locale. | B/M. |
| SEARCH-03 P0 | Select recorded result, reopen search, select another, then tap map/control. | Existing highlight/tooltip and focus view appear, search closes, and previous highlight is removed according to capture-pointer policy. Do not silently unify itinerary/waits highlights. Compare current `showPlace` zoom and animation options and resulting view. | B/M. |
| SEARCH-04 P1 | Reject the directory request in a fresh context; separately supply invalid top-level/coordinate/park entries in a harness. | Directory-unavailable text appears for load failure; existing filters reject invalid place entries without breaking other independent features. After recovery/reload, search works; record current recovery mechanism, not an invented retry button. | H/B. |

## 4. Itinerary, day/session policy, stops, and walking paths

Source: [itinerary-ui.js](../itinerary-ui.js), `loadItinerary`, `todayInAnaheim`, `anaheimMinutes`, `schedulePeriods`, `selectDay`, `readSavedSession`, `restoreMapView`, `refreshDayIfNeeded`, `renderRoute`, `loadWalkingData`, `renderWalking`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| TRIP-01 P0 | Cold-load at a supported trip-day clock and at a recorded unsupported date; select every UI day and tab. | Actual supported Anaheim date wins initially; outside supported dates preview uses `dayIndex=1`. Six JSON days do not become six selectable days: four supported park/rest dates and three walking dates remain as coded. Banner, summary, selected tab/day and button availability agree. | H/B; M with recorded actual clock. |
| TRIP-02 P0 | At fixed times immediately before/during/after selected timeline periods, open schedule, scroll manually, then wait a refresh interval. | NOW/UP NEXT follow current `schedulePeriods`/Anaheim time interpretation; display time text is unchanged. Opening can scroll to current/next activity; subsequent periodic refresh does not reset manual scrolling. Other dates are not highlighted as today. | H/B; M supplementary. |
| TRIP-03 P0 | Toggle Stops only, Route only, both, neither; close all panels; click a numbered pin. | Stops render numbered destinations, not connecting lines; Route independently renders walking geometry with casing/park-hop dashing. Visibility survives panel closing. Pin number, popup place/time/activity and coordinate match the ordered route/timeline join. Explicit Stops enabling may fit bounds; restoration must not do so. | B/M. |
| TRIP-04 P0 | Seed valid same-Anaheim-day `dav2-map-session-v1`, pan/zoom/toggle, reload; repeat invalid coordinates/zoom, stale day, and saved day conflicting with actual supported day. | Only valid permitted state restores; view uses street bounds/zoom validation. Actual supported day overrides another saved day. Stored schema/identifier stays unchanged; corrupt/blocked storage cannot crash the map. Record exact restored values and writes. | H/B. |
| TRIP-05 P0 | Advance a controlled park clock across midnight, trigger visibility/pageshow or relevant day refresh. | `refreshDayIfNeeded` resets overlays, tab and itinerary highlight and returns map overview; selects actual supported day or preview policy. No stale previous-day paths or recursive persistence loop. GPS and other feature selection channels retain their own documented policies. | H/B; D background smoke. |
| TRIP-06 P1 | Delay walking data, enable Route, change day before completion; reject/repair walking data; fail initial trip JSON then retry. | No path for the wrong selected day; walking validation/error state disables/resets appropriate state and permits retry. Itinerary load uses one in-flight initialization, one 650ms reload-cache retry per failed file, and an operable Retry control after failure. | H/B. |
| TRIP-07 P0 | Check IDs/date joins against reference data without changing it. | 317 unique places, six data days, 40 ordered stops across three dates, 37 walking segments/1,295 points and 54 resolving `mapIds` match baseline. All route place IDs and zero-based `timelineIndex` resolve; one-based walking leg indices are adjacent/in range; geometry is finite lat/lon. Report schema diagnostics separately from runtime acceptance. | S. |

## 5. GPS and lifecycle

Source: [gps-ui.js](../gps-ui.js), `locateOrRecenter`, `startWatch`, `onPosition`, `onError`, `centerOn`, `clearWatch`, `deactivate`, visibility/page handlers; `gps-ui.css`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| GPS-01 P0 | Fresh-load without clicking GPS; click GPS and accept permission with an in-bounds fixture. Repeat real permission flow on phone. | No watch/request before user intent. After activation, one watch, correct pressed/label state, noninteractive dot and positive-accuracy circle; focus is at least zoom 19 without zooming out from a closer view. Record fixture versus real hardware. | H/B plus D permission/GPS. |
| GPS-02 P0 | Pan away after initial fix, deliver another fix, then click GPS again; use the separate resort recenter. | Normal updates do not become continuous forced following. GPS click recenters on a valid latest fix; resort recenter calls `focusInitialView(true)`. These controls retain separate meanings. | H/B/M; D smoke. |
| GPS-03 P0 | While tracking, hide/pagehide and return; repeat before requesting tracking. | Watch is cleared and graphics/last fix removed while hidden/page-exited; resume only for existing tracking intent. No extra watch or automatic tracking after a nontracking visit. GPS watch state is not explicitly persisted; separately observe map-view session writes after GPS-driven movement. | H/B for simulated lifecycle; D for real suspension/resume. |
| GPS-04 P0 | Deny permission; simulate timeout/unavailable and missing/insecure capability. | Denial deactivates tracking and reports status; timeout/unavailable follows existing notice policy without inventing a new permission flow. Missing secure capability fails gracefully while map remains usable. Notice dismisses by click or existing 3500ms timer. Unreproducible API faults are BLOCKED, not passed. | H/B; D OS prompts. |
| GPS-05 P1 | Deliver out-of-bounds, malformed, stale (>60s) and invalid-accuracy fixes; then a fresh in-bounds fix. | No focus outside artwork bounds; stale/malformed input is not rendered as a fresh valid fix. Out-of-bounds notice is not repeated for every update; valid recovery restores graphics. Inspect actual accuracy behavior and avoid assuming all sensor values exist. | H/B; D real accuracy observation. |

## 6. Live waits and network failures

Source: [wait-times.js](../wait-times.js), `snapshot`, `subscribe`, `refreshIfStale`, `refresh`, `queueInfo`, `badgeInfo`, `paint`, `updateVisibility`; `toolbar-ui.js:renderWaits/loadWaits`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| WAIT-01 P0 | Supply fixed successful feeds for both configured parks; open waits panel; zoom main map below/above 18. | One shared provider drives list and bubbles, matched by local names/aliases. List is alphabetic; bubbles require online state and zoom >=18. List selection closes panel and creates the existing noninteractive focus/tooltip. Compare payload, marker count and ordered IDs. | H/B. |
| WAIT-02 P0 | Reopen panel repeatedly within one minute, advance beyond threshold, trigger timer/visibility/online events, and hold one refresh pending. | `refreshIfStale`, timer and visibility behavior match source; pending guard prevents overlapping refresh. Online handler can request refresh independently of timer staleness. No additional per-panel polling loop. Immediate subscriber snapshot/unsubscribe behavior remains intact. | H/B. |
| WAIT-03 P0 | Fail one park, then both; recover; repeat with malformed `liveData`. | Successful park survives partial failure, errors=1 is partial; both failures give unavailable state. `latest` is replaced on each refresh: do not expect failed-park data to be retained as stale results. Recovery repaints both views without reload where existing handlers permit. | H/B. |
| WAIT-04 P0 | Trigger offline while list/bubbles are visible, then online. | Items/markers are cleared and offline/unavailable messaging is shown; panel remains operable and recovery refreshes through the shared provider. External live requests bypass the same-origin worker filter. | B/H/M. |
| WAIT-05 P1 | Supply OPERATING, DOWN, CLOSED, REFURBISHMENT, missing waits and waits at 0/20/21/45/46/600 and invalid values. | Existing labels/styles and 20/45 badge boundaries match; numeric waits outside 0..600 are not treated as valid. Popup single-rider/last-updated details and list badges remain consistent; unmatched names remain diagnostics rather than arbitrary matches. | H/B. |

## 7. PWA, service worker, tile download, and offline startup

Source: [sw.js](../sw.js), `SHELL_FILES`, install/activate/fetch handlers; [offline-ui.js](../offline-ui.js), `start`, `canAutoDownload`, `ensureManifest`, `syncMissingTiles`, `suspendDownload`; [manifest.webmanifest](../manifest.webmanifest).

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| PWA-01 P0 | Inspect manifest/Apple title; install and launch on supported desktop and actual iPhone; inspect registration. | ID `/dav2-development/`, short/Apple title `Disney V2 Dev`, relative `start_url`/`scope` `./`; worker resolves in current application directory. Record actual launch UI/identity/scope. Existing 512-declared/1254-actual icon mismatch is a known issue, not evidence that installation must succeed. | S/B metadata; M/D installation. |
| PWA-02 P0 | First online visit in isolated profile; await worker install/activation/controller readiness; inspect shell cache. | 28 shell entries precache, then install calls `skipWaiting`; activation cleans only other `dav2-shell-*` caches and claims clients. Tile cache remains `dav2-tiles-v1`. A first page may not yet be controlled; compare after readiness/reload. Never touch V1 data to test cleanup. | B/H. |
| PWA-03 P0 | Compare tile requests on miss/hit; inspect an ordinary resource online and on rejected network fetch; separately return HTTP 500. | Tiles are cache-first; other same-origin in-scope GETs are network-first. Thrown fetch failure can use cached fallback; HTTP error alone does not guarantee fallback. An uncached non-tile, nonnavigation resource can yield 503; an uncached tile's rejected fetch yields a network-error response instead. Cached-index navigation fallback is distinguishable. No cross-origin/out-of-scope request is cached by this worker. | H/B. |
| PWA-04 P0 | Allow automatic downloading from an empty tile cache; wait for completion; revisit warm. | Index is 4,602 unique files/174,042,995 bytes. Up to six downloader workers fetch only missing URL keys; bulk header bypasses worker duplicate writes. Complete warm inventory causes no repeated full download/banner. Aggregate cache count/path matching, not raw disk size alone, establishes completion. | S index checks; B/H cache/network. |
| PWA-05 P0 | Hide during download, reopen; interrupt network mid-run and restore; repeat save-data/cellular hints where exposed. | Hiding aborts page download and later checks resume missing entries. Offline prevents starting; active network loss can cause failures rather than immediate abort. Exposed save-data/cellular hints suppress automatic start; Safari connection-type detection is not guaranteed. No corrupt response is counted as a saved successful tile. | B/H; D Safari/background. |
| PWA-06 P1 | Deny persistence or reject cache writes with a controlled quota failure; recover space in the isolated test context only. | Persistence denial does not prevent best-effort caching. Downloader stops/reports incomplete storage-full status; online tiles can still render when worker cache writes fail. No origin-wide deletion or silent removal of drafts. Record actual eviction behavior only if reproduced. | H/B; D retention observation. |
| PWA-07 P0 | After full download, go offline and reload main; inspect search, itinerary/stops/walking and cache hits. Repeat first offline navigation to unvisited editor in a fresh main-precache context. | Cached main/assets/data/tiles support main offline functions; waits cannot refresh. Editor is not in shell precache and registers no worker itself: fresh offline editor may fall back to main, a known coverage limitation. Do not mark it as supported offline editing. | B/M; D installed offline launch. |
| PWA-08 P1 | In an approved isolated secure environment, update an existing installed baseline to a candidate; compare a fresh install separately. | Same intended development identity/scope, coherent code/assets, retained tile cache and valid drafts. Current eager worker takeover can mix old pages/new worker; record problems, do not assume update atomicity. Root-path portability must be tested on DAv2-owned test paths, never by deploying to V1. | B/H partial; D actual installed upgrade. |

## 8. Route editor and data integrity

Source: [route-editor.js](../route-editor.js), `restoreDraft`, `saveDraft`, `modifyLeg`, `ptsForLeg`, `drawSolid`, `exportRoute`, import/reset handlers; `route-editor.html`, `route-editor.css`.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| EDIT-01 P0 | Cold-load editor without draft; inspect each supported day/leg; compare main map profile. | Separate map initializes with its own zoom controls, initial view, padded artwork bounds and disabled double-click zoom. Places/route/itinerary joins resolve; existing walking traces are not auto-imported. Each supported date needs at least two valid stops. | B/M; S joins. |
| EDIT-02 P0 | Select recorded leg, draw bends in order; switch refine mode, insert/drag/remove a bend, undo/redo, clear leg, finish it. | Stop endpoints stay anchored; current geometry/order, completion state and button availability match reference. Each recorded editing action can be inspected/exported consistently; undo/redo restores expected geometry/history state. Map drags do not accidentally insert vertices. | B/M; D touch accuracy. |
| EDIT-03 P0 | Save draft, switch day/leg, reload; seed malformed storage, invalid points, mismatched leg counts and out-of-range selected leg. | `dav2-route-editor-v1` restores only accepted structure/points and clamps selected leg. Blocked/full storage reports failure without rewriting repository files. Draft remains separate from main session; undo history is not presumed persisted. | H/B. |
| EDIT-04 P0 | Finish one leg while leaving another incomplete; export; inspect JSON without replacing repository data. | Only finished legs export, with one-based consecutive `legFrom/legTo`, full endpoint-plus-via geometry and correct date. Zero completed legs produces a status instead of a download. Source JSON files remain unchanged. | B/M plus S/H exported-data validation. |
| EDIT-05 P0 | Import compatible fixture, first cancel then confirm; change only its endpoint coordinates; test invalid/nonadjacent legs and malformed JSON. | Cancel preserves draft. Confirm replaces it; import stores interior points and reanchors to current stop coordinates through `ptsForLeg`. Invalid legs are skipped according to existing guards; incompatible/invalid file reports status. No silent acceptance of arbitrary old route formats. | H/B/M. |
| EDIT-06 P1 | Trigger reset, cancel, then confirm in a disposable draft; toggle show-all/fit-leg and collapse around 759/760/761px. | Cancel keeps edits; confirmed reset blanks editor work without changing project JSON. Existing undo/reset behavior, selected day/leg, completed-layer visibility and map sizing match reference. Panel/help hit testing remains usable. | B/M; D narrow touch. |
| EDIT-07 P1 | Fail/malformed-load each startup JSON input in fresh editor context. | Existing startup failure reports error and disables editor controls as coded; no wrong-day/orphan geometry or partial edits silently saved as valid. Recovery expectations follow current reload flow, not a new retry UI. | H/B. |

## 9. Responsive, orientation, keyboard, and platform coverage

Source: `index.html:focusInitialView/configureNavigationBounds`, `toolbar-ui.js:updateExpandedPanelTop/positionConnector`, `toolbar-ui.css`, `itinerary-ui.css`, `places-ui.css`, `route-editor.css`; matrix above.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| RESP-01 P0 | Run main/panels/editor screenshots at representative and applicable boundary widths; inspect rectangles and click targets. | No regression in sizing/wrapping, visible map controls, panel content scroll, connector alignment or hidden-state behavior. Preserve current breakpoint results; horizontal insets/hinges not handled by source are recorded issues rather than invented support. | B/M. |
| RESP-02 P0 | On iPhone Safari, open search keyboard, type/filter/clear, rotate, collapse/expand browser chrome, and pan after closing UI. | Real safe-area, visual viewport, scroll and pointer behavior remains usable and matches captured reference. Record panel top/bottom, input visibility, focus and map viewport; desktop emulation does not establish this pass. | D. |
| RESP-03 P0 | Repeat principal gestures/panels on installed iPhone PWA, background/foreground and offline reopen. | Standalone launch, safe areas, keyboard, lifecycle, retained caches/drafts and touch behavior match its own baseline. Do not assume Safari and standalone share storage. | D. |
| RESP-04 P1 | Use actual tablet/split view and foldable folded/unfolded/hinge layouts, portrait/landscape. | No new obscured controls, broken layout or gesture conflict relative to the same environment's reference. Identify unsupported existing conditions and record emulator-only observations separately. | D or identified emulation B/M; physical behavior still unverified without hardware. |

## 10. Accessibility, performance, startup/network, and console

Source: UI `aria-*`/roles and keyboard handlers in `places-ui.js`, `itinerary-ui.js`, `toolbar-ui.js`, `gps-ui.js`; CSS focus/motion/transparency rules; lifecycle/network code cited above.

| ID / priority | Steps | Expected behavior and observable pass/fail criteria | Method |
| --- | --- | --- | --- |
| QUAL-01 P0 | Navigate visible controls with keyboard, activate using expected keys, close panels with Escape, inspect focus/labels/states. | Accessible names, visible focus, pressed/expanded state and documented focus return match reference; hidden controls are not unintended reachable targets. Search is a plain panel; itinerary/waits declare dialog but do not implement blocking focus traps. Record baseline accessibility defects rather than asserting full compliance. | B/M; D screen-reader follow-up. |
| QUAL-02 P1 | Test screen reader announcements, text enlargement, high zoom, reduced motion/transparency and contrast measurement where tools exist. | Labels/status announcements and readable controls remain available; no new clipping or unexpected focus loss. Offline notice's existing reduced-motion handling is preserved; do not assume all map animations already honor it. Contrast/other baseline gaps require explicit issue review. | M/D; B static/style checks supplementary. |
| QUAL-03 P0 | Capture console/page errors, failed requests and worker errors throughout all P0 flows, distinguishing normal and injected failures. | Normal run has no new unhandled errors, missing assets or duplicate initialization. Fault cases show expected scoped messages without unrelated crashes. Record exact normalized error signatures and associated case; do not suppress all warnings or expose credentials in artifacts. | B/M; S syntax/path checks. |
| QUAL-04 P0 | Repeat at least three cold and three warm startup runs under identical machine/browser/throttling conditions. | Record timing to visible map and usable controls, request count/bytes, tile-cache hits and failure count; compare median/range with reference. Do not invent a latency budget or mix first full-tile downloading with a warm-load measurement. No unexplained new request storm or runtime dependency on remote Leaflet/V1. | B performance/network capture; M timings if instrumentation unavailable. |
| QUAL-05 P1 | Repeatedly switch panels, select results, toggle overlays and background/foreground; observe while tile downloading is active. | No new accumulating layers/controls, duplicate watches/subscriptions/polling, sustained responsiveness regression or obvious memory growth versus equivalent reference cycles. Record cycle count and available measurements; unavailable heap/listener metrics are unmeasured. | B/H/M; D responsiveness. |
| QUAL-06 P0 | Parse all application JSON and JS syntax; resolve HTML/CSS/manifest/worker asset references with exact casing; inspect baseline/candidate file diff. | No missing required resources or unintended data/artwork edits. Static checks do not certify inline/runtime behavior by themselves. Current expected inventory includes 22 HTML references and 28 worker shell entries; report only aggregate tile results. | S, existing Node/PowerShell/Git tools. |

## Evidence record and pre-refactor gate

Use one run header and one compact case record per execution. Preserve evidence from both reference and candidate; all proposed cases remain NOT RUN until executed.

```text
Run ID / operator / execution timestamp:
Reference commit: 4de6524820c9285480c4ee26306b549c5b76fa44
Candidate commit or source fingerprint / working-tree state:
Environment: OS; browser/version; device model; actual hardware or emulation:
Entry URL / origin / Safari or installed-PWA launch mode:
Viewport CSS width x height / DPR / zoom / orientation / posture:
Locale/timezone / Anaheim clock or controlled clock input:
Network/CPU throttle / API fixture IDs / geolocation or permission setup:
Storage seed / selected day-tab / map view / overlay state:
Worker script-controller/cache names / cold-warm/completeness state:

Case ID / priority / method S,B,H,M,D:
Exact fixture inputs and numbered steps actually executed:
Expected source behavior / reference observation:
Actual candidate observation:
Result: PASS | FAIL | NOT RUN | BLOCKED
Screenshots: reference path; candidate path; comparison/rectangle evidence:
Console/network trace or exported-data evidence:
Issue ID: new regression or known baseline limitation; severity; reproduction:
Blocker/coverage gap / approved deviation and reviewer:
```

Before the first code refactor, require recorded P0 static/desktop outcomes, deterministic fixture inputs, representative geometry/screenshots, the main offline-cache smoke, editor round trip and actual iPhone Safari/installed-PWA coverage for affected behavior. An unavailable device is an explicit reviewed gap, not a green checkbox. Carry known icon-size, first-offline-editor and other reference issues into the issue register. Do not require incidental fixes in the characterization task.

Review failures and unexecuted cases before approving a migration slice. Re-run its affected P0/P1 cases and shared startup/offline checks after each small change. This plan neither authorizes running tests that create artifacts beyond approved locations nor permits commits, pushes, deployments, dependencies, Git setting changes, maintenance scripts, V1 access or application modifications.
