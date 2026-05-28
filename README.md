# Planet Integration Demo for NemoClaw

A NemoClaw / OpenClaw integration for the [Planet Insights Platform](https://docs.planet.com/develop/apis/) — search Planet's satellite imagery catalog, estimate tasking costs, check satellite pass availability, view account quota, and download thumbnails, all driven from your NemoClaw agent through a Tier 1 host-side API proxy.

> **Tier 1 security:** your Planet API key stays on the host. The OpenShell sandbox only ever sees a proxy URL on the host's LAN. The proxy injects the key, enforces an order-creation blocklist, and forwards to `api.planet.com`. This is the whole reason we use the NemoClaw / OpenShell architecture — keys never enter the sandbox.

## Quick start

```bash
git clone https://github.com/tklawa-nvidia/planet-integration-demo.git
cd planet-integration-demo
./install.sh <sandbox-name>
```

If you omit `<sandbox-name>` the script reads the default sandbox from `~/.nemoclaw/sandboxes.json`. The script will prompt for your Planet API key (get one at https://www.planet.com/account/#/user-settings) and save it to `~/.nemoclaw/credentials.json` with `0600` permissions.

After install, restart the OpenClaw TUI so the gateway re-reads `openclaw.json` and picks up the new skill, then try:

- "What satellite imagery types does Planet offer?"
- "Search for clear imagery over San Francisco from last month"
- "How much would it cost to task a satellite over the Pentagon?"
- "What's my Planet quota?"

See [`planet-integration-guide.md`](./planet-integration-guide.md) for the full reference: architecture, file layout, prompt examples, CLI usage, troubleshooting, and a `policy/planet.yaml` template.

## Changes vs. upstream

This is a republished, fixed snapshot of [`brevdev/nemoclaw-demos/planet-integration-demo`](https://github.com/brevdev/nemoclaw-demos/tree/main/planet-integration-demo) with two `install.sh` fixes:

1. **Policy injection now nests under `network_policies:`.** Upstream's `if/else` had two identical branches, so it always appended a 2-space-indented `planet_proxy:` block to the end of the policy YAML. If the existing policy had no `network_policies:` key (or didn't end with one), the YAML parser would attach `planet_proxy:` to whichever top-level block came last (typically `process:`) and `openshell policy set` would reject it with `unknown field planet_proxy, expected run_as_user or run_as_group`. This fork adds a top-level `network_policies:` key when it's missing.

2. **`set -e` no longer aborts on a "no" answer to the update-key prompt.** Upstream used `[[ regex ]] && UPDATE_KEY=true`, which returns non-zero when the regex doesn't match. Combined with `set -euo pipefail` at the top of the script, answering "n" to `Update API key? (y/N):` silently killed the script. This fork uses an explicit `if/then/fi`.

## Commands

| Command | Description |
|---|---|
| `./install.sh` | Install (interactive sandbox + key prompt) |
| `./install.sh my-sandbox` | Install against a specific sandbox |
| `./install.sh --update-key` | Force re-prompt for the Planet API key |
| `./install.sh --port 9202` | Use a custom host proxy port |
| `./install.sh --status` | Show current install + proxy state |
| `./install.sh --uninstall` | Stop proxy, remove skill files, drop policy block, clean local files |

## Compatibility

Supported OpenClaw layouts (auto-detected):

- **New** — `/sandbox/.openclaw/` (openshell ≥ 0.0.44, openclaw ≥ 2026.5.x).
- **Legacy** — `/sandbox/.openclaw-data/` (older OpenShell builds).

Re-running `./install.sh` is safe and idempotent — it re-detects the layout, replaces the `planet_proxy` policy block if the host IP changed, and restarts the proxy.
