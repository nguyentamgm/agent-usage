# Roadmap

Build the screen first with mock data, then the always-on surfaces, then real data. Each
milestone should be a mergeable PR (or a few) that meets its acceptance criteria.

## M0 — Android scaffold
- Create `android/` with Gradle (Kotlin DSL and a version catalog) and a single `:app` module.
  Package `io.github.nguyentamgm.agentusage`, Jetpack Compose, minSdk 26, compile/target SDK =
  latest stable.
- Copy the fonts from `design/fonts/` into `res/font/`, with their OFL licences in the app's
  open-source notices.
- Add CI (GitHub Actions) that runs `./gradlew lint testDebugUnitTest assembleDebug`.

**Done when:** a debug APK builds in CI and the app opens a black screen with one line of
amber pixel text.

## M1 — The design, pixel-perfect, with mock data
- Build `PixelGrid` (the font-pixel unit `k` and text styles), `AgentBlock`, `PixelBar` and
  `AodScreen`, exactly as `design/README.md` specifies.
- `MockUsageSource` loads `shared/examples/overview.json` and `limit-reached.json`. Add a debug
  menu to switch between them, plus a synthetic "near limit" case.
- `StatusRules` covers status derivation, ordering, countdown and the formats `2H07M`,
  `WED 18 NOV 2026` and `BACK 15:55`, with unit tests for every row of the state table.
- Add screenshot tests (Roborazzi or Paparazzi) for the three design states.

**Done when:** screenshots side by side with `design/preview/v2-minimal.html` match in layout,
text is crisp when zoomed in on a real device (no fractional font-pixels), and the tests are green.

## M2 — Always-on surfaces
- `StandbyActivity`: keeps the screen on, shows when locked, immersive, low brightness, exits
  with a double tap, and the "only while charging" setting works.
- `UsageDreamService` hosts a Compose screen saver. The dream needs its own lifecycle owners
  for `ComposeView`.
- Pixel shift every 60 s. Minute-aligned clock. The blink can be turned off.

**Done when:** the screen runs overnight on a charger with no crash, no visible burn-in shift
pattern, and a battery drain you can live with (record it in the PR).

## M3 — Collector MVP (on the dev computer)
- `collector/` (TypeScript, Node ≥ 20) serves `GET /v1/usage` on a configurable port, with a
  pairing token.
- Readers for **Claude Code** (OAuth usage endpoint) and **Codex** (session logs), following
  `docs/data-sources.md`, with fixture-based tests.
- Output validates against `shared/usage-snapshot.schema.json`.

**Done when:** `npx` starts the collector and `curl -H "Authorization: Bearer …"` returns live
numbers for both agents.

## M4 — App ↔ collector
- `CollectorUsageSource`: polls every 60 s while visible, uses exponential backoff on errors,
  and the stale state shows `OLD 23M`.
- Settings: collector address, pairing token, agent visibility and order, near-limit
  threshold, amber tint (4 presets), blink on/off, brightness.
- `network_security_config.xml` for HTTP on the LAN.

**Done when:** the phone shows real Claude Code and Codex usage, recovers after the computer
sleeps, and never shows a stale value without the stale state.

## M5 — Antigravity (experimental)
- Local language-server reader (Option A in `data-sources.md`), behind a feature flag in the
  collector. Report the busiest model group in `group`.

## Later / ideas
- mDNS discovery (`NsdManager`) instead of typing the IP address.
- Off-LAN relay push.
- Quick Settings tile to start Standby.
- A 4th agent: extend the `id` enum in the schema, add a reader, and the UI needs no change.

**Out of scope:** iOS, and replacing the system AOD (not possible for third-party apps).
