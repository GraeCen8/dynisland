# DynIsland — Specification

A minimal, keyboard-driven desktop shell for **Niri**, built with **Quickshell**.

The entire user interface is a single **island** anchored to the top centre of the
focused monitor. At rest it is almost invisible: only a small strip of workspace
dots and a clock peek out from above the screen's top edge. It expands when you
invoke it or when a system event occurs, then retreats.

> **This document is a build specification for a coding agent.** Implement what is
> written here. Where the spec is silent, choose the simplest option consistent
> with §1 and keep it configurable rather than hard-coded. The decisions in
> Appendix A were settled during design and are **final** — do not reopen them.

---

## Table of contents

1. [Philosophy](#1-philosophy)
2. [Non-goals](#2-non-goals)
3. [Tech stack](#3-tech-stack)
4. [Project layout](#4-project-layout)
5. [The island](#5-the-island)
6. [States](#6-states)
7. [Interaction model](#7-interaction-model)
8. [Sections](#8-sections)
9. [Launcher](#9-launcher)
10. [Events](#10-events)
11. [Configuration](#11-configuration)
12. [Theming](#12-theming)
13. [Icons](#13-icons)
14. [Lifecycle](#14-lifecycle)
15. [IPC](#15-ipc)
16. [Service interfaces](#16-service-interfaces)
17. [Widget library](#17-widget-library)
18. [Acceptance criteria](#18-acceptance-criteria)
19. [Build order](#19-build-order)
20. [Appendix A — decision record](#appendix-a--decision-record)
21. [Appendix B — reference](#appendix-b--reference)

---

## 1. Philosophy

These are the rules that resolve ambiguity. When two requirements conflict, the
earlier one wins.

1. **One surface.** There is no bar, dock, dashboard, control centre, or
   notification centre as a separate visual system. Everything is a *state of one
   island*. If a feature would naturally be its own window, it is instead a
   state of the island.
2. **Minimal persistent UI.** At rest the island occupies almost no screen space.
   There is no permanent status bar. A user watching the resting shell should see
   dots and a clock and nothing else.
3. **Keyboard first.** Every action must be reachable from the keyboard. The mouse
   may be supported, but must never be *required* for anything.
4. **No decorative motion.** An animation must communicate a state change. No
   bounces, no flourishes, no animation for its own sake.
5. **Composable with Niri.** Niri owns windows and workspaces. Use Niri IPC rather
   than duplicating compositor functionality. If Niri already has a keybind for
   something, the shell does not need to provide it.
6. **Absent means absent.** A state whose data is not present is not rendered. No
   empty sections, no placeholder rows, no "Music" section when nothing is
   playing, no zero-valued metrics on hardware that lacks the sensor.
7. **One config file.** Shell behaviour is configured in a single TOML file. A
   user should never need to edit more than one file to change how the shell
   behaves.
8. **Deterministic.** Identical input produces identical output. No usage
   tracking, no behavioural learning, no telemetry.

---

## 2. Non-goals

Do **not** build any of the following. Each one turns this into a desktop
environment, which is the explicit opposite of the goal:

- A dock, taskbar, or persistent status bar
- A desktop dashboard or widget grid
- A window switcher, window preview grid, or per-workspace thumbnails
- A per-application audio mixer
- Display reconfiguration UI (resolution/refresh/scale editing)
- A lock screen, login screen, or wallpaper/background system
- A crash handler or error UI
- Any AI or LLM integration
- Application icons or branding inside the island beyond a launcher's entry
  icon
- A second compositor backend (this shell targets Niri; see §3)

---

## 3. Tech stack

| Concern | Choice | Notes |
|---|---|---|
| Framework | **Quickshell** (QML) | Not Qt Quick directly |
| Compositor | **Niri** | Via its IPC socket. No other backend. |
| Config format | **TOML** | Single file, hot-reloaded |
| Themes | One **QML** file per theme | Hot-swappable at runtime |
| Notifications | `org.freedesktop.Notifications` (D-Bus) | Standard spec, do not reinvent |
| Media | **MPRIS** (`org.mpris.MediaPlayer2.*`) | Works with browsers, Spotify, players |
| Audio | **PipeWire** + **WirePlumber** | |
| Brightness | sysfs `/sys/class/backlight`, `brightnessctl` fallback | |
| Network | **NetworkManager** (D-Bus) | |
| Bluetooth | **BlueZ** (`org.bluez`, D-Bus) | |
| Power profiles | `powerprofilesctl` / `net.dunst` equivalent D-Bus | |
| Night light | External command (`sct`, `gammastep`, `wlgamma`, `wlsunset`) | Neither compositor has this natively; the command is TOML-configurable |
| Screenshot | External (`grim` + `slurp`) | Niri has no built-in screenshot |
| Icons | Bundled core set, system theme as override | See §13 |
| Packaging | **NixOS module + Home Manager** | Additive, not required to run |

### 3.1 Standalone operation

The shell must run **without Nix**, for development and testing:

```sh
quickshell -p ./shell
```

Nix packaging is additive. There must be no build-time or run-time dependency on
Nix-provided paths.

### 3.2 Niri interaction

Use Niri IPC for everything compositor-related:

| Need | Mechanism |
|---|---|
| List workspaces | `niri msg --json workspaces` |
| List outputs | `niri msg --json outputs` |
| List windows | `niri msg --json windows` |
| Focused window | `niri msg --json focused-window` |
| Focused output | `niri msg --json focused-output` |
| Event stream | `niri msg --json event-stream` |
| Focus a workspace | `niri msg action focus-workspace <ref>` |
| Focus a monitor | `niri msg action focus-monitor <name>` |
| Output geometry | `outputs[].logical` → x, y, width, height, scale |

The shell must **not** reimplement window management. It never moves, resizes, or
closes windows except through explicit Niri actions.

### 3.3 Niri workspace semantics — important

Niri workspaces are **dynamic**. Niri automatically collapses workspaces that
become unused, keeping only the main one. The `active` flag in
`niri msg --json workspaces` distinguishes a live workspace from a collapsed one.

**The dot row renders only `active` workspaces.** Collapsed workspaces are not
rendered at all. This is not a shell policy — it is a direct reflection of Niri's
real state, and it is why the resting island can stay so small.

---

## 4. Project layout

```
dynisland/
├── shell.qml                  # root window, island geometry, state machine
├── SPEC.md                    # this document
├── config/
│   ├── Config.qml             # TOML loader + typed config singleton
│   └── Defaults.toml          # shipped defaults, copied on first run
├── core/
│   ├── Island.qml             # morphing container; owns geometry + animation
│   ├── StateMachine.qml       # resting / interactive / transient / overlay
│   ├── EventQueue.qml         # queueing, priority, interruption, resume
│   ├── SectionRegistry.qml    # registers sections, tracks relevance
│   ├── FocusModel.qml         # contextual focus + key routing
│   └── Ipc.qml                # `qs ipc` command surface
├── services/
│   ├── NiriService.qml        # workspaces, windows, outputs, focus
│   ├── NotificationService.qml
│   ├── MprisService.qml
│   ├── AudioService.qml
│   ├── BrightnessService.qml
│   ├── NetworkService.qml
│   ├── BluetoothService.qml
│   ├── PowerService.qml
│   └── NightLightService.qml
├── sections/
│   ├── Workspaces.qml
│   ├── Sysmon.qml
│   ├── Media.qml
│   ├── Notifications.qml
│   └── Settings.qml
├── states/
│   ├── Launcher.qml
│   ├── Volume.qml
│   ├── Brightness.qml
│   ├── PowerMenu.qml
│   └── Screenshot.qml
├── widgets/
│   ├── Dot.qml
│   ├── Slider.qml
│   ├── Toggle.qml
│   ├── List.qml
│   ├── ListItem.qml
│   ├── Button.qml
│   ├── Section.qml
│   └── Confirm.qml
└── themes/
    ├── rose_pine.qml
    ├── tokyo_night.qml
    └── ...
```

### 4.1 Layering rules

These are architectural constraints, not style preferences:

1. **UI never talks to D-Bus or Niri IPC directly.** It calls a service. This is
   what keeps the UI independent of the backend and makes services swappable.
2. **Sections never read config singletons directly for behaviour.** They declare
   requirements (e.g. `axis: Horizontal`, `requires: activeMprisPlayer`) and the
   registry decides whether they exist.
3. **Themes never affect logic.** A theme is data. See §12.
4. **No section may open its own window.** All content renders inside the island.

---

## 5. The island

### 5.1 Geometry

- Anchored **top centre of the focused monitor**.
- At rest the island's body sits **above the top edge of the screen**. Only a
  strip is visible.
- The strip's contents, left to right:
  1. **Workspace dots** — no numbers, ever
  2. **Clock** — fixed 24-hour `HH:MM`

```
              ·  ●  ◉  ●  ·      14:32
──────────────────────────────────────────  ← top edge of screen
```

- **No battery, no network, no audio indicators in the resting state.** Those are
  events, or they belong in Sysmon/Settings.

### 5.2 Workspace dot rendering

Dots mirror Niri's real workspace state (§3.3). Three visual states:

| State | Rendering |
|---|---|
| **Focused** workspace | **Largest**, brightest, **thin border ring** |
| **Occupied** (has windows) | Medium size, **brighter** fill |
| **Existing but empty** | **Small**, **dim** fill |
| **Collapsed / non-existent** | **Not rendered at all** |

Occupancy **must** be visually distinguishable from empty. This is a hard
requirement — an occupied workspace and an empty one may not look the same.

```
  unused       used       current        used       unused

    ·           ●           ◉              ●           ·
   dim       brighter    bright +       brighter      dim
                           thin border
```

The three states must differ on at least two visual channels (size and
brightness) so they remain distinguishable for users with reduced colour
perception.

### 5.3 Expansion model

The island has **no fixed expanded size**. Its geometry and contents are
determined entirely by its current state. There is no single "expanded layout"
that all states share.

| State | Expands |
|---|---|
| Resting | — (strip) |
| Launcher | downward |
| Media | horizontally |
| Sysmon | horizontally |
| Notifications | downward |
| Settings | downward |
| Power menu | downward |
| Volume / Brightness pill | horizontally, minimally |

Transitions **morph**. The island is one continuous object changing shape — not
one panel disappearing while another appears. Closing reverses the transition
back to rest.

### 5.4 Animation

- **Fast, subtle, interruptible, state-driven.**
- An interrupted transition **retargets from its current geometry**. It must never
  snap, restart from zero, or wait for the in-flight animation to finish.
- There must be **no hard visual pop** between any two states, including across
  a shell reload.
- Duration is configurable (§11). Default `fast` ≈ 180 ms.
- Prefer a single interpolating geometry property per axis so retargeting is
  natural. Avoid sequential `Behavior` animations on nested elements, which
  compound and produce visible lag.

### 5.5 Size limits

| Mode | Limit | Applies to |
|---|---|---|
| Vertical | up to **~40–45% of screen height** | Launcher, Notifications, Settings, Power menu |
| Horizontal | up to **~40–50% of screen width** | Media, Sysmon, Volume/Brightness pills |

**A section expands primarily on one axis, not both.** Each section declares its
preferred axis in §17 so the island keeps a purposeful shape instead of
becoming a generic large rectangle.

These are **fixed percentages**. The island does **not** adapt its maximum to
monitor size — the same percentages apply on a laptop panel and a 4K display.

### 5.6 Monitors — focused monitor only

There is **exactly one island**, living on the **currently focused monitor**.

- When Niri's focused output changes, the island transitions to the new monitor.
- No idle islands are left on inactive monitors.
- A keyboard invocation always opens on the focused monitor.
- Mouse interaction acts on the island under the pointer's monitor.
- On hotplug, the island re-anchors without a restart.

---

## 6. States

There are three classes of state, plus an overlay mechanism for interruptions.

### 6.1 Resting

The default, and the startup state. Workspace dots + clock.

**The shell starts directly in resting state** — no splash screen, no loading
animation, no layout flash.

### 6.2 Navigable sections

Sections the user moves *through*:

| Section | Exists when | Axis |
|---|---|---|
| **Workspaces** | Always (default landing) | Horizontal |
| **Sysmon** | Always | Horizontal |
| **Settings** | Always | Vertical |
| **Media** | An MPRIS player is active | Horizontal |
| **Notifications** | At least one notification exists | Vertical |

A section with nothing to show **does not exist**. It is removed from the cycle
entirely — not rendered as an empty container.

**Workspaces is the default landing section.** It is the fallback when nothing
else is relevant. It is *not* permanently pinned into every state; it yields the
island's space to whatever currently needs it.

**Relevance rule for the cycle:** `Tab` cycles through currently-relevant
sections. Workspaces joins the cycle only when it has had recent activity — a
workspace focus change, creation, removal, or occupancy change — within
`events.workspaces_relevance_ms`.

The **island key** always lands on Workspaces first, but pressing `Tab` moves
into whatever is currently active. So with music playing and no workspace
activity:

```
island key  →  Workspaces  →  Tab  →  Sysmon  →  Tab  →  Media  →  Tab  →  Settings
```

The cycle is **cyclic** — advancing past the last section wraps to the first.

### 6.3 Direct and transient states

These are **not** in the section cycle. They are entered by their own bind or by
a system event, and take over the island temporarily.

| State | Entered by | Dismissal | Focus |
|---|---|---|---|
| **Launcher** | Island bind | `Esc`, or auto-rest after an action | Takes focus |
| **Volume** | Volume key / service event | Auto-collapse | No focus |
| **Brightness** | Brightness key / service event | Auto-collapse | No focus |
| **Power menu** | Power bind, or Settings→Power | `Esc` or action | Takes focus |
| **Notifications** | Notification bind, or event | `Esc` | Takes focus |
| **Screenshot** | Screenshot bind | Auto-collapse | No focus |

**Interactive** states (entered by an explicit bind) stay open until `Esc` and
take keyboard focus.

**Transient** states (entered by a system event) display, then auto-collapse
after a configurable timeout, and **never steal keyboard focus**.

---

## 7. Interaction model

### 7.1 Keyboard

| Key | Action |
|---|---|
| **Island key** (`Super`) | Open island → Workspaces, focus first meaningful item |
| **Direct binds** | Jump straight to a named section or state |
| `Tab` | Next section |
| `Shift+Tab` | Previous section |
| `h` / `l` or `←` / `→` | Previous / next **item within** the current section |
| `j` / `k` or `↓` / `↑` | **Vertical action on the focused control** (§7.2) |
| `Enter` | Activate / select the focused item |
| `Esc` | Collapse the island to rest |
| `Ctrl+C` | Cancel (text-entry contexts) |

All keys are remappable in TOML where meaningful (§11).

### 7.2 The contextual meaning of `j` / `k`

`j`/`k` acts on **whatever has vertical structure at the focus point**. It is
**never** used to change sections.

| Focus is on | `j` / `k` does |
|---|---|
| A slider (volume, brightness, progress) | Decrease / increase the value |
| A vertical list (notifications, networks, devices) | Move down / up the list |
| A toggle | No vertical meaning — ignored |
| A horizontal dot row | No vertical meaning — ignored |

This is the core of the model: **navigation keys operate on the thing currently
under focus.** A key's meaning is determined by context, and no key ever does
something the user did not ask for. In particular, adjusting a volume slider
must never change your workspace.

### 7.3 `Tab` and text fields

`Tab` **always** means "next island section". Controls must never hijack it.

The single exception: while a **text field** has focus (the launcher), normal
text-editing behaviour takes priority, including `Tab` if the field uses it.

### 7.4 Focus on open

| Trigger | Focus lands on |
|---|---|
| Island key | First meaningful item in **Workspaces** |
| Direct section bind | That section's first / most relevant control |
| Launcher | The **search field**, immediately |
| Transient system event | **Nothing** — focus is not taken |

A volume change must never cause the shell to start consuming keystrokes. This
is a hard requirement.

### 7.5 Mouse

The mouse mirrors the keyboard model with different physical gestures. It is
never required.

| Gesture | Result |
|---|---|
| Pointer reaches the **top edge** | Island wakes (subtle; not a hotspot) |
| Pointer moves away without interaction | Collapses after a short delay |
| **Click** | Select / activate |
| **Drag left/right** | Move between items within the current section |
| Click / drag a control | Interact with it normally |
| **Click outside** | Collapse |

**Dragging is horizontal only.** Vertical drags must not navigate sections, so
dragging never fights with vertical controls like sliders or list scrolling.

Hover alone must never be required to operate anything.

---

## 8. Sections

### 8.1 Workspaces

Deliberately minimal. It **visualizes** Niri; it does not manage workspaces.

- Dot row mirroring `active` Niri workspaces, rendered per §5.2.
- **Clicking a dot switches to that workspace**, then the island collapses to
  rest and the dots re-render showing the new focus. Configurable via
  `events.collapse_on_workspace_click` for users who prefer it to linger.
- **No dedicated keyboard workspace navigation.** Niri's own keybinds already
  handle workspace switching; the shell must not duplicate that. The section only
  takes keyboard focus if there is another reason for it.
- **No window previews.** No numbers. No labels.
- Expands transiently on workspace focus / creation / removal / occupancy change.

### 8.2 Sysmon

A **compact status snapshot**, not a live dashboard with graphs.

```
┌────────────────────────────────┐
│  14:32                        │
│                                │
│  CPU    32%       RAM   48%    │
│  GPU    21%       DISK  67%    │
│  NET    ↓ 2.4M    ↑ 180K       │
│  BAT    82%                     │
└────────────────────────────────┘
```

- Metrics: **CPU, RAM, GPU, DISK, NET (↓/↑ transfer rates), BAT**, plus the
  clock.
- **No history graphs in v1.** The layout must be structured so graphs can be
  added later without redesigning the island or the section.
- **Adaptive:** a metric with no data on this machine (no battery on a desktop,
  no discrete GPU) is **omitted entirely** — never shown as `0`, `N/A`, or an
  empty row. This is rule 6 of the philosophy.
- Sampling interval should be modest (~1 s) and the section must be cheap enough
  to open instantly.

### 8.3 Media

**Exists only while an MPRIS player is active.** No player → no section.

Contents:

- Artwork, if the player provides it
- Title
- Artist
- Playback progress
- Transport: previous, play/pause, next

**No volume control.** System volume is its own state (§8.7) and is reachable
separately. This is intentional — volume is duplicated in three places otherwise.

### 8.4 Notifications

- A **vertical stack, maximum 4 visible at once**, scrolling with `j`/`k`.
- `h`/`l` navigates **within** the focused notification — specifically its action
  buttons.
- `Enter` opens/activates the focused notification.
- **Only actions the application provided may appear.** The shell must never
  invent actions.
- Notifications are **data inside the island**, not a separate notification
  centre window.
- Backed by standard `org.freedesktop.Notifications`.

**Do Not Disturb** is a simple **on/off toggle**:

- Normal-priority notifications are **suppressed** while DND is on.
- **Urgent/high-priority notifications can still interrupt** (§10.2).
- Suppressed notifications **remain in history** and become visible when DND is
  turned off.

**History is per-shell-instance.** Restarting Quickshell clears it. There is no
persistence to disk.

### 8.5 Settings — system controls

The Settings section controls **the computer**. Shell configuration is TOML and
is deliberately *not* exposed here.

| Group | Capabilities |
|---|---|
| **Wi-Fi** | Enable/disable, scan, list networks, connect/disconnect, signal strength, security type |
| **Bluetooth** | Enable/disable, scan, list devices, pair, connect/disconnect, device status |
| **Audio** | Output device, input device, master volume, mute, device status |
| **Displays** | **Status only** — connected monitors, resolution, refresh rate, scale, enabled state |
| **Power** | Battery status, charging state, power profile, suspend behaviour |
| **Night Light** | **On/off toggle only** — no colour-temperature slider |
| **Do Not Disturb** | **On/off toggle only** — no scheduling |
| **Appearance** | Theme selection, live switch |

Explicit limitations:

- **Audio has no per-application mixer.** Output/input selection, master volume,
  mute, and status only.
- **Displays are status only.** No resolution/refresh/scale/position editing.
- **Night Light and DND are toggles only.** No sliders, no schedules.
- **Power here is status and configuration.** Session actions (lock, suspend,
  logout, reboot, shutdown) live in the **Power menu** (§8.6).

Network and Bluetooth are **fully interactive inside the shell**. Do not shell
out to an external settings application for normal operation.

Settings is the most vertical section: it grows downward as groups open, up to
the height cap.

### 8.6 Power menu

A compact **vertical** direct state, in this order:

```
Lock  →  Suspend  →  Logout  →  Reboot  →  Shutdown
```

- Stays open until `Esc` or an action is chosen.
- **Destructive actions require confirmation:** Reboot and Shutdown. Logout
  confirms as well. **Lock and Suspend do not.**
- Reachable both from `Settings → Power` and from a configurable direct bind.

### 8.7 Volume and Brightness

Not sections. Compact transient pills:

```
🔊 ━━━━━━━●━━  72%
☀  ━━━━●━━━━   45%
```

- Display on change, then auto-collapse after a short configurable timeout.
- **Never steal keyboard focus.**
- Volume/brightness **jump the event queue** (§10.3) and **update in place** on
  repeated changes.
- Volume over 100% is shown clamped to 100% in the bar while still permitting
  over-amplification through interaction.

---

## 9. Launcher

`Super+Space` opens the Launcher. It is a **direct state, not a section**.

### 9.1 Behaviour

- The **search field is focused immediately**. Type straight away — no extra
  keypress, no click required.
- Results **fold downward** beneath the field, inside the island.
- `Enter` executes the highlighted result.
- `Esc` closes and returns to rest.
- After an action executes, the island **returns to rest automatically**.

### 9.2 Matching and ranking

- **Fuzzy matching.** Matching quality is the feature. Subsequence matching with
  scoring that rewards prefix hits, word-boundary hits, and contiguity.
- **Deterministic ranking.** Identical queries always produce identical order.
  Ties broken by a stable secondary key (alphabetical title), never by
  insertion order or timestamps.
- **No usage tracking, no learning, no personalisation, no telemetry.**
- **One unified ranked list.** Not a menu hierarchy, not grouped by category.

```
╭────────────────────────────╮
│ > fire                     │
├────────────────────────────┤
│ 📶 Firefox                 │
│ 📁 Files                   │
│ 🎵 Firefox (Music)         │
╰────────────────────────────╯
```

**Icons visually distinguish entry types** — applications, future toggles, and
future actions all appear in the same list, distinguished by icon.

### 9.3 v1 scope and the provider interface

**v1 searches applications only**, sourced from the system desktop-entry
directories.

**Architectural requirement:** the launcher must be built as a generic **action
launcher** behind a provider interface, even though only the application
provider is enabled initially.

```qml
// LauncherProvider.qml — interface
property string id          // stable identifier
property string title       // display name
property string icon        // icon name or path
property string type        // "app" | "toggle" | "action"
property int    weight      // base ranking weight
function matches(query)     // fuzzy match + score
function activate()         // execute
```

Future providers must be addable **without touching the launcher UI**:

- **Toggles** — Night Light, DND, mute
- **Actions** — change power profile, jump to a shell section
- **Later** — files, commands, web search

This is why the launcher is a direct state and not a section: it has its own
lifecycle, its own focus model, and its own provider contract.

---

## 10. Events

### 10.1 The event queue

Simultaneous events do not fight for the island. They **queue**, and each is
given enough display time to actually be read before the next appears.

```
Notification A  →  Notification B  →  Network change  →  Battery warning
```

The queue is a first-in-first-out list of pending events, each with its own
configured duration. A new event is appended, not prepended (with the exceptions
in §10.3).

### 10.2 Interrupting an interactive state

If the user is in an interactive section (reading Sysmon, say) and an event
arrives:

- **Normal / low-priority event** → shows as a small transient indicator. Does
  **not** take over the island, does not disturb the user's selection, does not
  take focus.
- **High-priority / urgent event** → temporarily morphs the island into the
  event state, then **returns to the previous state with the exact selection and
  focus restored**.

The island behaves like an application with **interruptible overlays that
restore state**. It must not reset to defaults every time something happens.

An overlay is a stack: an urgent notification during an urgent battery warning
pushes the battery warning, and unwinding restores both, in order.

### 10.3 Volume and brightness are special

Volume and brightness **jump to the front** of the queue for a short time, then
the interrupted event **resumes exactly where it left off**.

```
Notification A (at 1.2 s of 2.5 s)
   ↓ volume key
🔊 72%          ← jumps to front
   ↓ timeout
Notification A  ← resumes at 1.2 s, not from the beginning
   ↓ completes
Notification B
```

**Repeated volume or brightness changes update the existing event in place.**
Holding the volume key produces **one** event with a live-updating value — never
twenty stacked events. The queue must never grow from key-repeat.

### 10.4 Transient event catalogue

| Event | Display | Notes |
|---|---|---|
| Volume | `🔊 ━━━━━●━━ 72%` | Updates in place; jumps queue |
| Brightness | `☀ ━━━━●━━━━ 45%` | Updates in place; jumps queue |
| Network | `⚠ Wi-Fi disconnected` / `✓ Wi-Fi connected — SSID` | **Filtered** so routine connectivity noise is not annoying |
| Battery | Threshold crossings at **50 / 20 / 10 / 5 %** | Plus charging and full-state transitions. Thresholds configurable |
| Screenshot | `↑ Screenshot saved` | |
| Media | Track change | |
| Workspace | Dot row re-render (§8.1) | |

**Battery is not in the resting state** — it is purely event-driven.

All transient events auto-collapse and **never steal keyboard focus** (§7.4).

---

## 11. Configuration

One file: `~/.config/dynisland/config.toml`. A commented `Defaults.toml` ships
with the shell and is copied into place on first run.

**Reloading the config must not require restarting the shell or the compositor.**

```toml
[general]
theme      = "rose_pine"   # filename in themes/, without the .qml extension
icon_theme = "auto"        # "auto" = bundled; otherwise a system icon theme name
position   = "top"         # top is the only supported position in v1

[rendering]
transparency   = true
opacity        = 0.92      # 0.0–1.0, multiplies the theme's surface colour
blur           = true
blur_radius    = 24
animation_speed = "fast"   # instant | fast | normal | slow, or an integer ms
                            # fast ≈ 180 ms, normal ≈ 260 ms, slow ≈ 380 ms

[geometry]
max_height_percent = 45    # vertical-mode cap, % of screen height
max_width_percent  = 50    # horizontal-mode cap, % of screen width

[keybinds]
island        = "Super"
launcher      = "Super+Space"
sysmon        = "Super+Shift+S"
media         = "Super+Shift+M"
notifications = "Super+Shift+N"
power         = "Super+Shift+P"
screenshot    = "Print"
reload_shell  = "Super+Shift+R"

[sections]
workspaces    = true
sysmon        = true
media         = true
notifications = true
settings      = true
# Tab-cycle order. Sections whose data is absent are skipped regardless.
order = ["workspaces", "sysmon", "media", "notifications", "settings"]

[events]
display_ms                  = 2500  # default transient event duration
volume_display_ms           = 1200
brightness_display_ms       = 1200
volume_interrupts           = true
brightness_interrupts       = true
urgent_notification_ms      = 4000  # urgent events are given longer
workspaces_relevance_ms     = 3000  # how long Workspaces stays in the Tab cycle
                                   # after a workspace event
collapse_on_workspace_click = true
network_events_enabled      = true
overlay_restore_ms          = 250   # grace period before restoring the prior state

[battery]
thresholds = [50, 20, 10, 5]  # percent, descending

[notifications]
max_visible       = 4
history_persists = false      # false = history lives only for this shell instance

[backend]
audio_service   = "auto"      # auto | wireplumber | pipewire
network_service = "NetworkManager"
brightness      = "auto"      # auto | sysfs | brightnessctl
night_light_cmd = "sct"       # external command; see §3
screenshot_cmd  = "grim"
powerprofiles   = "auto"
```

### 11.1 Config reload

Reloading must be graceful:

1. Capture current state and focus.
2. Morph through a brief reload state.
3. Apply the new config.
4. Restore the previous state if it is still valid under the new config.
5. Otherwise return smoothly to rest.

There must be **no hard visual pop** between old and new configuration.

### 11.2 Validation

- Unknown keys produce a **warning, not a crash**. The shell must always start.
- Invalid values (out-of-range opacity, unknown animation speed) fall back to the
  default and warn.
- TOML parse failure leaves the **previous, working config active**.

---

## 12. Theming

- **One QML file per theme** in `themes/`. Adding a theme means dropping in a new
  `.qml` file — nothing else to edit, no registry to update.
- A theme defines the **global visual language of the entire shell**. Themes do
  **not** define per-state styling; every state shares one visual language.
- **Bundled themes:** `rose_pine.qml` and `tokyo_night.qml` at minimum.
  `catppuccin_mocha`, `gruvbox`, and `nord` are expected additions.
- **Live switching.** Changing the theme must not require restarting Quickshell
  or Niri. The island transitions smoothly to the new theme rather than popping.
- **Primary interface:** `Settings → Appearance → Theme`. The selection persists
  to TOML.
- **Fallback:** edit TOML and trigger `reload-shell` if runtime loading proves
  unreliable.

### 12.1 Theme scope

A theme is a **pure style object plus a limited effects layer**.

A theme **may** define:

- Colours and gradients
- Typography (family, sizes, weights)
- Corner radii
- Border widths and colours
- Shadows and glow
- Blur / glass parameters
- Spacing and density
- Slider, progress, and toggle styling
- Visual effects explicitly supported by the shell

A theme **must not** define:

- Keyboard behaviour
- Section logic, or which sections exist
- Event handling or queueing
- System commands
- Layout or state logic
- Geometry or animation timing

This keeps themes safe: a theme can restyle the entire shell but can never
change its behaviour or break it.

### 12.2 Precedence

**The theme supplies visual defaults. `config.toml` overrides rendering.**

| Property | Theme | TOML override |
|---|---|---|
| Colours, typography, radii, borders, shadows, spacing | ✅ | — |
| Transparency, opacity, blur radius | ✅ (default) | ✅ |
| Animation speed | ✅ (default) | ✅ |
| Position, size caps | — | ✅ |

So Tokyo Night can run heavily transparent, or Rose Pine fully opaque, without
editing either file.

---

## 13. Icons

**Hybrid approach:**

- A **bundled, consistent icon set** for the shell's own core UI, so the island
  looks coherent regardless of the host system's theme.
- The **system icon theme** is a configurable fallback/override.
- **Application icons** come from the desktop entry's `Icon` key, resolved
  through the system icon theme.
- **Themes do not define individual icons.** Iconography is the shell's
  responsibility, separate from colour.

---

## 14. Lifecycle

### 14.1 Startup

The shell enters **resting state directly**. No splash, no loading animation, no
layout flash, no "welcome" state.

The island should appear once its first Niri state is available, and not before.
A one-frame delay is preferable to rendering an empty island that pops.

### 14.2 Reload

`reload-shell` (also available as a keybind and an IPC call) is
**state-preserving where practical**, with a graceful fallback to rest and a
smooth morph throughout. Never a visual pop.

This applies to reloading config, themes, and QML.

### 14.3 Hotplug and resilience

- **Monitor hotplug** must be handled without a restart; the island re-anchors to
  the focused monitor.
- **Service loss** (NetworkManager, BlueZ, PipeWire restarting) must degrade
  gracefully — the affected section disappears, the shell keeps running.
- **A missing D-Bus service** must not prevent startup.

---

## 15. IPC

Expose an IPC surface so the shell can be driven externally, allowing Niri
keybinds to reach **every** part of the shell:

```sh
qs -p ./shell ipc call island toggle
qs -p ./shell ipc call island open launcher
qs -p ./shell ipc call island open sysmon
qs -p ./shell ipc call island section next
qs -p ./shell ipc call island theme set tokyo_night
qs -p ./shell ipc call island reload
```

This is what makes the shell **composable with Niri** rather than dictating
interaction. Every interactive bind the shell defines should also be reachable
this way, so a user can remap everything in `config.kdl`.

---

## 16. Service interfaces

Each service is an adapter exposing a narrow, stable interface. UI code depends
only on these signatures, never on the transport.

### 16.1 `NiriService`

```qml
property list<Workspace> workspaces   // { ref, id, output, isFocused, active, windowCount }
property list<Output>    outputs      // { name, isFocused, x, y, width, height, scale, refreshRate }
property list<Window>    windows      // { id, title, appId, workspaceRef, isFocused }
property Output          focusedOutput
property Window          focusedWindow
function refresh()                    // re-query full state
function focusWorkspace(ref)         // niri msg action focus-workspace <ref>
function focusMonitor(name)
```

Backed by `niri msg --json` for queries and a long-lived
`niri msg --json event-stream` for change notification. The event stream is the
**primary** update path; polling is a fallback only.

### 16.2 Other services

| Service | Interface |
|---|---|
| `NotificationService` | `notifications`, `notify()`, `invokeAction()`, `close()`, `setDnd(bool)`, `clearAll()` |
| `MprisService` | `players`, `activePlayer` (title, artist, artwork, position, length, status), `playPause()`, `next()`, `previous()` |
| `AudioService` | `outputDevices`, `inputDevices`, `volume`, `muted`, `setVolume()`, `setOutput()`, `setInput()`, `setMuted()` |
| `BrightnessService` | `value`, `max`, `setValue()` |
| `NetworkService` | `enabled`, `activeConnection`, `ssid`, `networks[]`, `scan()`, `connect(ssid)`, `disconnect()` |
| `BluetoothService` | `enabled`, `powered`, `devices[]`, `scan()`, `pair(addr)`, `connect(addr)`, `disconnect(addr)` |
| `PowerService` | `battery`, `charging`, `profile`, `setProfile()`, `suspend()`, `lock()`, `logout()`, `reboot()`, `shutdown()` |
| `NightLightService` | `enabled`, `toggle()`, `setEnabled()` — wraps the configured external command |

Every service must:

- Degrade to "unavailable" rather than throw
- Expose an `available` property so dependent sections can disappear per rule 6
- Never block the UI thread

---

## 17. Widget library

Shared, theme-driven components. All of them respect the focus model (§7).

| Widget | Contract |
|---|---|
| `Dot.qml` | Renders one workspace in any of the three states (§5.2) |
| `Section.qml` | Declares `axis`, `priority`, `focusable`, and a relevance predicate |
| `List.qml` | Vertical list with `j`/`k` navigation, scroll, and 4-visible windowing |
| `ListItem.qml` | One row; shows focus; emits `activate` on Enter and on click |
| `Slider.qml` | `h`/`l` moves between, `j`/`k` adjusts, drag to scrub |
| `Toggle.qml` | On/off switch; `Enter` or click to flip; state animation |
| `Button.qml` | Activates on `Enter` or click |
| `Confirm.qml` | Two-step confirmation for destructive actions |

A section declares its properties rather than hard-coding layout behaviour:

```qml
Section {
    id: mediaSection
    axis: Island.Horizontal
    priority: 50
    focusable: true
    relevant: MprisService.hasActivePlayer   // absent ⇒ not in the cycle
}
```

---

## 18. Acceptance criteria

The shell is complete when all of the following hold.

### Resting

- [ ] The island runs at the top centre of the focused monitor, body above the
      screen edge, with only dots and `HH:MM` visible.
- [ ] Dots distinguish focused (large, bright, thin ring), occupied (brighter),
      and existing-empty (small, dim).
- [ ] Collapsed Niri workspaces are not rendered.
- [ ] Occupied and empty are distinguishable without relying on colour alone.
- [ ] No numbers, labels, battery, or network in the resting state.
- [ ] Startup goes straight to rest — no splash, no flash.

### Navigation

- [ ] The island key opens to Workspaces with the first item focused.
- [ ] `Tab` / `Shift+Tab` cycle sections and never alter the focused control.
- [ ] The cycle is cyclic, and skips sections whose data is absent.
- [ ] Workspaces leaves the cycle after `workspaces_relevance_ms` of inactivity,
      but the island key still lands there.
- [ ] `h`/`l` move between items in a section.
- [ ] `j`/`k` adjust a focused slider or move within a focused list, and
      **never** change sections.
- [ ] Arrow keys mirror `h`/`j`/`k`/`l`; the mouse mirrors the same model.
- [ ] Nothing requires the mouse.

### States

- [ ] Direct binds open their state and hold it until `Esc`.
- [ ] Transient events display and auto-collapse, and **never steal focus**.
- [ ] An urgent event interrupts an interactive state and, on completion,
      restores the previous state with the exact selection intact.
- [ ] Nested urgent events unwind in order.
- [ ] Simultaneous events queue, and each is readable in turn.
- [ ] Volume/brightness jump to the front, then the prior event resumes from
      where it was, not from the start.
- [ ] Repeated volume changes update one event in place — the queue never grows.

### Geometry and motion

- [ ] Transitions morph fluidly; an interrupted animation retargets from its
      current geometry with no snap.
- [ ] Vertical sections respect `max_height_percent`; horizontal sections respect
      `max_width_percent`.
- [ ] Each section expands on one axis.
- [ ] There is no hard pop between any two states, including across a reload.

### Sections

- [ ] Media does not exist when nothing is playing.
- [ ] Notifications does not exist when empty.
- [ ] Sysmon omits metrics the machine does not have, with no zero/N/A rows.
- [ ] The notification stack shows at most 4 and scrolls with `j`/`k`; `h`/`l`
      moves within the focused notification; only app-provided actions appear.
- [ ] DND suppresses normal notifications, still admits urgent ones, and reveals
      suppressed history when disabled.
- [ ] Notification history does not survive a shell restart.
- [ ] Settings controls Wi-Fi and Bluetooth fully in-shell.
- [ ] Audio has no per-app mixer; Displays are status-only; Night Light and DND
      are toggles only.
- [ ] Reboot and Shutdown confirm; Lock and Suspend do not.

### Launcher

- [ ] `Super+Space` focuses the search field immediately; results fold downward.
- [ ] Fuzzy matching with deterministic ranking — same query, same order.
- [ ] One unified ranked list with icons distinguishing entry types.
- [ ] Applications only in v1, behind a provider interface that permits future
      toggles and actions without UI changes.
- [ ] `Enter` acts and returns to rest; `Esc` closes.

### Config and theme

- [ ] One TOML file configures behaviour and reloads without a restart.
- [ ] A TOML parse error leaves the previous working config active; the shell
      still starts.
- [ ] Unknown keys and invalid values warn rather than crash.
- [ ] Themes are one QML file each, global-only, and switch live.
- [ ] TOML overrides theme transparency, opacity, blur, and animation speed.
- [ ] Adding a theme is a matter of dropping in one file.
- [ ] `reload-shell` restores valid state and morphs smoothly.

### Monitors

- [ ] Exactly one island, on the focused monitor; it transfers on focus change.
- [ ] Hotplug re-anchors without a restart.

---

## 19. Build order

Each stage should be runnable and demonstrable before starting the next.

| Stage | Deliverable | Verified by |
|---|---|---|
| 1 | `shell.qml`, island geometry, resting state, dots + clock, Niri service | Resting state matches §5 |
| 2 | State machine, section registry, `Tab`/`h`/`j`/`k`/`Enter`/`Esc`, morphing, interruptible animation | Navigation feels right |
| 3 | Event queue, priority, overlay interrupt + restore, volume/brightness | §10 behaviour |
| 4 | Notifications: D-Bus service, stack of 4, actions, DND | §8.4 |
| 5 | Launcher: fuzzy search, deterministic ranking, provider interface | §9 |
| 6 | Media + Sysmon: MPRIS, adaptive metrics | §8.2, §8.3 |
| 7 | Settings: Wi-Fi, Bluetooth, audio, displays, power, night light, DND, power menu | §8.5, §8.6 |
| 8 | Config + themes: TOML loader with hot reload, theme system, live switching, transparency/blur | §11, §12 |
| 9 | Mouse parity, multi-monitor, lifecycle, hotplug, resilience | §7.5, §5.6, §14 |

**Stages 1–9 are ordered so that the hardest-to-get-right work — the state
machine, the morphing, and the event queue — is validated early**, before
service integrations are layered on top.

### 19.1 Development note

Quickshell is a plain Wayland client, so **stages 1, 2, 3, 5, and 8 are fully
testable on any Wayland compositor**, including one that is not Niri. Only
`NiriService` (§16.1) requires a live Niri session. During early development a
temporary stub workspace source may stand in for it; this is a development aid
only and must not ship.

---

## Appendix A — decision record

Settled during design. Recorded so they are not reopened.

| Question | Decision |
|---|---|
| Shell architecture | One morphing island; no bar, dock, or dashboard |
| Compositor | Niri, driven via IPC; never duplicated |
| Second backend | **None** — Niri only |
| Resting content | Workspace dots + 24h clock, nothing else |
| Workspace representation | Dots, no numbers; mirrors Niri's collapsed dynamic workspaces |
| Dot states | Focused = large/bright/ring; occupied = brighter; empty = small/dim |
| Island position | Top centre, mostly hidden above the screen edge |
| Wakes on | Keybinds, top-edge hover, and system events |
| Expanded model | Morph per state; no fixed expanded size |
| Section navigation | `Tab` / `Shift+Tab` |
| Item navigation | `h` / `l` |
| Vertical control | `j` / `k`, contextual |
| Launcher | A direct state, not a section |
| Section cycle | Cyclic over relevant sections; Workspaces joins on activity |
| Open behaviour | Always lands on Workspaces |
| Interactive states | Stay open until `Esc` |
| Transient events | Auto-collapse, never steal focus |
| Event conflicts | Readable queue; volume/brightness jump the front and resume |
| Rapid volume changes | Update the existing event, never queue |
| Interruptions | Interruptible overlays that restore prior state |
| Notifications | Vertical stack, max 4, `j`/`k` scroll, `h`/`l` within |
| Notification history | Per shell instance only |
| DND | On/off toggle; urgent still interrupts |
| Notification actions | App-provided only, inside the island |
| Sysmon | Static snapshot + time; no graphs yet |
| Sysmon on absent hardware | Metric omitted entirely, never zeroed |
| Media | MPRIS; no volume control; only when playing |
| Settings | System controls; Wi-Fi/Bluetooth fully in-shell |
| Audio settings | Devices + master + mute; no per-app mixer |
| Displays | Status only |
| Night Light / DND | On/off toggles only |
| Power | Status in Settings; actions in a separate confirmed Power menu |
| Battery | Not in resting UI; threshold events at 50/20/10/5 % |
| Network events | Transient, filtered to avoid noise |
| Launcher ranking | Deterministic fuzzy; no learning |
| Launcher results | One unified ranked list, icons distinguish |
| Launcher scope | Apps now; provider interface for future toggles/actions |
| Config | Single TOML file |
| Themes | One QML file each; global styling; live switching |
| Theme scope | Pure style + limited effects; no behaviour |
| Transparency | TOML, overriding theme |
| Icons | Bundled core set, system theme as override |
| Monitors | Focused monitor only |
| Startup | Straight to rest |
| Reload | State-preserving, smooth morph, no pop |
| Size caps | ~40–45 % height vertical, ~40–50 % width horizontal |
| Size adaptation | Fixed percentages; not adapted per monitor |
| Axis rule | One axis per section, not both |
| Animation | Interruptible, retargeting, state-driven |
| Mouse | Mirrors keyboard; drag is horizontal-only; never required |

---

## Appendix B — reference

### B.1 Niri IPC

```
niri msg --json workspaces          # ref, id, output, is_focused, active, windows
niri msg --json outputs             # name, is_focused, logical{x,y,w,h,scale}, current_mode
niri msg --json windows             # id, title, app_id, workspace_id, is_focused, output
niri msg --json focused-window
niri msg --json focused-output
niri msg --json event-stream        # long-lived change feed
niri msg action focus-workspace <ref>
niri msg action focus-monitor <name>
```

The `active` field on a workspace distinguishes a live workspace from a
collapsed one (§3.3).

### B.2 D-Bus interfaces

| Purpose | Interface |
|---|---|
| Notifications | `org.freedesktop.Notifications` |
| Media | `org.mpris.MediaPlayer2.*` |
| Network | `org.freedesktop.NetworkManager` |
| Bluetooth | `org.bluez` |
| Power profiles | `net.duns..PowerProfiles` (`powerprofilesctl`) |

### B.3 External commands

Niri provides neither of these natively, so both are TOML-configurable and
invoked by the shell:

| Purpose | Candidates |
|---|---|
| Night light | `sct`, `gammastep`, `wlgamma`, `wlsunset` |
| Screenshot | `grim` (+ `slurp` for region select) |
| Brightness | `brightnessctl` (fallback for sysfs) |

### B.4 Keymap reference

| Keys | Meaning |
|---|---|
| `Super` | Open island → Workspaces |
| `Tab` / `Shift+Tab` | Next / previous section |
| `h` `l` / `←` `→` | Previous / next item in section |
| `j` `k` / `↓` `↑` | Vertical action on focused control |
| `Enter` | Activate |
| `Esc` | Collapse to rest |
| `Ctrl+C` | Cancel (text entry) |
