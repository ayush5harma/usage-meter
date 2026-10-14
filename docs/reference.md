# Technical reference

## Data sources

Usage Meter shows only tools that are installed and can report locally available usage data.

- **Claude Code:** the collector reads the selected Claude configuration, retrieves the matching OAuth credential from macOS Keychain, and refreshes the usage endpoint used by Claude Code. It can fall back to Claude Code's local session cache and marks the result accordingly.
- **Codex:** the collector calls Codex's local app server for account and rate-limit data. It does not make a model request.
- **Antigravity (`agy`):** `bin/agy-usage-statusline` receives agy's documented status-line JSON and writes quota-only fields to `~/.cache/usage-meter/agy-quota.json` with mode `0600`.

The app never accesses the network or Keychain directly. The collector never draws UI.

## Identities and cache

Claude configuration directories are considered in this order:

1. `USAGE_METER_CONFIG_DIRS`, if set, replaces discovery.
2. `CLAUDE_CONFIG_DIR`, if set.
3. `~/.claude` and matching `~/.claude-*` directories that hold Claude configuration.

`USAGE_METER_CONFIG_DIRS` accepts a colon-separated list. A value may be labelled with `label=path`, for example:

```sh
USAGE_METER_CONFIG_DIRS="work=$HOME/.claude-work:personal=$HOME/.claude"
```

The collector keeps its cache in `~/.cache/usage-meter` by default. Set `USAGE_METER_CACHE_DIR` to use a different location. Cache files can include account email, quota values, reset times, timestamps, and local usage history; they do not contain OAuth tokens. The collector creates its cache directory with mode `0700` and sensitive cache files with mode `0600`.

## Freshness and rate limits

The app asks the collector to refresh every 30 seconds. The collector avoids calling upstream on every poll: it uses a shorter TTL while the related CLI is active, a longer TTL while idle, and a backoff period after an error. Cached data is shown with its age and source. A stale reading is visibly distinct from a fresh one.

`USAGE_API_TTL` overrides the Claude usage TTL. `CODEX_USAGE_TTL` overrides the Codex TTL. `USAGE_METER_CODEX=0` omits Codex; `USAGE_METER_AGY=0` omits Antigravity.

## Antigravity setup

To show Antigravity quota, install the status-line bridge and configure agy to invoke it whenever its state changes:

```sh
mkdir -p "$HOME/.local/bin"
install -m 755 bin/agy-usage-statusline "$HOME/.local/bin/agy-usage-statusline"
```

Merge the following object into `~/.gemini/antigravity-cli/settings.json` while preserving any other settings:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.local/bin/agy-usage-statusline"
  }
}
```

Start an agy session after saving the setting. The bridge receives agy's documented status-line payload, prints the status line, and saves only quota percentages, reset times, model, and plan to the private Usage Meter cache. It does not read credentials or make network calls.

## Collector contract

`usage-meter-stats` prints one JSON object to standard output. `claude` and `ts` are always present. Tool keys such as `codex` and `agy` are included only when that tool is installed.

```json
{
  "claude": {
    "ok": true,
    "email": "account@example.test",
    "source": "api",
    "age_s": 8,
    "session": { "pct": 42, "reset_in": 7200, "severity": "normal" }
  },
  "ts": 1789210900
}
```

For every reported window, `pct` is a whole percentage, `reset_in` is seconds remaining (or `0` when unavailable), and `severity` is `normal`, `warning`, or `critical`. A reading with `ok: false` is intentionally shown as unavailable rather than as zero usage.

### Claude fields

| Field | Meaning |
| --- | --- |
| `ok` | Whether the collector has an honest reading to show. |
| `account`, `email` | The selected local identity label and its account email. |
| `source` | `api` for a live endpoint response or `session-cache` for Claude Code's local cache. |
| `age_s` | Seconds since the displayed reading was obtained. |
| `fetch_err`, `retry_in` | Why a live refresh is unavailable and when it will be retried. Present only when applicable. |
| `session`, `weekly`, `scoped` | Reported usage windows. `scoped_label` names the model-family window. |

### Tool fields

The optional `codex` and `agy` objects use the same `ok`, `source`, `age_s`, and `limits` concepts. Each `limits` entry has a server-supplied `label`, `pct`, `reset_in`, and `severity`; Codex entries may also report `duration_mins` and `cadence`.

Tool objects may also contain `email`, `plan`, `details`, `fetch_err`, and `retry_in`. A tool that is installed but cannot yet provide quota data returns a short `note`; a tool that is not installed is omitted entirely.

### Environment

| Variable | Purpose |
| --- | --- |
| `USAGE_METER_CONFIG_DIRS` | Replaces Claude configuration discovery with labelled or unlabelled paths. |
| `CLAUDE_CONFIG_DIR` | Adds Claude Code's selected configuration directory. |
| `USAGE_METER_CACHE_DIR` | Changes the collector cache location. |
| `USAGE_API_TTL`, `CODEX_USAGE_TTL` | Override Claude and Codex refresh TTLs. |
| `CODEX_HOME`, `GEMINI_HOME` | Change the local state locations used by Codex and agy. |
| `USAGE_METER_CODEX`, `USAGE_METER_CODEX_MODELS`, `USAGE_METER_AGY` | Set to `0` to omit the associated optional output. |

## App command line

| Command | Purpose |
| --- | --- |
| `UsageMeter` | Starts the menu-bar app. |
| `UsageMeter --run <program> [args…]` | Runs a child process under the app identity and exits with its status. |
| `UsageMeter --glyph <out.png>` | Renders a deterministic legacy gauge asset. |
| `UsageMeter --menu-preview <out.png>` | Renders the exact menu-bar geometry with fixed sample values. |

## Preview assets

The menu-bar images in `docs/menu-bar/` are deterministic assets. Regenerate the main preview with a scratch build:

```sh
APP_DIR=/tmp/usage-meter-preview bash build.sh --force
"/tmp/usage-meter-preview/Usage Meter.app/Contents/MacOS/UsageMeter" \
  --menu-preview docs/menu-bar/usage-meter-preview.png
```

`--menu-preview` uses fixed example values and does not start the collector, access a live account, read Keychain, or alter the installed app.
