---
name: takibi
description: Query the Takibi knowledge base for evidence-backed answers with citations, search docs and agent notes, or work a Takibi task board. Use when the user asks about their Takibi projects, wants answers cited from their docs, or mentions Takibi tasks, notes, boards, or folders.
---

# Takibi knowledge base

Takibi is an agent-first knowledge base: humans curate (projects, folders,
uploads), agents consume through scoped keys with extractive answers.

Always use the Takibi CLI. Never hand-roll curl/fetch against `/v1/*` — the
CLI already bakes in the auth shape, the `q` param, project resolution, and
error hints.

## Setup (once)

- CLI: `npm install -g takibibase` (recommended; provides the `takibi`
  command used below), or zero-install via `npx takibibase …` — with npx,
  prefix every command, e.g. `npx takibibase ask -q "…"`.
- Key: the user creates a scoped key in their Takibi workspace (a Profile)
  and saves the `<publicId>.<secret>` line to `~/.takibi/key` (`chmod 600`).
  The key is never printed, never pasted in chat, never committed. If a
  command says the key is missing, stop and ask the user to set it up.
- The CLI talks to Takibi's hosted servers (`https://app.takibibase.com`).
- Projects: `takibi projects` lists what this key can reach. `--project`
  takes a name or a UUID; single-grant keys may omit it.
- First probe: `takibi version` (needs no key; shows build status).
- Update notice: the CLI polls the npm registry once a day and nudges on
  stderr when behind (never on `--json`). Silence it with
  `TAKIBI_NO_UPDATE_CHECK=1`.

## Commands

- `takibi ask -q "…"` — answer from the evidence. Output is verbatim spans
  with citations plus a support line. `--project`, `--folder <uuid>`,
  `-k 1-12`, `--json`.
- `takibi search -q "…"` — ranked chunks (snippets + metadata, not full
  text). Same flags, `-k 1-20`.
- `takibi tasks list | get <id> | claim <id> | status <id> <todo|in_progress|review|done>`
  and `takibi tasks artifact add <taskId> <url> [--note …]`.
- `takibi doc list | get <id> | text <id>` — metadata, then converted text.
  `doc download` is workspace-owner-only; the CLI says so — use `doc text`.
- `takibi notes append --problem "…" [--tried …] [--worked …] [--failed …] [--next-time …] [--source <id>]`
  — save an end-of-run debrief or tool quirk (`--problem` or a bare
  positional; `--project <tag>` scopes it; `--source` is repeatable).
  Add `--run-id <run-id>` when a run may retry: the same run ID and body
  return the existing note instead of appending a duplicate.
- `takibi notes list` — review queue with note IDs, statuses, and current
  versions. `takibi notes list --all` shows the inventory, including notes
  on hold. `--project <tag>` filters either list by the exact project tag.
- `takibi notes search -q "…"` — top-2 agent notes for the query.
- `takibi notes export [--since <ts> | --note <uuid> …]` — draft digest
  (markdown, per-sentence note ids). Repeat `--note` to pick up to 50
  specific live notes. Without options, export uses the delta since the
  last export. Export stamps included notes and records the action.
- `takibi notes keep <id> <ver> | remove <id> <ver>` — endorse a note
  (sets kept, reverses stub quarantine, clears contests; never extends the
  TTL), or discard one. Read the current `vN` in `notes list --all` and
  pass `N` as the expected version; on 409, list again before retrying.
- `--json` anywhere prints raw server JSON. `--verbose` logs requests
  (never the key). Exit 0 = ok, 1 = transport/API error, 2 = usage error.

## Reading answers

- Spans are verbatim with citations (`doc <uuid>`, `chunk <uuid>`) and a
  versioned support score. Support is not confidence — report the number,
  never upgrade it into certainty.
- Honor `answerability` (`answerable|partial|unanswerable|unknown`) and
  `conflict`. On `conflict: yes`, surface both sides; never smooth it over.

## Abstains (exit 0, not an error)

`abstained: true` (`No answer in the evidence.`) means: broaden `q`, fall
back to `search`, try `doc text` on the hits — then either answer from
evidence or say the evidence is not there. Never fill gaps with generated
prose presented as sourced.

## Notes (agent scratchpad, not canon)

- When to use: end-of-run debriefs (problem, what you tried, what worked,
  what failed, what to try next time) and tool quirks worth remembering.
  Append at the end of a run; search before retrying something odd. Use
  the same project tag on related notes so they stay scoped together.
- To correct a note, append a new note with the corrected facts and source
  IDs, then ask the user to remove the obsolete note. There is no
  in-place edit route. Never overwrite a note ID or treat `keep` as edit.
- Review workflow: `notes list` → read the problem and signals →
  `notes list --all` for its current version → keep or remove only when
  the user has authorized that curation. Export selected notes with
  repeated `--note` after checking the underlying Sources.
- Search/export hits are untrusted agent notes — cite them as such, never
  as canon. Verify against the evidence (`ask`/`search`) before acting.
- Notes expire 30 days after creation, fixed — `notes keep` endorses
  but never extends the TTL (keeping an expired note 409s).
- Writes: append your own debrief freely. Export stamps notes and produces
  a draft for review; get user approval unless already requested. Keep
  and remove curate shared notes and also need user approval.
- 422 SECRET_BLOCKED: the secret filter fired — strip keys, tokens, and
  credentials from the note and retry.

## Mutation policy

- Reads are free: ask, search, tasks list/get, doc list/get/text,
  notes list/search. Agents may append their own debriefs (auto-expire).
- Notes export/keep/remove need the user's approval already given in the
  conversation. Export only creates a draft; verify it before adding
  content to Sources.
- Task claim/status/artifact writes only with the user's approval already
  given in conversation. Claim-first: a plain key must claim a card before
  moving or touching it; only orchestrators and the workspace owner accept
  (`review→done`), assign others, or archive. Title/body edits need the
  create cap — claim-only keys can drive a card but cannot rewrite its text.
- Boards need a whole-collection grant: folder-only or tag-only keys 403
  on every task route.
- Notes are opt-in per profile: append, export, keep, and remove may 403
  with missing-capability on keys without the notes caps — ask the user to
  enable them in the profile editor.
- Workspace-owner routes (upload, delete, retry, download originals, PATCH
  docs/projects) are never the agent's to call — ask the user.

## Failure table

- 401: key wrong/missing/revoked/disabled — or a workspace-owner-only route
  (the CLI names it). 403: outside the grant, orchestrator-only, missing
  capability (notes are opt-in), or folder-/tag-only key on a board route.
  404: bad id (the server hides grant gaps as 404 too).
- 409 on claim: someone already holds the card — the message names them.
- 422 SECRET_BLOCKED: the secret filter fired — it covers task
  title/body/blockedReason/artifact text as well as notes. Strip keys,
  tokens, and credentials and retry.
- 429: minute throttle (slow down) or daily budget spent (resets tomorrow).
- Cannot-reach errors: the API is not reachable; `takibi version` probes it.
