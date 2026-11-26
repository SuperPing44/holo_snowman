# How one web app drives a drawing window and a fullscreen projector window

Research for issue #2 (map: #1). Researched 2026-11-26 against primary sources (W3C spec, MDN, Chrome/Chromium, Microsoft Learn, Wacom developer docs). Claims marked **[unverified]** come from no primary source and must be tested on the event PC.

## Question

How can a single TypeScript browser app on one Windows PC run a drawing window on the monitor and a fullscreen projector window on the second display, synced live? This covers window-to-window sync, placing a fullscreen window on the second display, auto-start on Windows, and Wacom Intuos pen pressure via Pointer Events in Chrome/Edge.

## Recommendation (one approach)

**One Chromium browser instance (Edge is preinstalled on Windows; Chrome works the same way) loads the app from a small local server on `http://localhost`. The drawing window opens the projector window itself with the Window Management API and makes it fullscreen on the second screen. The two windows sync over a `BroadcastChannel`. Two enterprise policies set in the registry auto-grant window management and gesture-free fullscreen for the localhost origin. A shortcut in the Windows Startup folder launches the browser in app mode at sign-in. The Wacom driver keeps "Use Windows Ink" on, so pressure arrives as `PointerEvent.pressure` with `pointerType === "pen"`.**

Startup sequence:

1. Auto-login, then the Startup-folder shortcut runs a `start.cmd`. The script starts the local server, then runs
   `msedge.exe --app=http://localhost:5173/draw --no-first-run --user-data-dir=C:\holo\profile`
   (`--app` = "launched in 'application' mode", no browser UI; `--no-first-run`; a dedicated `--user-data-dir` keeps the kiosk profile separate from normal browsing).
2. The draw page calls `getScreenDetails()`, picks the screen that is not `isPrimary` (or the one matching the projector resolution), and calls `window.open('/projector', 'projector', 'popup,left=…,top=…,width=…,height=…')` with coordinates in multi-screen space.
3. The projector page calls `document.documentElement.requestFullscreen({ screen })`. The draw page also makes itself fullscreen on the primary screen.
4. If a policy is missing or not applied, the same steps run from a hidden operator button ("Start projector"), which supplies the user gesture. The operator is usually nearby (map #1).

### Limits

- **Policies are per browser.** Set them under `HKLM\SOFTWARE\Policies\Microsoft\Edge\…` for Edge and `…\Google\Chrome\…` for Chrome. Without them, the first run shows a permission prompt and every fullscreen entry needs a click.
- **Not a locked-down kiosk.** `--kiosk` / Edge's `--edge-kiosk-type=fullscreen` is a one-window, InPrivate signage mode. Popups and `localStorage` persistence in it are untested, so app mode plus page fullscreen is the safer route **[unverified: test `--kiosk` with a popup if a harder lockdown is wanted]**. Visitors could still reach Alt+F4, the Windows key and so on. That matches "semi-kiosk, operator nearby".
- **Crash recovery is manual-ish.** If the browser dies, nothing restarts it unless the start script loops or a scheduled task restarts it. Edge's idle timeout explicitly does not restart Edge. If only the projector window closes, the draw window can detect it (`popup.closed`) and reopen it. Re-opening fullscreen without a gesture again relies on the policy.
- **Display topology must be stable.** Windows must be in "Extend" mode, and the projector must be connected before the browser starts (or handle `screenschange`).
- **Pen pressure depends on the Windows Ink path.** Wacom's own docs never mention the "Use Windows Ink" checkbox. The claim that Chromium has no WinTab path, so pressure needs Windows Ink, comes from bug reports, not a vendor doc. Verify on the real tablet with a 10-line test page.
- **BroadcastChannel latency has no published figure.** It is in-browser IPC with structured clone, expected well under one 60 Hz frame for small stroke messages. Measure it (see below).

## Findings

### 1. Window-to-window sync

| Option | Fits? | Notes |
|---|---|---|
| **BroadcastChannel** | **Yes (recommended)** | "allows basic communication between browsing contexts (that is, windows, tabs, frames, or iframes) and workers on the same origin". Join by name, no references needed, messages are structured-cloned, Baseline since March 2022 [MDN]. Both windows must share origin **and storage partition**, so they must run in the **same browser profile** [MDN]. A reloaded window simply re-joins by name. |
| `window.open` + `postMessage` | Yes, alternative | The draw window holds the popup's `WindowProxy`, so it can `postMessage` directly. After a reload of the draw window the reference is lost, which makes BroadcastChannel more robust for crash recovery. |
| SharedWorker | Possible, not needed | Useful only if one authoritative state should live outside both windows. It adds complexity. The Design state can live in the draw window, with the projector as a pure renderer. |
| Local WebSocket server | Only if the windows are separate profiles/processes | Needed when two independent browser instances are launched (e.g. two `--user-data-dir`s to get `--window-position` to work). It adds a network hop and a process to supervise. |

**Message design for live strokes.** Send incremental Stroke points (e.g. batches per `pointermove`/`requestAnimationFrame`: `{strokeId, points:[x,y,pressure]…}`) plus coarse events (undo, clear, sticker transform). On projector (re)connect, send a full snapshot of the current Design. Coordinates should be in template space; the projector applies the per-Surface calibration.

**Latency.** No primary source publishes BroadcastChannel latency. Measure it on the event PC by sending `performance.timeOrigin + performance.now()` and diffing in the receiver; both windows share a clock via `timeOrigin`. End-to-end, the projector's own frame and input lag will dominate.

### 2. Placing a fullscreen window on the second display

- **API:** `window.getScreenDetails()` resolves to a `ScreenDetailed` whose `screens` lists every display. It is available "from Chrome 100" [Chrome docs]. Permission name: `"window-management"` (formerly `"window-placement"`) [W3C spec §3.7, Chrome docs].
- **window.open coordinates:** "open() steps must interpret 'left' and 'top' feature values to be specified relative to the multi-screen origin" [W3C spec §3.2.7].
- **Fullscreen on a chosen screen:** `requestFullscreen({ screen })` via the new `screen` member of `FullscreenOptions` [Chrome docs; W3C spec §3.5].
- **User activation:** "Transient user activation is typically already required for requestFullscreen() and open()" [W3C spec §4]. The spec's pattern for "a compelling multi-screen experience from a single user activation" is to go fullscreen on one screen and then open a popup on another in the same listener. The UA *may* waive the activation needed for that `open()` [W3C spec §1.2.6, §3.2.8].
- **Removing the gestures for an unattended start (enterprise policies):**
  - `WindowManagementAllowedForUrls` "automatically grant[s] the window management permission". Edge ≥ 123. Registry: `SOFTWARE\Policies\Microsoft\Edge\WindowManagementAllowedForUrls\1 = http://localhost:5173` [Microsoft Learn].
  - `AutomaticFullscreenAllowedForUrls` "allows matching origins to call [requestFullscreen] without a prior user gesture". Edge ≥ 132; Chrome shipped it on desktop in 126 [Microsoft Learn; Chromium intent-to-ship]. It combines with Window Management for "opening a popup on another display and making it fullscreen" [chromeos.dev]. Check with `navigator.permissions.query({name:'fullscreen', allowWithoutGesture:true})` (Chrome 128+) [chromeos.dev].
  - Chrome uses the same policy names under `SOFTWARE\Policies\Google\Chrome\…`.
- **"Fullscreen popups"** (`window.open(..., 'fullscreen')`) was an origin trial in Chrome 119–122 [Chrome blog]. Its later status is unclear, so do not rely on it.
- **Command-line alternative (not recommended):** Chromium has `--window-position=x,y`, `--window-size=w,h`, `--app`, `--kiosk` ("not Chrome OS kiosk mode") [chrome_switches.h]. Launching a second window this way into an already running instance reportedly ignores the position flags. Separate `--user-data-dir`s fix that but break BroadcastChannel/localStorage sharing **[unverified, forum reports]**, which forces a WebSocket relay.
- **Edge kiosk mode:** `msedge.exe --kiosk URL --edge-kiosk-type=fullscreen` gives a single-site, InPrivate fullscreen session. It blocks F11/F12 and, in assigned access, Ctrl+N. `--kiosk-idle-timeout-minutes` closes the browser but "will not restart Microsoft Edge after it's closed" [Microsoft Learn]. InPrivate means no persisted storage, so autosave must go to the local server or a file anyway.

### 3. Auto-start on Windows

- The Startup folder (`shell:startup`) holds shortcuts to "applications that start automatically when a user signs in" [Microsoft Support]. Put a shortcut to `start.cmd` there, with Windows auto-sign-in on a dedicated local account.
- `start.cmd` starts the local server (it serves the app; it can also write calibration, autosave and Design images to disk, which a browser alone cannot do silently). It then launches the browser in `--app` mode. A `:loop` that relaunches the browser when it exits gives basic crash recovery.
- A local server is needed anyway. ES modules and BroadcastChannel do not work well from `file://` (opaque origin) **[unverified for BroadcastChannel; ES-module CORS on file:// is standard browser behaviour]**, and the policies match an origin.

### 4. Wacom Intuos pressure via Pointer Events (Chrome/Edge, Windows)

- `PointerEvent.pressure` is "normalized ... in the range of 0 to 1". Hardware without pressure reports 0.5 while buttons are down [MDN]. `tiltX`, `tiltY` and `altitudeAngle` are also on `PointerEvent`.
- Wacom: HTML5 Pointer Events support "X, Y location, Pressure, X, Y tilt" and rotation and buttons, and "All Wacom tablets supported by the Wacom driver are supported" [Wacom Web API overview]. Microsoft Ink APIs "are used on the back end" for "HTML5 Pointer Events (Web API) for web browser apps" [Wacom Windows Ink overview].
- **Driver setting:** keep **"Use Windows Ink" enabled** (Wacom Tablet Properties → Mapping, global or per-app for the browser). Bug reports say that with it off the pen shows up as `pointerType: "mouse"` and Chromium gets no pressure/tilt, because Chromium does not use WinTab **[unverified: Mozilla bug reports, not vendor docs]**.
- App side: use `pointerdown/move/up`, `setPointerCapture`, `touch-action: none` on the canvas, and `getCoalescedEvents()` for smooth fast strokes. Touch (`pointerType: "touch"`) uses the same code path with a constant pressure.

## Sources

- W3C Window Management spec: https://w3c.github.io/window-management/
- Chrome docs, Window Management API: https://developer.chrome.com/docs/capabilities/web-apis/window-management
- Chrome blog, Fullscreen popups origin trial: https://developer.chrome.com/blog/fullscreen-popups-origin-trial
- chromeos.dev, Using the Fullscreen API without gestures: https://chromeos.dev/en/posts/using-the-fullscreen-api-without-gestures
- Chromium intent to ship, Automatic Fullscreen Content Setting: https://groups.google.com/a/chromium.org/g/blink-dev/c/WOch5LPq9RY
- Chromium `chrome_switches.h`: https://github.com/chromium/chromium/blob/main/chrome/common/chrome_switches.h
- Chrome docs, Chrome flags / user data dir: https://developer.chrome.com/docs/web-platform/chrome-flags
- Microsoft Learn, WindowManagementAllowedForUrls: https://learn.microsoft.com/en-us/deployedge/microsoft-edge-browser-policies/windowmanagementallowedforurls
- Microsoft Learn, AutomaticFullscreenAllowedForUrls: https://learn.microsoft.com/en-us/deployedge/microsoft-edge-policies/automaticfullscreenallowedforurls
- Microsoft Learn, Configure Edge kiosk mode: https://learn.microsoft.com/en-us/deployedge/microsoft-edge-configure-kiosk-mode
- Microsoft Support, Configure startup applications: https://support.microsoft.com/en-us/windows/configure-startup-applications-in-windows-115a420a-0bff-4a6f-90e0-1934c844e473
- MDN, Broadcast Channel API: https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API
- MDN, PointerEvent.pressure: https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/pressure
- Wacom, Web API overview: https://developer-docs.wacom.com/docs/icbt/web/web-api-overview/
- Wacom, Windows Ink overview: https://developer-docs.wacom.com/docs/icbt/windows/windows-ink/windows-ink-overview/
- Mozilla bug 1447580 (Windows Ink off → pointerType mouse): https://bugzilla.mozilla.org/show_bug.cgi?id=1447580
