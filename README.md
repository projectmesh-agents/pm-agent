# pm-agent

PM template for [Project Mesh](https://github.com/projectmesh-io/mesh). Takes ambiguous asks and produces spec-ready work items. Runs autonomously via the Ralph Loop. Reachable from other mesh agents over the **peers** channel; optionally surfaces on Discord and WhatsApp.

## One-line run

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm
```

The PM is now running on your node, polling its work-item queue, writing specs, and handing off ready items to DevLead. **Peers is on by default** — other mesh agents can reach this PM at `peers://agent/pm-agent` or `peers://team/pm`. `mesh ps` shows it. `mesh attach pm` opens its live terminal.

## Add Discord

Pair a Discord bot once with the mesh node — the token is stored in `~/.mesh/discord-accounts/<account>/token.txt`, never in this template.

```bash
# Create a bot at https://discord.com/developers/applications
# Bot perms: View Channel, Send Messages, Read Message History.
# Invite it to your server.
mesh discord pair --account my-bot --token <bot-token>
```

Then run the PM with the discord channel enabled:

```bash
# One specific channel
mesh run --template github.com/projectmesh-agents/pm-agent --name pm \
  --set discord.enabled=true \
  --set discord.account=my-bot \
  --set discord.channelId=123456789012345678

# Whole-server (guild) mode — bot listens everywhere it has perms
mesh run --template github.com/projectmesh-agents/pm-agent --name pm \
  --set discord.enabled=true \
  --set discord.account=my-bot \
  --set discord.subscriptionMode=guild \
  --set discord.guildId=987654321098765432 \
  --set discord.requireMention=true
```

Find channelIds via `mesh discord list-channels --account my-bot`.

## Add WhatsApp

Pair your WhatsApp account once via QR scan (uses the WA Web protocol; same trust as a phone-linked device — the session DB lives in `~/.mesh/whatsapp-accounts/<account>/`):

```bash
mesh whatsapp pair --account personal
# scan the QR with WhatsApp → Settings → Linked Devices
```

Find chat JIDs (`<number>@s.whatsapp.net` for DMs, `<id>@g.us` for groups) via `mesh whatsapp list-chats --account personal`. Then:

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm \
  --set whatsapp.enabled=true \
  --set whatsapp.account=personal \
  --set whatsapp.chatJid=120363xxx@g.us
```

## Peers

Default-on. Other mesh agents on the same node reach this PM via:

- `mesh_send target="peers://agent/pm-agent" content="..."` — direct.
- `mesh_send target="peers://team/pm" content="..."` — team broadcast (members of the `pm` team).

Override teams: `--set peers.teams=[pm,planning]`. Disable entirely: `--set peers.enabled=false`. List active peers: `mesh peers list`.

## Common tweaks

Cheaper model:
```bash
mesh run ... --set model=claude-sonnet-4-6
```

Respond faster (15 s active poll, 60 s idle):
```bash
mesh run ... --set cadence.activeSeconds=15 --set cadence.idleSeconds=60
```

Allow-list discord users / roles:
```bash
mesh run ... --set discord.allowedUsers='["111","222"]'
mesh run ... --set discord.allowedRoles='["333"]'
```

## Use a values file

```yaml
# my-pm.yaml
model: claude-sonnet-4-6
discord:
  enabled: true
  account: my-bot
  channelId: "123456789012345678"
  requireMention: true
peers:
  teams: [pm, planning]
cadence:
  activeSeconds: 30
  idleSeconds: 180
```

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm \
  --values my-pm.yaml
```

No more `--env-file` — secrets live in the paired account dirs, not in this template.

## What the PM does (and doesn't)

**Does**:
- Polls `mesh work-item list --assigned-to-me` at its configured cadence.
- Moves items `inbox` → `clarifying` → `spec_ready` → hand off to DevLead.
- Writes specs in the Mesh spec format (outcome / scope / affected / criteria / open questions).
- Asks clarifying questions on ambiguous requests.
- Via Discord / WhatsApp / peers (optional): answers questions, scopes new asks, gives status updates.

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
| `discord.enabled` | `false` | Enable the Discord channel. |
| `discord.account` | `""` | Paired account name from `mesh discord pair`. |
| `discord.subscriptionMode` | `channel` | `channel` (one channelId) or `guild` (whole server). |
| `discord.channelId` | `""` | Required when `subscriptionMode=channel`. |
| `discord.guildId` | `""` | Required when `subscriptionMode=guild`. |
| `discord.requireMention` | `false` | Only respond when the bot is @-mentioned. |
| `discord.allowedUsers` | `[]` | Allow-list of Discord user IDs. Empty = anyone in the channel. |
| `discord.allowedRoles` | `[]` | Allow-list of Discord role IDs. Empty = anyone. |
| `whatsapp.enabled` | `false` | Enable the WhatsApp channel. |
| `whatsapp.account` | `""` | Paired account name from `mesh whatsapp pair`. |
| `whatsapp.chatJid` | `""` | DM JID (`<number>@s.whatsapp.net`) or group JID (`<id>@g.us`). |
| `peers.enabled` | `true` | Register on the local peers broker. Default on. |
| `peers.scope` | `machine` | Currently only `machine` (cross-node arrives in slice 4). |
| `peers.teams` | `[pm]` | Teams the PM joins; addressable as `peers://team/<name>`. |

## Updating

```bash
mesh update pm
```

Re-fetches the template git SHA and re-renders with the values that were in effect at `mesh run` time. The running agent restarts on the new version (same Claude conversation preserved).

## Files

- `values.yaml` — defaults for everything a user might tweak.
- `agent.yaml.tmpl` — template manifest; conditionally renders `channels:` based on `discord.enabled` / `whatsapp.enabled` / `peers.enabled`.
- `system_prompt.md.tmpl` — system prompt; varies based on which channels are enabled so the agent knows what tools to expect.

## Versioning

Tags follow semver. Breaking prompt / values changes bump the major. Pin with `--template github.com/projectmesh-agents/pm-agent@v0.2.0`.
