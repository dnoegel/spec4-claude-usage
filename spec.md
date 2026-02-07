# spec4-claude-usage

> **What is this?** This is a *spec4* spec — a human-readable specification that, when given to an AI coding assistant with terminal access (Claude Code, Cursor, Aider, etc.), builds a complete macOS menu bar app for tracking your Claude AI usage limits. No code to trust — just a spec that builds the app on your machine. See the [README](README.md) for more about spec4.

## For Humans — Read Before Running

This spec instructs an AI to build a macOS menu bar app. Here's exactly what it does:

**What the app does:**
- Shows your Claude.ai usage limits (5-hour session, 7-day rolling) in the macOS menu bar
- Fetches data from `https://claude.ai/api/organizations/{org_id}/usage` using your session cookie
- Refreshes on a configurable interval (default: 5 minutes)

**What credentials it needs:**
- Your Claude.ai session key (`sessionKey` cookie from claude.ai)
- The organization ID is auto-fetched after you provide the session key

**Where credentials are stored:**
- In `UserDefaults` (plaintext plist at `~/Library/Preferences/com.spec4.claudeusage.plist`)
- Not encrypted. Acceptable for a personal machine; be aware if others have access to your account.

**What it does NOT do:**
- No analytics, telemetry, or phone-home
- No network calls except to `claude.ai`
- No data leaves your machine
- No external dependencies — pure Swift/SwiftUI using only Apple frameworks

**Requirements:**
- macOS 14.0 (Sonoma) or later
- Xcode Command Line Tools (`xcode-select --install`)

**How to use:** Copy this entire file into your AI coding tool (Claude Code, Cursor, etc.) and let it run.

---

## Instructions for the AI

You are building a native macOS menu bar application called **ClaudeUsage**. It shows Claude AI usage limits (5-hour session, 7-day rolling) directly in the macOS menu bar.

**IMPORTANT: Do not ask the user any questions. Detect the system, build everything, and launch the app automatically.**

---

### Phase 1: System Detection

Run these checks silently. If any check fails, print a clear error and stop.

```
1. Verify macOS 14.0+ :  sw_vers -productVersion
2. Detect architecture :  uname -m          (arm64 = Apple Silicon, x86_64 = Intel)
3. Verify Swift 5.9+  :  swift --version
4. If swift is missing :  Print "Please install Xcode Command Line Tools: xcode-select --install" and STOP.
```

---

### Phase 2: Create Project

Create a Swift Package executable project called `ClaudeUsage` in the current working directory.

- Use Swift Package Manager with an executable target
- Target macOS 14+, Swift 5.9+
- **Zero external dependencies** — only Apple system frameworks (SwiftUI, AppKit, Foundation, Combine)
- No `main.swift` — use `@main` on the SwiftUI `App` struct
- Target macOS 14+ allows using `SettingsLink`, `@Observable`, and `Window` scenes — prefer these modern APIs

---

### Phase 3: Implement the App

Build the app according to this specification. Write clean, idiomatic Swift/SwiftUI code. Use separate files for logical components.

#### 3.1 — App Entry Point

Use SwiftUI's `App` protocol with:
- A `MenuBarExtra` scene (`.menuBarExtraStyle(.window)`) — this is the menu bar item + popover
- A `Settings` scene — this is the preferences window
- A single shared `UsageService` as `@StateObject` that is passed to both scenes

The menu bar label is a `@ViewBuilder` — it must render **SwiftUI views**, not just text. Depending on the display style, it shows different views (see section 3.3 for details). Always include a small SF Symbol icon (`chart.bar.fill`) as a leading element.

**IMPORTANT menu bar styling:** The label MUST look like a native macOS menu bar item. Do NOT wrap it in a background color, pill shape, or rounded rectangle container. The background must stay transparent — the system handles highlighting on hover/click automatically. Custom colored views (like mini progress bars) inside the label are fine, but the overall item must not have a container/background style applied to it.

#### 3.2 — Data Models

Define `Codable` structs matching these API responses:

**Usage endpoint** (`GET https://claude.ai/api/organizations/{org_id}/usage`):
```json
{
  "five_hour": {
    "utilization": 78.0,
    "resets_at": "2025-11-13T04:00:00+00:00"
  },
  "seven_day": {
    "utilization": 35.0,
    "resets_at": "2025-11-06T03:59:59+00:00"
  },
  "seven_day_opus": {
    "utilization": 0.0,
    "resets_at": null
  },
  "extra_usage": {
    "utilization": 0.0,
    "monthly_limit": 0,
    "used_credits": 0
  }
}
```

**Organizations endpoint** (`GET https://claude.ai/api/organizations`):
Returns an array of objects. We only need `uuid` (String) and `name` (String, optional). Ignore all other fields.

Also define a `DisplayStyle` enum with cases: `text`, `bars`, `dots`.

#### 3.3 — Usage Service

Create an `ObservableObject` called `UsageService` that manages all app state:

**Published state:**
- `usageData` — the latest fetched usage (optional)
- `organizations` — list of organizations
- `isLoading`, `errorMessage`, `lastUpdate`

**Persisted settings** (stored in `UserDefaults`, loaded in `init`):
- `sessionKey` (String, default: empty)
- `organizationId` (String, default: empty)
- `displayStyle` (DisplayStyle, default: `.bars`)
- `showFiveHour` (Bool, default: true) — controls menu bar label visibility
- `showSevenDay` (Bool, default: true) — controls menu bar label visibility
- `refreshInterval` (TimeInterval, default: 300 seconds)

**Menu bar label** — this is a **SwiftUI View** (not a String). The `MenuBarExtra` label is a `@ViewBuilder`, so use real SwiftUI views.

Implement a `MenuBarLabel` view that the App passes as the `MenuBarExtra` label. It reads the display style from the usage service and renders accordingly:

**Style: `bars` (default)**

**CRITICAL:** `MenuBarExtra` on macOS does NOT reliably render complex SwiftUI views (like `RoundedRectangle`, `ZStack`, `GeometryReader`) in the label. These views are silently ignored or only partially rendered. You MUST use the `NSImage` approach described below — do NOT attempt to use SwiftUI shape views in the menu bar label.

Render the bars as a pre-rasterized `NSImage` using AppKit/Core Graphics drawing, then display it via `Image(nsImage:)`. This is the only reliable way to show custom graphics in a `MenuBarExtra` label. The technique:

1. Create an `NSImage` of the right size for all metric rows
2. Use `lockFocus()` (or `NSGraphicsContext` / `CGContext`) to draw into the image
3. For each metric: draw a label (`NSAttributedString`, ~7pt monospaced font, use `NSColor.labelColor` for Dark/Light Mode support) + a gray rounded-rect background bar + a colored rounded-rect fill bar (width proportional to utilization %)
4. Set `image.isTemplate = false` — this is critical, otherwise macOS renders it as a monochrome template image and the colors are lost
5. In the `MenuBarLabel` SwiftUI view, embed with `Image(nsImage: renderedImage)`

Target bar dimensions: ~40pt wide, ~6pt tall, ~2pt corner radius. Two rows (5h + 7d) fit comfortably in the ~22pt menu bar height.

Color coding: green (`NSColor.systemGreen`) for utilization < 50%, orange (`NSColor.systemOrange`) for 50–80%, red (`NSColor.systemRed`) for > 80%.

**Style: `text`**
Render a `Text` view: `5h:45% 7d:72%`

**Style: `dots`**
Render a `Text` view with emoji dots: `🟢🟡`

**When no data:** Render `Text("Setup ⚙")` if no session key, or `Text("...")` if loading.

This view must be a separate `struct MenuBarLabel: View` so it can be used directly in the `MenuBarExtra` label builder.

**Polling:**
- `startPolling()` — fetches immediately, then repeats on a `Timer` at `refreshInterval`
- On init, auto-start polling if `sessionKey` and `organizationId` are already saved

**API calls:**

`fetchOrganizations(completion:)`:
- `GET https://claude.ai/api/organizations`
- Header: `Cookie: sessionKey={sessionKey}`
- Header: `Accept: application/json`
- On success: populate `organizations`, auto-select the first org if none selected
- Takes a completion callback so the Settings UI can react

`fetchUsage()`:
- `GET https://claude.ai/api/organizations/{organizationId}/usage`
- Header: `Cookie: sessionKey={sessionKey}`
- Header: `Accept: application/json`
- Handle HTTP 401/403 → show "Session expired" error
- Handle other errors gracefully (network, parse, etc.)
- On success: decode response, update `lastUpdate`

#### 3.4 — Popover View

This view appears when the user clicks the menu bar item. Layout:

```
┌──────────────────────────────────┐
│  📊 Claude Usage                 │
│                                  │
│  5-Hour     ████████░░░░  78%    │
│  Resets in 2h 15m                │
│                                  │
│  7-Day      ████░░░░░░░░  35%   │
│  Resets in 4d 12h                │
│                                  │
│  ──────────────────────────────  │
│  ↻ Refresh              ⚙ Settings │
│  Updated 2 min ago         Quit  │
└──────────────────────────────────┘
```

**Behavior:**
- If no session key: show a centered prompt with a key icon, message "No session key configured", and an "Open Settings" button
- If data loaded: show a `UsageRow` for each non-nil metric (5-hour, 7-day, 7-day opus if > 0%, extra credits if > 0%)
- If loading: show a spinner
- If error: show the error message in red
- Footer: "Refresh" button (left), "Settings" button (right), last update time, "Quit" button

**UsageRow** subview:
- Title + percentage on one line (percentage colored: green < 50%, orange 50-80%, red > 80%)
- A custom progress bar using `GeometryReader` with a `RoundedRectangle` track (gray, ~6pt height) and a filled `RoundedRectangle` overlay (color-coded). This looks much better than the default `ProgressView`. Use a smooth corner radius (~3pt).
- "Resets in X" using a relative date formatter on the `resets_at` ISO 8601 timestamp

**Opening Settings:** Since we target macOS 14+, use `SettingsLink()` as a native SwiftUI button to open the Settings scene. Alternatively, use `openWindow(id:)` environment action if using a named `Window` scene.

**IMPORTANT — Activation Policy trick for LSUIElement apps:** Because this app uses `LSUIElement = true` (no Dock icon), macOS treats it as an accessory app that cannot own the active window. This causes the Settings window to open behind other apps and render without proper window controls (red/yellow/green buttons). Fix this with the activation policy pattern:
- When opening Settings: call `NSApp.setActivationPolicy(.regular)` to temporarily make the app a normal app that can own windows. Then call `NSApp.activate(ignoringOtherApps: true)`.
- When closing Settings: in the Settings view's `onDisappear`, call `NSApp.setActivationPolicy(.accessory)` to return to menu bar-only mode.
This ensures the Settings window comes to the foreground with correct window controls, and the app disappears from the Dock again when settings are closed.

#### 3.5 — Settings View

A tabbed window (480 x 400 points) with two tabs:

**Tab 1: Account** (icon: `person.circle`)
- **Session Key field:** `SecureField` with a Show/Hide toggle button. Placeholder: `sk-ant-sid01-...`
- **Instructions text** (caption, secondary color): "How to find your session key: 1. Go to claude.ai in your browser, 2. Open DevTools (⌘⌥I), 3. Go to Application tab → Cookies → claude.ai, 4. Copy the value of the `sessionKey` cookie"
- **Connect button** (borderedProminent style): saves the key, calls `fetchOrganizations`, shows success/failure status next to it
- **Organization picker:** only shown after successful connection. Dropdown of fetched organizations. On selection change, start polling with the new org.

**Tab 2: Display** (icon: `paintbrush`)
- **Menu Bar Style:** radio group picker with three options (text/bars/dots), each with a visual example
- **Visible Metrics in Menu Bar:** two toggles for 5-hour and 7-day (these control which bars appear in the menu bar label; Opus and Extra usage are always shown in the popover when > 0%)
- **Refresh Interval:** slider from 1 to 10 minutes (step: 30 seconds), with a label showing current value

Use `.formStyle(.grouped)` on both tab forms.

---

### Phase 4: Build

Run from inside the project directory:

```bash
swift build -c release 2>&1
```

If the build fails, read the error messages carefully, fix the source code, and rebuild. Iterate until the build succeeds. Do NOT give up — debug and fix compiler errors.

The binary should appear at `.build/release/ClaudeUsage`.

---

### Phase 5: Package as .app Bundle

Create a proper macOS app bundle so it runs as a menu bar app (no Dock icon):

1. Create `~/Applications/ClaudeUsage.app/Contents/MacOS/` and `~/Applications/ClaudeUsage.app/Contents/Resources/`
2. Copy the built binary to `Contents/MacOS/ClaudeUsage`, ensure it's executable
3. **Generate an app icon**: Create a visually appealing `AppIcon.icns` file and place it in `Contents/Resources/`. The icon should convey "usage monitoring" or "quota tracking" — for example, a gauge, a pie chart, or a bar chart with the Claude purple/orange brand colors. Use `sips` and `iconutil` to generate the `.icns` from PNG files. Create the PNGs programmatically using a Swift script, Python with Pillow, or any tool available on the system. Include all required sizes (16, 32, 128, 256, 512 at 1x and 2x).
4. Write a `Contents/Info.plist` with:
   - `CFBundleExecutable`: `ClaudeUsage`
   - `CFBundleIdentifier`: `com.spec4.claudeusage`
   - `CFBundleName`: `Claude Usage`
   - `CFBundleIconFile`: `AppIcon`
   - `CFBundlePackageType`: `APPL`
   - `CFBundleVersion`: `1`
   - `CFBundleShortVersionString`: `1.0`
   - `LSMinimumSystemVersion`: `14.0`
   - **`LSUIElement`: `true`** — this makes it a menu bar-only app with no Dock icon

---

### Phase 6: Launch and Report

Launch the app with `open ~/Applications/ClaudeUsage.app`.

Then print this message to the user:

```
ClaudeUsage is now running in your menu bar! Look for the chart icon (📊) in the top-right of your screen.

First-time setup:
1. Click the chart icon in the menu bar
2. Click "Open Settings"
3. Paste your Claude session key (instructions are in the settings window)
4. Click "Connect" — the app will fetch your usage automatically

If macOS blocks the app ("unidentified developer" warning):
- Go to System Settings → Privacy & Security → scroll down → click "Open Anyway"
- Or: right-click the app in ~/Applications → Open → Open

To quit: Click the menu bar icon → Quit

The app source code is in ./ClaudeUsage/ if you want to inspect or modify it.
```

---

### Troubleshooting (for the AI)

If any of these issues occur, fix them:

1. **`MenuBarExtra` not available**: Target must be macOS 14+. Check `Package.swift` platform.
2. **App doesn't appear in menu bar**: Ensure `LSUIElement` is `true` in Info.plist and the app is launched via `open`, not by running the binary directly.
3. **Network requests fail**: The app connects to `https://claude.ai` which requires HTTPS. No special ATS configuration needed.
4. **Binary not found after build**: Check `.build/release/ClaudeUsage` exists. If using a different build path, find it with `find .build -name ClaudeUsage -type f`.
5. **App crashes on launch**: Run the binary directly from terminal to see the crash log: `.build/release/ClaudeUsage`. Fix the issue and rebuild.
6. **`@Published` with `didSet` not persisting**: Ensure each persisted property writes to `UserDefaults` in its `didSet` and reads from `UserDefaults` in `init()`.
7. **Settings window won't open or opens behind other apps**: This is a known issue with `LSUIElement` apps. The selectors `showSettingsWindow:` / `showPreferencesWindow:` may silently fail. Fix: use `SettingsLink()` on macOS 14+, or fall back to manually creating an `NSWindow` with `NSHostingController(rootView: SettingsView(...))`. To bring the window to front, use the activation policy trick: call `NSApp.setActivationPolicy(.regular)` before opening, then `NSApp.activate(ignoringOtherApps: true)`. Reset to `.accessory` in the Settings view's `onDisappear`.
8. **Menu bar bars not rendering / only text visible**: `MenuBarExtra` does NOT reliably render SwiftUI shape views (`RoundedRectangle`, `ZStack`, etc.) in the label. These are silently ignored. You MUST use the `NSImage` + Core Graphics approach described in section 3.3 — draw the bars into an `NSImage` using `lockFocus()` and `NSBezierPath`, then display via `Image(nsImage:)`. Do NOT attempt SwiftUI shapes as a fallback.
