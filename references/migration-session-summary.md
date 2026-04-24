# Migration Session Summary

This reference summarizes the concrete session that produced the `openclaw-to-hermes-migration` skill.

## Source State

- OpenClaw home: `/root/.openclaw`
- Hermes home: `/root/.hermes`
- Workspace: `/root/.openclaw/workspace`
- OpenClaw size: about `520M`
- Final frozen file count: `7011`
- Workspace files: `2781`
- Main session JSONL logs after gateway stop: `128`
- Global skills: `19`
- Workspace skills: `4`
- Cron jobs: `1`
- Channels: `feishu`
- Providers: `zai`, `zenmux`

## Main Decisions

- Chose strict migration rather than online migration.
- Stopped `openclaw-gateway.service` before final checksum capture.
- Installed Hermes with the official script.
- Treated installer no-TTY setup wizard failure as non-blocking after `hermes doctor` passed.
- Ran dry-run before migration.
- Used `--preset user-data` first to avoid secrets migration.
- Used `--overwrite` to replace fresh Hermes template `SOUL.md`, model config, and workspace `AGENTS.md`.
- Used `--skill-conflict rename` to preserve duplicate skills.
- Later migrated allowlisted secrets and manually configured z.ai.
- In this specific session, removed ZenMux because the user explicitly requested it; this is not a general migration requirement.
- Migrated Feishu credentials to Hermes.
- Installed Hermes gateway as a background user service.

## Commands That Mattered

Freeze:

```bash
systemctl --user stop openclaw-gateway.service
lsof +D /root/.openclaw
find /root/.openclaw -type f -print0 | sort -z | xargs -0 sha256sum > /tmp/openclaw-migration-20260424/source-sha256.txt
```

Install:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes doctor
```

Dry-run:

```bash
hermes claw migrate --source /root/.openclaw --workspace-target /root/.openclaw/workspace --preset user-data --skill-conflict rename --overwrite --dry-run
```

Execute:

```bash
hermes claw migrate --source /root/.openclaw --workspace-target /root/.openclaw/workspace --preset user-data --skill-conflict rename --overwrite --yes
```

z.ai smoke test:

```bash
timeout 90 hermes chat -q '只回复 OK' --provider zai -m zai/glm-5-turbo -Q --max-turns 1 --ignore-rules
```

Gateway:

```bash
hermes gateway install
hermes gateway start
hermes gateway status
journalctl --user -u hermes-gateway -f
```

## Problems Encountered

- OpenClaw kept writing runtime files until `openclaw-gateway.service` was stopped.
- Migration dry-run reported stale OpenClaw PIDs; independent process and lsof checks showed no real writer.
- Hermes installer exited non-zero when setup wizard could not open `/dev/tty`, but installation was otherwise complete.
- `hermes config get` did not exist; actual CLI used `config show`, `config set`, `config check`.
- Provider keys were not fully bound after first migration. z.ai required explicit `ZAI_API_KEY` and `${ZAI_API_KEY}` references in config.
- Provider cleanup should be target-specific. Do not remove an old provider unless the user requests it or the provider is known to be obsolete for that environment.
- Feishu gateway connected but initially did not reply because the sender was not in `FEISHU_ALLOWED_USERS`.
- Feishu app also lacked `im:chat:readonly` or equivalent chat read scope, which showed as access denied in logs.
- Cron was archived, not recreated automatically. It was recreated as paused with local delivery.

## Final Verified State

- `hermes doctor` recognized `✓ Z.AI / GLM`.
- z.ai model call returned `OK`.
- Feishu gateway connected to `msg-frontier.feishu.cn`.
- `hermes-gateway.service` was active and enabled.
- `openclaw-gateway.service` remained inactive.
- Imported skills were visible under `openclaw-imports`.
- Cron job existed in Hermes but was paused.
