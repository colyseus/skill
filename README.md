# Colyseus agent skill

An [Agent Skill](https://agentskills.io) that teaches coding agents the current
Colyseus APIs.

Most models learned Colyseus from pre-0.18 material, so left to memory they
write `colyseus.js` imports, `Room<MyState>` generics, `client.id`, and
`onMessage()` registrations. This skill pins them to 0.18 and, just as
importantly, makes them check the installed version before writing anything.

## Install

Agent Skills are an [open standard](https://agentskills.io/specification), so
this works in Claude Code, Codex, Cursor, Gemini CLI, Copilot, opencode, and
[every other client that implements it](https://agentskills.io/clients).

```sh
npx skills add colyseus/skill            # this project
npx skills add colyseus/skill --global   # every project
```

That asks which of your installed agents to set up, and writes to each one's
own directory. To do it by hand, clone into the directory your tool scans. The
target directory **must** be named `colyseus`, to match the `name` in
`SKILL.md`:

| Tool | Every project | One project |
| --- | --- | --- |
| Codex, Cursor, Gemini CLI, Copilot, opencode | `~/.agents/skills/colyseus` | `.agents/skills/colyseus` |
| Claude Code | `~/.claude/skills/colyseus` | `.claude/skills/colyseus` |

```sh
git clone https://github.com/colyseus/skill ~/.agents/skills/colyseus
```

The skill activates on its own when a task involves Colyseus, matched from the
`description` in `SKILL.md`. Nothing in it is tool-specific: the frontmatter
uses only fields the specification defines, it declares no `compatibility`
requirement, and `references/` is the standard bundled documentation slot.

Skills are instructions an agent will act on, so read one before installing it.
This one ships no `scripts/`: it asks the agent to read documentation and check
your installed version, and it never asks it to run anything. The sync tooling
in `tools/` is for maintaining this repository and is not part of the skill.

## What is in it

- `SKILL.md`: version detection, the anti-pattern table, old-shape → new-shape
  diffs for the APIs that changed structure (server definition, state
  callbacks, reconnection), the canonical 0.18 shapes, a symptom table for
  existing projects, and a routing table. Hand-written.
- `references/`: full documentation text for the areas agents get wrong:
  schema, room, client SDK, netcode. **Generated. Do not edit.**

## Keeping references current

Every page on docs.colyseus.io is served as raw markdown at `<route>.md`.
`sources.json` lists which pages compose each reference, and the sync script
assembles them:

```sh
npm run sync      # rewrite references/
npm run check     # fail if references/ is out of date
```

`npm run check` also resolves every documentation URL named in `SKILL.md`. A
404 means a page was renamed: the site's redirect map is client-side
JavaScript, so it cannot rescue a raw fetch, and a stale URL is dead for the
agent that follows it. The same goes for the `references/<file> § "Heading"`
pointers `SKILL.md` uses to send an agent to one section of a reference
instead of the whole file: each must match a heading in the generated text.

To test against an unpublished docs build, point `DOCS_ORIGIN` at its output
directory:

```sh
DOCS_ORIGIN=../docs/out npm run sync
```

CI runs the check on every pull request and re-syncs weekly, opening a pull
request when the documentation has moved.
