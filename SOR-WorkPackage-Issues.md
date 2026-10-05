# Werkpakket (Workpackage) tool — issue report

**Date:** 2026-10-05  
**Scope:** SOR Werkpakket tool plus the shared Functie informatie (feature-info) flow it relies on.  
**Environment:** local dev — SOR `http://localhost:8151`, BGT module `http://localhost:8086/build/js/BGT.js`, BGT Editor `http://localhost:9191`.  
**Method:** code inspection (primary source); issues 2 and 3 were originally observed via Chrome DevTools network throttling (2G) and Offline mode.

---

## Summary

| # | Issue | Area | Impact | Where it lives |
| - | ----- | ---- | ------ | -------------- |
| 1 | Opening the Gegevens browser while Werkpakket is active leaves the tool button pressed (“actief”) although the workpackage mode has been switched off | Toolbar / tool state | Looks active but is dead; menu can be left on the map; takes an extra click to recover | `MainLayout.js` `openChalo` click handler + `Util.ClearAllObjectEvents` / `Util.CancelActiveTool` |
| 2 | No feedback while a response is in flight for Werkpakket or Functie informatie (visible under 2G) | UX | User cannot tell whether the click registered; invites repeat clicking | OneMap emits `featureInfo.requested`; SOR only listens to `featureInfo.received` |
| 3 | Functie informatie silently ignores failed / empty responses (reproduced with DevTools Offline) | Error handling | Dead click, no error, no timeout; in Werkpakket mode it even misreports “Werkpakket niet gevonden.” | `Common.js` `FeatureInfoReceived` ignores `evt.errors`; no watchdog anywhere |

---

## Call chain at a glance

```text
[sorToolbar] workPackage  (MainLayout.js:2087-2123)
   │ toggle ON
   ▼
SOR.Global.VIEWER.WorkPackage_MODE_ENABLED = true        (Globals.js:307)
   + Util.EnableFeatureInfoMode()                        (Utilities.js:3529)
   │ WMS GetFeatureInfo click on workpackage layer        ◄── Issue 2: nothing shown while in flight
   ▼
Common.MapEvents.FeatureInfoReceived                      (Common.js:749-774)
   │ id + status via Common.GetFeaturePropertyValue       (Common.js:1779)
   ▼
Util.PopulateWorkPackageMenu                              (Utilities.js:2851-2897)
   │ #WorkPackageMenu with 6 items
   ▼
WpkManager                                                (WpkManager.js)
   ├─ CheckoutWorkpackage  ──► HIT.BGT.CheckoutAOI             ──┐
   ├─ ExportWorkPackage    ──► HIT.BGT.ExportWorkPackage       ──┤
   ├─ DownloadLVResponse   ──► HIT.BGT.DownloadLVResponseFile  ──┤ BGT module
   ├─ DeleteWorkPackage    ──► HIT.BGT.DeleteWpk               ──┤ (localhost:8086)
   ├─ AbortWorkPackage     ──► HIT.BGT.AbortWpk                ──┘
   └─ OpenWpkEditSession   ──► BGTEditor.UI.OpenEditSession    ─── BGT Editor (localhost:9191)
   │ callback
   ▼
Util.RemoveWorkPackageDivFromMap + Util.AfterWorkPackageOp   (Utilities.js:3722-3727)
```

Issue placement:

- **Issue 1** happens *before* this chain — **Openen Gegevens browser** clears the mode flags but leaves the `workPackage` toolbar toggle pressed.
- **Issue 2** sits between the map click and `FeatureInfoReceived` (the fetch), and to a lesser degree on the module calls after the menu.
- **Issue 3** sits inside `FeatureInfoReceived`: on a failed fetch, `evt.errors` is populated and `evt.results` is empty, and the handler ignores both.

---

<details>
<summary><strong>Issue 1 — Gegevens browser leaves Werkpakket shown as active</strong></summary>

### Issue details

### Steps to reproduce

1. Enable **Werkpakket** on the map toolbar (button pressed, workpackage click works).
2. Click **Openen Gegevens browser**.
3. Observe the Werkpakket button, then click it once (and map-click once).

### Expected

Workpackage mode is switched off, the button is released, the map is back to neutral/pan behaviour, and a single click reactivates the tool.

### Actual

The button stays pressed (shown as active) while the mode flag is off, so map clicks no longer open the workpackage menu. Clicking the button once only clears the stale pressed state; a second click is needed to reactivate. If the workpackage menu/div was open, it can stay behind on the map.

### Evidence

- `openChalo` click handler (MainLayout.js:2032) runs `Util.ClearAllObjectEvents()` and `Util.CancelActiveTool()`, then opens the CHALOIS popup.
- `Util.ClearAllObjectEvents()` (Utilities.js:896) resets `SOR.Global.VIEWER.WorkPackage_MODE_ENABLED` (Utilities.js:955-957) but never touches the ExtJS toggle `workPackage` and never removes `#workpackageDivOnMap_Parent` / `#WorkPackageMenu`.
- `Util.CancelActiveTool()` (Utilities.js:2753) releases a toolbar toggle only via `SOR.Global.VIEWER.SelectedTool.id` (Utilities.js:2774-2779). The `workPackage` toggle handler (MainLayout.js:2103-2121) never registers itself via `Util.SetSelectedToolObject()` — compare `btnObliquoPan` (MainLayout.js:2002-2004) — so `SelectedTool.id` is empty and there is nothing to release.
- Precedent for the fix already exists: `WpkManager.CloseWpkEditSession` explicitly releases this toggle for the same class of bug (WpkManager.js:82-85).

### Suggested fix

- **Minimal (covers this repro):** in the `openChalo` click handler, after `Util.CancelActiveTool()`, release the toggle explicitly:

  ```js
  var workPackageButton = Ext.getCmp('workPackage');
  if (workPackageButton && workPackageButton.pressed) {
      workPackageButton.toggle(false);
  }
  ```

  This runs the existing “off” branch (MainLayout.js:2113-2119), which also removes the map div/menu and returns to pan mode.

- **Structural:** have the `workPackage` toggle register itself with `Util.SetSelectedToolObject(ctrl)` on activation (as other tools do), so `CancelActiveTool()` handles it generically; optionally let `ClearAllObjectEvents()` clean the workpackage UI elements too, so the flag and the UI cannot drift apart.

</details>

---

<details>
<summary><strong>Issue 2 — No loader while a response is in flight (Werkpakket + Functie informatie)</strong></summary>

### Issue details

### Steps to reproduce

1. DevTools → Network → throttling **2G**.
2. Enable **Werkpakket** (or plain **Functie informatie**) and click the map.

### Expected

Visible progress feedback from the click until the result arrives (spinner / mask / busy state), so the user knows the click is being processed.

### Actual

Nothing changes during the whole wait; the click looks ignored and invites repeated clicks.

### Evidence

- The map-click feature-info fetch is the first thing both tools wait on. SOR subscribes only to `featureInfo.received` (map-onemap.js:351-353). OneMap emits `featureInfo.requested` when the query is issued, and `featureInfo.received` when it finishes (verified inside the loaded bundle `Scripts/Onemap/lib/js/onemap-viewer.js`, loaded from Default.aspx:319). No SOR code listens to `featureInfo.requested` (grep shows no other subscription).
- For the workpackage actions after the menu, the BGT module does show its own mask on several calls (`HIT.BGT.MASK`: `GetWorkpackageData` bgt-init.js:117-119, `CheckoutWorkPackage` CreateAOIUtility.js:414, `AbortWpk` :360, `DownloadLVResponseFile` bgt-init.js:488) — but this is applied ad hoc per call, and the initial map-click fetch that opens the menu is not covered at all.

### Suggested fix

- In `map-onemap.js`, next to the existing `featureInfo.received` subscription, listen to `featureInfo.requested` and show a lightweight loader; hide it on `featureInfo.received` (plus the watchdog from Issue 3). One change covers Werkpakket and Functie informatie for the map-click fetch.
- Standardise the `WpkManager.*` → module action calls so every pending request has a visible indicator, instead of relying on each module call remembering to call `HIT.BGT.MASK.show()`.

</details>

---

<details>
<summary><strong>Issue 3 — Functie informatie ignores failed / empty responses</strong></summary>

### Issue details

### Steps to reproduce

1. DevTools → Network → **Offline**.
2. Enable **Functie informatie** and click the map (also reproducible with a stalled — never answering — request).

### Expected

A clear error or timeout message, and the UI returns to a usable state.

### Actual

Complete silence: no message, no cleanup. In Werkpakket mode the same failure is reported as “Werkpakket niet gevonden.”, which is misleading.

### Evidence

- On failure OneMap does not drop the click: each layer query is caught into `{ ok: false, error }`, and the bundle then emits `featureInfo.received` with `{ results: [], errors: [{ layerId, message }] }` (verified in the bundle; the legitimate “no features found” case is emitted as `null`).
- `Common.MapEvents.FeatureInfoReceived` (Common.js:670) only handles `null` (early return, Common.js:671-678) and usable results. It never looks at `evt.errors` and never treats `results.length === 0` as a failure, so in plain feature-info mode nothing is shown.
- In Werkpakket mode the failure falls into the workpackage branch (Common.js:749-774) and is misreported as a missing workpackage.
- A truly stalled request emits no `featureInfo.received` at all, and there is no timeout/watchdog anywhere, so the click stays dead indefinitely.

### Suggested fix

- In `FeatureInfoReceived`, distinguish “no result” from “request failed”: check `evt.errors` (non-empty) and/or empty `evt.results`, and surface a message via `messageBox.show(...)` instead of silence; keep the current workpackage-specific text for the genuine no-feature case.
- Add a watchdog when `featureInfo.requested` fires (from the Issue 2 fix): if no `featureInfo.received` arrives within N seconds, hide the loader and inform the user.

</details>

---

## Notes / open points

- The `featureInfo.*` events are emitted by the vendored OneMap bundle (`Scripts/Onemap/lib/...`). A hard failure event or a proper request timeout would be a OneMap-side change (the click query in the bundle has no timeout). The suggested SOR-side fixes above do not depend on that.
- BGT module `HIT.BGT.Util.GetJsonData` hides its mask on failure, but its default error message uses `Ext.decode(response.responseText).Message` (BGT module `Scripts/Utilities.js:113`), which can itself fail when the response body is empty (e.g. offline) — a failed workpackage call may therefore show no message even though the mask clears. Worth checking while fixing the feedback gaps.
- Line numbers verified against the code as of 2026-10-05 and will drift as files change.
