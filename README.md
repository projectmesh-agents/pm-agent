# pm-agent

A PM agent template for [Project Mesh](https://github.com/projectmesh-io/mesh). Takes ambiguous asks and produces spec-ready work items. Operates autonomously via the Ralph Loop.

## What it does

- Periodically checks `mesh work-item list --assigned-to-me`.
- For items in `inbox` / `clarifying`: scopes them, writes specs, asks clarifying questions, updates status.
- For items in `spec_ready`: verifies acceptance criteria, hands off to DevLead.
- Stops there — downstream phases are handled by other agent roles.

Does NOT write production code. Scopes, specs, clarifies, hands off.

## Quickstart

On a machine with the `mesh` CLI and a running `mesh node start`:

```bash
mesh run --template github.com/projectmesh-agents/pm-agent --name pm
mesh attach pm          # watch it work
```

Or pin to a specific version:

```bash
mesh run --template github.com/projectmesh-agents/pm-agent@v0.1.0 --name pm
```

Once the PM agent is running, assign work to it by creating work items with `owner_role=pm` and `current_owner=<agent-name>` (or `pm`):

```bash
mesh work-item create "Rework auth flow for CLI" project <project-id> --owner-role pm
```

The agent's next tick will pick it up.

## Configuration

### Model / tools

Defined in `agent.yaml`. The defaults are:
- `claude-opus-4-7`
- `Read, Write, Edit, Bash, Grep, Glob, WebFetch, Task`

Change the model (e.g. to `claude-sonnet-4-6` for cheaper scoping) by editing `agent.yaml` in your fork.

### Dangerously-skip-permissions

Set to `true` so the agent operates unattended. The agent's role is scoping, not production changes, so the blast radius of tool calls is low — it reads code, runs `mesh work-item` commands, and writes specs. If you prefer to review each permission prompt, set it to `false`.

### Add Discord

To let the PM take clarifying questions over Discord, add a channel block to `agent.yaml` in your fork:

```yaml
spec:
  # ... existing fields ...
  requiredEnv:
    - DISCORD_BOT_TOKEN
  channels:
    - type: discord
      config:
        channelId: "YOUR_DISCORD_CHANNEL_ID"
        tokenEnv: DISCORD_BOT_TOKEN
        inboundPrefix: user
        # Optional allow-list: only these users/roles can talk to the PM
        allowedUsers: []
        allowedRoles: []
        requireMention: false
```

Then:

```bash
DISCORD_BOT_TOKEN=… mesh run --template <your-fork> --name pm
```

The PM's system prompt already tells it to periodically call `discord_receive_next` and reply via `discord_send_message`, so Discord flow works out of the box once the channel is configured.

## Files

- `agent.yaml` — template manifest (model, tools, channels).
- `system_prompt.md` — the PM's role, tools, rules, and the Ralph Loop polling loop it follows.

## Versioning

Tags follow semver. Breaking prompt changes (e.g. new required tools, changed workflow rules) bump the major version. `mesh run --template <repo>@v0.1` pins to `v0.1.*`.

## Contributing

Fork, edit `system_prompt.md`, test against a throwaway project, tag a release. A pattern that works well: keep the base template minimal and use agent-specific forks for heavy customization.
