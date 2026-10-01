# Takibi for Claude

Takibi is an agent-first knowledge base: humans curate projects, folders, and
uploads, and agents consume them through scoped keys with extractive,
cited answers. This plugin teaches Claude how to work with Takibi: asking
questions against the evidence, searching documents and agent notes, reading
converted document text, and driving Takibi task boards (list, claim, move,
and attach artifacts) — all through the Takibi CLI with the user's own key.

## How it works

The plugin contains a single skill. It carries no code that runs on install
and connects to nothing by itself. When the skill is active, it instructs
Claude to run the `takibibase` CLI (via `npx`, zero-install) in the agent's
own shell. The CLI reads the user's scoped key from `~/.takibi/key` and talks
to Takibi's hosted API at `https://app.takibibase.com` (overridable with
`$TAKIBI_BASE_URL` for local development). Reads are free; anything that
writes or curates shared state needs the user's approval in the conversation
first, and workspace-owner routes are never called by the agent.

## Setup

1. Install this plugin from the Claude directory, or add this repository as
   a marketplace from Customize > Plugins.
2. Create a scoped key in your Takibi workspace and save the
   `<publicId>.<secret>` line to `~/.takibi/key` with `chmod 600`.
   The key is never printed, pasted in chat, or committed.
3. Run `npx takibibase version` to confirm the API is reachable, then ask
   Claude anything about your Takibi projects.

## Surfaces

The skill loads everywhere, but only surfaces that can run shell commands get
live Takibi data. In Claude Code the skill is fully functional. In claude.ai
chat it currently serves as guidance, because chat cannot execute the CLI; a
remote MCP connector for live chat queries is planned as a follow-up.

## License

MIT — see [LICENSE](LICENSE).
