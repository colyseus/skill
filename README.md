# Colyseus agent skill

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills)
that teaches coding agents the current Colyseus APIs.

Most models learned Colyseus from pre-0.18 material, so left to memory they
write `colyseus.js` imports, `Room<MyState>` generics, `client.id`, and
`onMessage()` registrations. This skill pins them to 0.18 and, just as
importantly, makes them check the installed version before writing anything.

## Install

**Claude Code**, for one project:

```sh
git clone https://github.com/colyseus/skill .claude/skills/colyseus
```

Or for every project on the machine:

```sh
git clone https://github.com/colyseus/skill ~/.claude/skills/colyseus
```

The skill activates on its own when a task involves Colyseus. Other agents that
read `SKILL.md` conventions can use the same checkout.

## What is in it

- `SKILL.md`: version detection, the anti-pattern table, the canonical 0.18
  shapes, and a routing table. Hand-written.
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
agent that follows it.

To test against an unpublished docs build, point `DOCS_ORIGIN` at its output
directory:

```sh
DOCS_ORIGIN=../docs/out npm run sync
```

CI runs the check on every pull request and re-syncs weekly, opening a pull
request when the documentation has moved.
