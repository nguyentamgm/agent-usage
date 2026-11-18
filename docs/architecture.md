# Architecture

## The constraint that shapes everything

The usage numbers live on the **dev computer**, where Claude Code, Codex and Antigravity run
and keep their tokens and logs. The display lives on an **Android phone**. The phone must never
hold those credentials. So the system has two parts:

```
 Dev computer (macOS / Linux / Windows)                 Android phone
┌───────────────────────────────────────┐            ┌──────────────────────────────┐
│ Claude Code ─┐                        │            │  app (Kotlin + Compose)      │
│ Codex ───────┼─> readers ─> collector │  LAN HTTP  │   • Standby screen (Activity)│
│ Antigravity ─┘     (per agent)  :8787 │ ─────────> │   • Screen saver (Dream)     │
│                                       │  JSON only │   • Settings                 │
│  tokens + logs stay here              │  no tokens │  polls every 60 s while shown│
└───────────────────────────────────────┘            └──────────────────────────────┘
```

- **Collector** (`collector/`, planned). A small daemon that runs the per-agent readers from
  `docs/data-sources.md` and serves `GET /v1/usage`, which returns a `UsageSnapshot`
  (`shared/usage-snapshot.schema.json`). Suggested stack: TypeScript on Node ≥ 20, started
  with `npx`. It is not part of the Android build.
- **Android app** (`android/`, to be created). Only Android; there is **no iOS target**.

## Contract

`shared/usage-snapshot.schema.json` is the single source of truth. `shared/examples/*.json`
are fixtures that the app's mock source and tests both use.

- The collector reports **facts**: percentages, reset times, activity, data age and errors.
- The app **derives status** (`working`, `idle`, `near_limit`, `limited`, `stale`, `unknown`),
  works out ordering and formats text. Keeping this logic in one place, the app, makes it testable.

### Status derivation (app)

```
if no data ever                          -> unknown
else if now - dataAt > staleAfter (15m)  -> stale
else if any window.usedPercent >= 100    -> limited   (countdown to that window's resetsAt)
else if fiveHour.usedPercent >= nearAt   -> near_limit  (nearAt default 85, setting)
else if active                           -> working
else                                     -> idle
```

When `resetsAt` passes, treat that window as 0 % until fresh data arrives. Do not keep showing
`limited` past the reset.

Ordering: `limited` agents first (soonest reset first), then the user's order.

## Transport

**MVP: polling over the LAN.**
- The phone calls `http://<computer-ip>:8787/v1/usage` every 60 s, but only while the screen is
  visible (`repeatOnLifecycle(STARTED)`).
- Pairing: the collector prints a random token on its first run, and the user types it into the
  app. Every request sends `Authorization: Bearer <pairing-token>`, and the collector rejects
  anything without it.
- Plain HTTP on a private network needs a `network_security_config.xml`. Keep cleartext as
  narrow as Android allows, and document why.
- Discovery (later): advertise `_agentusage._tcp` over mDNS from the collector and discover it
  with `NsdManager`.

**Later: off-LAN.** Push snapshots through a relay (for example a self-hosted ntfy topic, or a
tiny cloud function) so the phone works away from home. Snapshots contain no secrets, but keep
the topic private.

## Android app structure (target)

```
android/
  app/src/main/java/io/github/nguyentamgm/agentusage/
    MainActivity.kt              settings + "Start standby" entry
    standby/StandbyActivity.kt   full-screen always-on surface
    dream/UsageDreamService.kt   screen saver (charging / docked)
    ui/aod/AodScreen.kt          the design (one composable used by both surfaces)
    ui/aod/AgentBlock.kt         one agent block (name/status, hero, bar, weekly row)
    ui/aod/PixelBar.kt           4-unit bar with dotted track
    ui/aod/PixelGrid.kt          the font-pixel unit `k` and text styles (see AGENTS.md)
    ui/settings/…                Material 3 settings screen (normal phone UI)
    domain/StatusRules.kt        status derivation, ordering, formatting (pure Kotlin, unit-tested)
    data/model/…                 UsageSnapshot, AgentUsage, UsageWindow (kotlinx.serialization)
    data/source/UsageSource.kt   interface: Flow<UsageSnapshot?>
    data/source/MockUsageSource.kt      reads shared/examples/*.json (copied to assets)
    data/source/CollectorUsageSource.kt polls the collector
    data/settings/SettingsStore.kt      DataStore: address, token, agents, tint, blink, brightness
  app/src/main/res/font/press_start_2p.ttf, silkscreen.ttf   (from design/fonts, OFL)
```

## Display surfaces

Android does **not** let third-party apps draw on the system always-on display. The app gets as
close as it can with two surfaces that share `AodScreen`:

1. **Standby screen.** A full-screen `StandbyActivity` that keeps the screen on, shows over the
   lock screen, hides the system bars, uses low brightness and a pure black background. The user
   opens it from the app, from a Quick Settings tile (later) or from a home-screen shortcut.
   Setting: *keep on only while charging*. When unplugged, release `FLAG_KEEP_SCREEN_ON` so the
   phone can sleep normally.
2. **Screen saver.** `UsageDreamService`, which the user picks under Settings → Display → Screen
   saver. It is shown while charging or docked (exactly when depends on the OEM).

## Power and burn-in

- On OLED, black pixels are off. Keep lit pixels to a minimum, and never use a grey background.
- **Pixel shift:** move the whole layout by a few font-pixels in a cycle every 60 s, staying
  inside the safe margin. Always shift by whole font-pixel units so the text stays crisp.
- Recompose rarely:
  - the clock ticks on minute boundaries;
  - the per-second countdown runs only while some agent is `limited`;
  - the blink is a coroutine that toggles a boolean every 600 ms, not a 60 fps animation;
  - the blink can be turned off in settings.
- Network polling stops when the surface is not visible.
