# Data sources — where the usage numbers come from

> **Status: research notes, checked 2026-11-18.** None of the three vendors offers a
> documented public API for subscription usage limits. Every source below is either
> undocumented or reads local files or processes, and **any of them can break without
> notice**. Keep each reader isolated behind one interface, mark its output with `source`
> and `dataAt`, and fail soft (`error` set, last value shown as stale).

All three products meter usage the same way: a **rolling 5-hour window** stacked on a
**weekly window**. That is why every agent is drawn with the same block (see `design/README.md`).

The readers run in the **collector on the dev computer** (see `architecture.md`), never on the
phone. Credentials stay on the computer; only percentages and timestamps leave it.

---

## Claude Code

**Primary — OAuth usage endpoint (undocumented).** This is the endpoint Claude Code's own usage
screen calls, and community statusline tools use it too.

```
GET https://api.anthropic.com/api/oauth/usage
Authorization: Bearer <oauth_access_token>
anthropic-beta: oauth-2025-04-20
```

The response holds `five_hour` and `seven_day` objects, each with `utilization` (0–100) and
`resets_at` (ISO 8601). There are also optional per-model buckets such as `seven_day_sonnet` and
`seven_day_opus`.

- **Token:** on Linux and Windows it is in `~/.claude/.credentials.json`; on macOS it is in the
  Keychain. The collector must read it locally and must never log or forward it.
- **Map to the contract:** `fiveHour.usedPercent = five_hour.utilization`, and
  `weekly.usedPercent = seven_day.utilization`.
- **Be polite:** cache the result and poll at most about once a minute.

**Alternative — statusline stdin.** Recent Claude Code versions pass `rate_limits.five_hour` and
`rate_limits.seven_day` to a custom statusline script on stdin for Pro and Max subscribers. A
statusline hook could write that JSON to a file for the collector to read, with no network calls.
The catch is that it only updates while a Claude Code session is open.

**Activity (`active`):** the most recent change time of files under `~/.claude/projects/**.jsonl`.

References:
- https://pypi.org/project/ccusage/ — documents the endpoint, the headers and the response shape
- https://github.com/ohugonnot/claude-code-statusline — the statusline stdin path

---

## Codex

**Primary — session logs.** Codex writes rollout files to `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`.
Its `token_count` events carry:

```json
"rate_limits": {
  "primary":   { "used_percent": 12.0, "window_minutes": 300,   "resets_at": 1765975828 },
  "secondary": { "used_percent": 35.0, "window_minutes": 10080, "resets_at": 1766562628 }
}
```

- `primary` is the 5-hour window and `secondary` is the weekly one. Identify them by
  `window_minutes` (≈300 and ≈10080) rather than by position.
- Older Codex versions wrote `resets_in_seconds` instead of `resets_at`. Support both, and
  compute the reset time from the event timestamp when only the relative value is present.
- **Staleness:** the logs only update while Codex is in use, so `dataAt` is the time of the
  event. The app shows `OLD 23M` once the data passes the stale threshold.
- Some plans have no 5-hour window. In that case set `fiveHour: null`, and the design falls
  back to showing the weekly percentage as the hero number.

**Activity:** the most recent change time of the newest rollout file.

**Alternative (later):** the ChatGPT backend's live usage endpoint, which some third-party tools
call with the token from `~/.codex/auth.json`. It is undocumented and token handling is riskier,
so only add it if the logs turn out not to be enough.

References:
- https://github.com/xiangz19/codex-ratelimit — a parser for the same session logs
- https://lin-guanguo.github.io/llm-memory-research/agent-cli/codex-session-file-schema/ — the session file schema

---

## Antigravity

**No public API.** Quota is **per model group** (for example Gemini Pro, Gemini Flash, and a
Claude/GPT group). Each group has its own 5-hour refresh and weekly limit, and the source reports
**remaining fraction** rather than used.

**Option A — local language server (what the community quota extensions do).** While the
Antigravity IDE is running, its language server process exposes a local HTTPS endpoint on
`127.0.0.1`. The collector finds the process, reads `--extension_server_port` and
`--csrf_token` from its arguments, and calls
`…/exa.language_server_pb.LanguageServerService/GetUserStatus`. The response includes the
remaining fraction and the reset time for each model.
- This is fragile: it depends on internal flags and it only works while the IDE is open.
- Map it as `usedPercent = (1 - remainingFraction) * 100`.
- Report the **group with the highest usedPercent** in `windows`, and put its name in `group`.

**Option B — Antigravity CLI.** `/usage` (alias `/quota`) is the officially supported way to
check quota. It is interactive, though, so it is not easy to automate. Look at it again if it
gains a non-interactive or JSON output.

**Do not** reuse Antigravity's own OAuth client to call Google's internal Cloud Code API from the
collector. It raises terms-of-service concerns.

References:
- https://antigravity.google/docs/plans?app=cli — the 5-hour refresh and the weekly limits per plan
- https://discuss.ai.google.dev/t/feature-request-official-read-only-quota-api-for-third-party-apps/188769 — the request for an official read-only API, and a pointer to CLI `/usage`
- https://skills.sh/sundial-org/awesome-openclaw-skills/antigravity-balance — the local `GetUserStatus` approach

---

## Reader checklist (every source)

- Return the contract shape from `shared/usage-snapshot.schema.json`.
- Never throw out of the reader. Set `error` instead, for example `"no_credentials"`,
  `"ide_not_running"` or `"parse_failed"`.
- Never put tokens, file contents or prompts into the snapshot or the logs.
- Unit-test every parser against fixture files in `collector/test/fixtures/`, including the
  older and newer field variants.
