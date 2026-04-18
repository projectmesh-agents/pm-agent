# pm-agent

PM template for [Project Mesh](https://github.com/projectmesh-io/mesh). Takes ambiguous asks and produces spec-ready work items. Runs autonomously via the Ralph Loop; optionally listens on Discord.

## One-line run

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm
```

That's it. The PM is now running on your node, polling its work-item queue, writing specs, and handing off ready items to DevLead. `mesh ps` shows it. `mesh attach pm` opens its live terminal.

## Add Discord with three flags

Get a bot token from https://discord.com/developers/applications, invite the bot to your server (Send Messages, Read Messages, View Channel), and grab the channel ID from Discord (enable Developer Mode → right-click channel → Copy ID).

```bash
DISCORD_BOT_TOKEN=your-bot-token \
mesh run --template github.com/projectmesh-agents/pm-agent --name pm \
  --set discord.enabled=true \
  --set discord.channelId=123456789012345678
```

The PM will connect to the channel, listen for messages, and reply to you inline. Allow-list users or roles with `--set discord.allowedUsers='["111","222"]'` or `--set discord.allowedRoles='["333"]'`.

## Common tweaks

Cheaper model:
```bash
mesh run ... --set model=claude-sonnet-4-6
```

Respond faster to changes (15 s active, 60 s idle):
```bash
mesh run ... --set cadence.activeSeconds=15 --set cadence.idleSeconds=60
```

Only respond when @'d in Discord:
```bash
mesh run ... --set discord.enabled=true --set discord.channelId=... --set discord.requireMention=true
```

Use a values file instead of flags:

```yaml
# my-pm.yaml
model: claude-sonnet-4-6
discord:
  enabled: true
  channelId: "123456789012345678"
  requireMention: true
  allowedRoles: ["987654321098765432"]
cadence:
  activeSeconds: 30
  idleSeconds: 180
```

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm --values my-pm.yaml
```

## What the PM does (and doesn't)

**Does**:
- Polls `mesh work-item list --assigned-to-me` at its configured cadence.
- Moves items `inbox` → `clarifying` → `spec_ready` → hand off to DevLead.
- Writes specs in the Mesh spec format (outcome / scope / affected / criteria / open questions).
- Asks clarifying questions on ambiguous requests.
- Via Discord (optional): answers questions, scopes new asks, gives status updates.

**Doesn't**:
- Write production code. Hand-offs to DevLead / Builder for that.
- Make architectural decisions in isolation — surfaces tradeoffs, lets the human / DevLead pick.
- Touch items past `ready_for_devlead` — downstream roles own those.
- Auto-approve its own specs without acceptance criteria.

## Values reference

All keys have defaults; override any subset.

| Key | Default | What it does |
| --- | --- | --- |
| `model` | `claude-opus-4-7` | Claude model. `claude-sonnet-4-6` is 20% the cost, still good. |
| `autonomous` | `true` | `--dangerously-skip-permissions` on the claude CLI. Keep true for unattended Ralph Loop. |
| `cadence.activeSeconds` | `60` | Poll cadence when items are in the queue. |
| `cadence.idleSeconds` | `300` | Poll cadence when queue is empty. |
| `discord.enabled` | `false` | Enable Discord channel. Requires `channelId` + `DISCORD_BOT_TOKEN`. |
| `discord.channelId` | `""` | Snowflake ID of the Discord channel. |
| `discord.tokenEnv` | `DISCORD_BOT_TOKEN` | Env var holding the bot token. |
| `discord.requireMention` | `false` | Only respond when the bot is @-mentioned. |
| `discord.allowedUsers` | `[]` | Allow-list of Discord user IDs. Empty = anyone in the channel. |
| `discord.allowedRoles` | `[]` | Allow-list of Discord role IDs. Empty = anyone. |
| `whatsapp.enabled` | `false` | Placeholder — ships when the WhatsApp channel-server lands. |

## Updating

To pull the latest template version without recreating the agent:

```bash
mesh update pm
```

That re-fetches the template git SHA and re-renders with the values that were in effect at `mesh run` time. If the template ships prompt or flow changes, the running agent restarts on the new version (same Claude conversation preserved).

## Files

- `values.yaml` — defaults for everything a user might tweak.
- `agent.yaml.tmpl` — template manifest; renders `channels:` and `requiredEnv:` based on `discord.enabled` / `whatsapp.enabled`.
- `system_prompt.md.tmpl` — system prompt; varies based on whether Discord is enabled so the agent knows to poll the Discord tools.

## Versioning

Tags follow semver. Breaking prompt / values changes bump the major. Pin with `--template github.com/projectmesh-agents/pm-agent@v0.1.0`.
