# Takibi for Claude

Takibi Base (Takibi) is an agent-first knowledge base: humans curate projects, folders, and
uploads, and agents consume them through scoped keys with extractive,
cited answers. This plugin teaches Claude how to work with Takibi: asking
questions against the evidence, searching documents and agent notes, reading
converted document text, and driving Takibi task boards (list, claim, move,
and attach artifacts) — all through the Takibi CLI with the user's own key.

## How it works

The plugin contains a single skill. It carries no code that runs on install
and connects to nothing by itself. When the skill is active, it instructs
Claude to run the `takibibase` CLI in the agent's own shell (global install,
or `npx` zero-install, which downloads the CLI from the npm registry at
`registry.npmjs.org`). The CLI reads the user's scoped key from
`~/.takibi/key` and talks to Takibi's hosted API at
`https://app.takibibase.com`; it also checks the npm registry daily for
updates (silence with `TAKIBI_NO_UPDATE_CHECK=1`). Reads are free; anything
that writes or curates shared state needs the user's approval in the
conversation first, and workspace-owner routes are never called by the agent.

## Setup

1. Install this plugin from the Claude directory, or add this repository as
   a marketplace from Customize > Plugins.
2. Install the CLI: `npm install -g takibibase` (or prefix every command
   with `npx takibibase` instead of installing).
3. Create a scoped key in your Takibi workspace and save the
   `<publicId>.<secret>` line to `~/.takibi/key` with `chmod 600`.
   The key is never printed, pasted in chat, or committed.
4. Run `takibi version` to confirm the API is reachable, then ask
   Claude anything about your Takibi projects.

## Surfaces

The skill loads everywhere, but only surfaces that can run shell commands get
live Takibi data. In Claude Code the skill is fully functional. In claude.ai
chat it currently serves as guidance, because chat cannot execute the CLI; a
remote MCP connector for live chat queries is planned as a follow-up. Cowork
loads the skill too; live queries there need a session that can run the CLI
with your key file.

## Links

- Homepage: [takibibase.com](https://takibibase.com)
- App: [app.takibibase.com](https://app.takibibase.com)
- Agent docs: [takibibase.com/docs/agents](https://takibibase.com/docs/agents)
- CLI (`takibibase` on npm): [npmjs.com/package/takibibase](https://www.npmjs.com/package/takibibase)
- Main repo: [tofuchick3n/takibi-base](https://github.com/tofuchick3n/takibi-base)

## Directory submission

Submitted to the Claude directory. Review status is tracked in the
[developer portal](https://claude.ai/directory/manage/plugins/16a5524b-1934-41ea-995d-4d10b360427b)
(submitting organization only).

## License

MIT — see [LICENSE](LICENSE).
