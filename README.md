# Agents

This is the `peti.agents` clone of the stock Omarchy plugin. It keeps only
four subscriptions on the panel — Claude Code, Codex, Grok, and OpenCode —
and skips the rest of the stock collectors.

Grok Build usage comes from `~/.grok/sessions/**/usage.json` (or
`$GROK_HOME/sessions`). Grok shows today's tokens, prompts and sessions, the
last seven days, and all-time model totals. The account allowance comes from
Grok's billing endpoint using the existing Grok Build sign-in in `auth.json`.
The plan label comes from the local settings cache. Open Grok Build to refresh
an expired sign-in.

OpenCode usage comes from the CLI's own database, and the OpenCode Go
allowance from the Console usage endpoint, reusing the Console sign-in the
CLI already has. Go shows the 5-hour, weekly, and monthly windows; the plan
line reads "Go".

Refresh manually with `omarchy-shell omarchy.agents refresh`; normal
refreshes use the panel's configured interval. The bundled collectors can
also be run directly with `bin/omarchy-agent-usage-grok` or
`bin/omarchy-agent-usage-opencode` (JSON output) or `--write` (update the
panel's usage record).

One bar icon and one panel for every AI coding subscription on the machine.
The panel is strictly a display: it watches the usage records that
`omarchy-agent-usage-update` writes to `~/.local/state/omarchy/agents/usage/`
and draws whatever appears there. `Panel.qml` owns the bar button and the
popup; `Main.qml` discovers and watches the records (and handles the optional
cross-device aggregation); `Agent.qml` is the per-record file watcher.

## Panel

- **Hero** — the mark, the tool, and the plan it runs on ("Max 20x", "Pro").
  Auth and endpoint problems replace the plan line and repeat in a card.
- **Subscription switch** — one chip per enabled agent (`h`/`l` or click).
  It appears only when more than one agent is enabled.
- **Limits** — the percentage of each allowance used, a matching meter, and
  the time until the session or weekly window resets.
- **Balance** — prepaid agents report a credit ledger instead of limits:
  remaining credit, a fuel-gauge meter that drains toward empty, and
  funded-versus-spent detail.
- **Tokens by day** — one row per day for the last week: day, bar, tokens, with today
  bolded at the bottom. Hover today for its prompt and session count.
- **Tokens by model** — tokens per model with the bar behind each row scaled
  to the heaviest model,
  the same way the weekly chart scales to its busiest day. Hover for the
  input / output / cache split.

Only Claude Code, Codex, Grok, and OpenCode are considered; every other record
the stock collectors write is ignored. A subscription appears only when it is
enabled in settings and has actually recorded usage — on this machine or on a
synced one. With one such agent
there is no switch row at all; with none, the module leaves the bar entirely
rather than sitting there with nothing to say. A CLI installed mid-session
shows up at the next refresh, so nothing polls the disk waiting for it.

That self-hiding is why the widget ships in the default bar layout: a machine
that has never run an AI coding agent draws nothing, and the icon arrives on
its own the first time a scan finds usage. Drop it with
`omarchy plugin disable omarchy.agents`.

## Data

Each agent is one JSON record in `~/.local/state/omarchy/agents/usage/`,
written by `omarchy-agent-usage-update`. That command runs one
`omarchy-agent-usage-<agent>` collector per agent; the widget invokes it
on its refresh timer and whenever you ask for a refresh, and picks up any
record that lands in the directory regardless of who wrote it.

Adding another agent means allowlisting it in `Main.qml` and shipping a
collector that prints the record contract (see the `claude` and `codex`
collectors in `bin/`). An `assets/<id>.svg` mark is optional —
with an `assets/<id>-light.svg` twin if the mark needs a dark variant for
light surfaces — and the bar glyph stands in when there is none.

| Collector | Limits | Local stats |
|---|---|---|
| `claude` | Anthropic's OAuth usage endpoint (5-hour session + 7-day weekly) | `~/.claude/projects` transcripts, opencode sessions on an Anthropic provider, plus `stats-cache.json` and `history.jsonl` as fallback |
| `codex` | The Codex app-server RPC | native Codex CLI session files (plus pi and opencode sessions) |
| `grok` | Grok's billing endpoint | `~/.grok/sessions` usage records |
| `opencode` | The Console usage endpoint: 5-hour, weekly, and monthly Go allowances | `~/.local/share/opencode/opencode.db` messages on the Console providers (both storage generations) |

Claude limits need a signed-in CLI; without credentials the panel says so and
falls back to local stats only. A non-default Claude directory is honored via
`CLAUDE_CONFIG_DIR`, Codex via `CODEX_HOME`, Grok via `GROK_HOME`, and
OpenCode's database via `OPENCODE_DB`.

### OpenCode Go

The collector probes `https://opencode.ai/zen/go/v1/usage` with the Console
sign-in OpenCode already stores, so a machine that can use Go in OpenCode
shows its allowance without a second login. The token is read, never
refreshed: OpenCode rotates its own, and a refresh here would sign the CLI
out. Sources are tried in order until one answers — an `opencode-go` then
`opencode` key in `~/.local/share/opencode/auth.json`, the Console credential
in the CLI's database, then `OPENCODE_API_KEY`.

Only the `opencode` and `opencode-go` providers count as local usage; a
session that ran on another provider belongs to that provider's own tab.
`opencode` covers the pay-as-you-go and free models, so a machine without a
Go subscription still gets today's tokens, the week, and the all-time model
breakdown, just no limits. A 401 says the sign-in expired, a 403 that the
sign-in has no Go subscription, and a failed probe keeps the last allowance
on screen while the panel retries.

## Interactions

- Bar icon: left = panel, right = launch agent, middle = next subscription.
- Panel: `h`/`l` switch subscription, `j`/`k` scroll, `r` or Enter refresh,
  Tab moves to the neighboring bar panel, Esc closes.
- IPC: `omarchy-shell omarchy.agents <open|close|toggle|refresh|next>`.

## Settings

Settings live in the widget's entry in `~/.config/omarchy/shell.json`. The
top-level keys can be set with
`omarchy bar set omarchy.agents <key> <value>`:

| Key | Default | What it does |
|---|---|---|
| `refreshIntervalSec` | `900` | How often the usage records regenerate |
| `syncMode` | `"Off"` | `"On"` writes this machine's snapshot and merges the others |
| `syncDir` | `""` | A folder synced by Syncthing, Dropbox, rsync, … |
| `syncFileName` | `<hostname>.json` | This machine's snapshot file |
| `syncDeviceId` | hostname | Stable device name inside the snapshot |

Numbers need `--json`, or they land in `shell.json` as strings:

```bash
omarchy bar set omarchy.agents refreshIntervalSec 300 --json
omarchy bar set omarchy.agents syncDir '~/Sync/agent-usage'
```

Per-agent enablement is nested, and `set` writes its key literally rather
than walking a dotted path — so pass the whole `providers` object as JSON (or
edit `shell.json` directly):

```bash
omarchy bar set peti.agents providers '{
  "claude": { "enabled": true },
  "codex": { "enabled": true },
  "grok": { "enabled": true },
  "opencode": { "enabled": true }
}' --json
```

`enabled` defaults to `true` for every discovered agent; set it to `false` to
hide a subscription that is installed. Disabled agents are also skipped when
the records regenerate.

With `syncMode` on, every `*.json` snapshot in `syncDir` is merged, so today,
the last 7 days, and the all-time totals cover every machine you code on —
active days are unioned by date rather than summed. Rate limits stay
per-account and are never merged. A record may declare `"scope": "account"`
when its stats are account-global rather than machine-local (Fireworks'
billing API); those merge by taking the widest value instead of summing, so
the same account synced from two machines is not counted twice.

One caveat on "all-time": the Codex collector only reads native session files
touched in the last 30 days, and Fireworks requests the last 30 days from its
billing API, so their totals and day counts cover that window. Claude's cover
every transcript still on disk.
