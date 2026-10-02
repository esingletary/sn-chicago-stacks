# hermes

Two [Hermes Agent](https://github.com/NousResearch/hermes-agent) chief-of-staff
agents, one container each:

| Container    | Business           | Data dir      |
|--------------|--------------------|---------------|
| `hermes-srp` | Stockroom Projects | `data/srp/`   |
| `hermes-vc`  | Vibrant Chicago    | `data/vc/`    |

Separate containers (not Hermes profiles in one container) so memory,
sessions, credentials and resource limits never mix between businesses.
Never point two containers at the same data dir.

## Model: Claude subscription via the Claude Code CLI

Inference goes through the official
[`claude-subscription-directsdk`](https://hermes-agent.nousresearch.com/docs/plugins/claude-subscription-directsdk)
plugin, which runs each turn through the `claude` CLI on a Claude Pro/Max
subscription (no API key). The image is upstream `nousresearch/hermes-agent`
plus a pinned `@anthropic-ai/claude-code` (see `Dockerfile`).

- Model: `claude-sonnet-5` (`model.default` in `data/<agent>/config.yaml`).
  Sonnet 5.5 isn't in the plugin's model catalog yet (v0.3.0); switch
  with `hermes config set model.default <id>` once it is.
- CLI login lives in `data/<agent>/claude/` (`CLAUDE_CONFIG_DIR`), so each
  agent can be logged into a different Claude account. If both use the same
  account, they share its usage limits.
- Usage counts against the subscription's Agent SDK allowance.

## First-time setup (per agent)

```sh
# 1. Log the Claude CLI in as the runtime user (prints a URL; paste back the code)
docker exec -it -u hermes hermes-srp claude auth login
docker exec -it -u hermes hermes-srp claude auth status     # "loggedIn": true
# Always pass `-u hermes` to `claude`: plain `docker exec` runs as root and
# leaves data/<agent>/claude root-owned, so the agent's login can't be saved.

# 2. Connect a chat platform (Telegram/Slack/...), SOUL, tools
docker exec -it hermes-srp hermes setup

# 3. Smoke test
docker exec -it hermes-srp hermes chat -q "Say hi"
```

Repeat with `hermes-vc`. Already done for both: plugin installed + enabled,
`model.provider: claude-subscription-directsdk-experimental`,
`model.default: claude-sonnet-5`.

## Operations

```sh
docker compose up -d --build                  # after editing Dockerfile
docker logs -f hermes-srp                     # gateway output
docker exec hermes-srp hermes logs --follow   # agent.log / errors.log
```

Upgrade: bump `HERMES_TAG` / `CLAUDE_CODE_VERSION` in the `Dockerfile`, then
`docker compose up -d --build`. All state is under `data/`, the image is
stateless.

## Notes

- **No ports, no Caddy route.** Chat platforms connect outbound.
- **Dashboard is off.** Hermes refuses to bind it beyond loopback without
  auth. To enable: set `dashboard.basic_auth` in the agent's `config.yaml`,
  add `HERMES_DASHBOARD: "1"` and `ports: ["127.0.0.1:9119:9119"]` (9120 for
  vc), then reach it with `ssh -L 9119:localhost:9119`.
- **Limits:** 2 GB / 1.5 CPUs per agent, so a runaway agent can't starve
  Technitium (LAN DNS). No local browser: use a cloud browser
  (`BROWSER_USE_API_KEY` or `BROWSERBASE_*` in `data/<agent>/.env`), with
  a separate key per business.
- **Backups:** `data/` is covered by the nightly restic run of `/opt/stacks`
  (includes `.env` secrets and the Claude login). `state.db` is SQLite in WAL
  mode; for a guaranteed-consistent snapshot, stop the containers around the
  backup like Beszel.
- **SD card:** agents write constantly; move `data/` with the NVMe migration.
