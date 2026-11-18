# AGENTS.md — working on agent-usage

Instructions for coding agents (Claude Code, Codex, Antigravity, …) and humans working in
this repo. Read this first, then the file for your task:

| Task | Read |
|---|---|
| Anything visual | `design/README.md`, and open `design/preview/v2-minimal.html` |
| Data, networking, collector | `docs/architecture.md`, `docs/data-sources.md` |
| What to build next | `docs/roadmap.md` |
| JSON shape | `shared/usage-snapshot.schema.json`, `shared/examples/` |

## What this is

An **Android-only** app that turns the phone into an always-on style display showing the usage
limits (5-hour window and weekly window) of the coding agents in use: Claude Code, Codex and
Antigravity. It is amber pixel text on pure black, kept minimal.

- **Android only. Do not add iOS, Kotlin Multiplatform iOS targets, Flutter or React Native.**
- Android does not allow replacing the system AOD. The app provides a full-screen **Standby
  screen** and a **Screen saver** (DreamService) instead. See `docs/architecture.md`.
- The numbers come from a **collector** running on the dev computer (planned in `collector/`).
  The phone never holds agent credentials.

## Repo layout

```
design/      design spec, offline HTML preview, fonts (OFL), canvas sources (V2 current, V1 backup)
docs/        architecture, data sources, roadmap
shared/      JSON schema for the collector → app contract, example snapshots
android/     the app (created in milestone M0)
collector/   desktop daemon (milestone M3)
```

## Android stack

- Kotlin, Jetpack Compose (use the Compose BOM), Gradle Kotlin DSL with `gradle/libs.versions.toml`.
- Use the **latest stable** AGP, Kotlin and Compose BOM when scaffolding. Do not copy version
  numbers from old examples.
- A single `:app` module to start with. Package: `io.github.nguyentamgm.agentusage`.
- minSdk 26; compileSdk and targetSdk = latest stable.
- Libraries: kotlinx.coroutines and Flow, kotlinx.serialization, OkHttp (or the Ktor client),
  AndroidX DataStore, and Material 3 **for the settings screen only**.
- No DI framework at first; wire things by hand in an `AppContainer`. Add Hilt only if it starts
  to hurt.
- Tests: JUnit for `domain/`, and Compose screenshot tests (Roborazzi or Paparazzi) for the AOD states.

### Commands (run from `android/` once it exists)

```bash
./gradlew assembleDebug            # build
./gradlew testDebugUnitTest        # unit tests
./gradlew lint                     # Android lint
./gradlew installDebug             # install on a connected device
```

## The design rules that are easy to get wrong

`design/README.md` is the source of truth. The parts agents most often get wrong:

1. **Two colours:** `#E8A33D` on `#000000`. No greys, no opacity steps, no glow or shadow, no
   gradients, no rounded corners on bars.
2. **Fonts:** Press Start 2P for numbers and the clock, Silkscreen for text. Both are in
   `design/fonts/`; copy them into `res/font/` (keep the OFL licences). They are Latin-only, so
   **AOD copy is uppercase ASCII English** and is not localised. The normal settings screen can
   use the system font and be localised (English and Vietnamese).
3. **No footer, no dividers.** Every agent uses the same block: name and status, hero number
   with a sub-label, bar, then the weekly row.
4. **Only the status dot blinks** (hard on/off, 1.2 s period), and only for
   working/near-limit/limited. It must be possible to turn the blink off.

### Pixel-perfect text (must do)

Both fonts are drawn on **8 font-pixels per em**. Text is sharp only if one font-pixel is a
**whole number of device pixels**. Do not use plain `16.sp`/`32.sp`: density (e.g. 2.625) and
the user's font scale make the font-pixels fractional, and the result is blurry.

Lay out the AOD on a font-pixel unit `k` (device px). The reference frame is 412 × 915 CSS px,
which is **206 × 457.5 font-pixels** (one font-pixel of the 16-size text is 2 reference px):

```kotlin
// PixelGrid.kt (sketch)
@Composable
fun rememberPixelUnit(widthPx: Int, heightPx: Int): Int =
    remember(widthPx, heightPx) {
        maxOf(1, minOf(widthPx / 206, (heightPx * 2) / 915))  // whole device px per font-pixel
    }

// AOD ignores the user's font scale: it is a fixed pixel design.
CompositionLocalProvider(LocalDensity provides Density(LocalDensity.current.density, fontScale = 1f)) {
    val d = LocalDensity.current
    val text = TextStyle(
        fontFamily = Silkscreen,
        fontSize = with(d) { (8 * k).toFloat().toSp() },     // "16" in the design
        lineHeight = with(d) { (8 * k).toFloat().toSp() },
        letterSpacing = 0.sp,
        platformStyle = PlatformTextStyle(includeFontPadding = false),
        lineHeightStyle = LineHeightStyle(LineHeightStyle.Alignment.Center, LineHeightStyle.Trim.Both),
    )
    val hero = text.copy(fontFamily = PressStart2P, fontSize = with(d) { (16 * k).toFloat().toSp() }, lineHeight = /* same */)
}
```

- Express **every** spacing in units of `k`. Design values in reference px divide by 2:
  - padding top 88 → `44k`, sides 40 → `20k`, bottom 64 → `32k`;
  - gap 16 → `8k`, gap between agents 64 → `32k`;
  - bar 4 → `2k`, track dot 2 → `1k`, dot pitch 8 → `4k`, status dot 8 → `4k`.
- Content width = `166k`. Centre it horizontally and leave the remainder as side margin.
- Snap the bar fill width to whole `k`. Place things with integer px offsets
  (`Modifier.offset { IntOffset(...) }`), not fractional dp.
- Pixel shift (burn-in) moves the whole layout by whole multiples of `k`.
- Check on a real device: take a screenshot and zoom in. Each font-pixel must be a solid square
  with no anti-aliased fringe.

## Behaviour rules

- Status, ordering and formatting live in `domain/` as **pure Kotlin** with unit tests. The
  collector only sends facts (see the derivation table in `docs/architecture.md`).
- A `limited` agent goes first, its hero number becomes an `HH:MM:SS` countdown, and its status
  reads `BACK HH:MM`.
- Never show stale numbers as if they were fresh. After 15 minutes show `OLD 23M`.
- Poll only while the surface is visible. Tick the clock once a minute. Run a per-second ticker
  only while a countdown is shown.
- `StandbyActivity` keeps the screen on only while that is allowed by settings (default: while
  charging).

## Security and privacy

- **Never** put agent tokens (Claude OAuth, `~/.codex/auth.json`, Antigravity CSRF/OAuth) on
  the phone, in the snapshot, in logs or in git.
- The app stores only the collector address and the pairing token (DataStore). Do not log the
  pairing token.
- The data sources are undocumented (`docs/data-sources.md`). Keep readers isolated and
  fail-soft. Do not reuse a vendor's own OAuth *client* to call its internal APIs (see the
  Antigravity notes). Only use the user's own tokens with the endpoints described there.

## Working conventions

- Branch per milestone or feature (`m1-aod-screen`, …), small PRs, Conventional Commit messages.
- UI changes need screenshots in the PR (device and/or screenshot test), next to the matching
  state in `design/preview/v2-minimal.html`.
- Keep `design/README.md`, `docs/` and `shared/` in sync with the code. If you change the
  contract, update the schema, the examples and both sides in the same PR.
- Do not edit `design/source/*.dc.html` by hand. They are exports of the design canvas; ask the
  owner to update the canvas instead.
- Definition of done: builds, `lint` and unit tests pass, screenshot tests updated on purpose,
  docs updated.
