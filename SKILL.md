---
name: colyseus
description: Colyseus multiplayer game servers and clients: rooms, Schema state synchronization, matchmaking, client SDKs, client-side prediction. Use when colyseus, @colyseus/schema, @colyseus/sdk or colyseus.js appears in the task or the project's dependencies, and before writing any Room or Schema code.
license: MIT
metadata:
  version: 0.18.0
  documentation: https://docs.colyseus.io
---

# Colyseus

Colyseus is an authoritative multiplayer framework for Node.js. The server owns
the state; clients send input and receive binary delta patches.

The APIs moved substantially in 0.18, and most model training data predates it.
Assume your recall of Colyseus is out of date, and check before you write.

## Step 1: establish the version. Do not skip this.

Read the project's `package.json` (or `pnpm-lock.yaml` / `yarn.lock`) for
`colyseus` and `@colyseus/schema`.

| Installed | What to do |
| --- | --- |
| `colyseus` 0.18.x, `@colyseus/schema` 5.x | Use this skill's guidance directly. |
| 0.17.x | Read `https://docs.colyseus.io/migrating/0.18.md` to see what does **not** yet apply, then follow 0.17 idioms. |
| 0.16.x or older | Follow the version's own docs. Do **not** apply anything below. Version-specific sites: `https://0-16-x.docs.colyseus.io/`. |
| Nothing installed yet | Scaffold with `npm create colyseus-app@latest`, which is 0.18. |

If the version is ambiguous, ask rather than guess. Writing 0.18 APIs into a
0.16 project produces code that fails at import time, and writing 0.16 APIs into
a 0.18 project produces code that fails at runtime or silently desynchronizes.

## Step 2: what memory gets wrong

These are the errors to actively check for in your own output. Left column is
what an out-of-date recall produces.

| Do not write | Write instead | Why |
| --- | --- | --- |
| `import { Client } from "colyseus.js"` | `import { Client } from "@colyseus/sdk"` | The JS client package was renamed. |
| `class MyRoom extends Room<MyState>` | `class MyRoom extends Room` with `state = new MyState()` | The generic is now an options object, `Room<{ state, metadata, client }>`. A bare state type no longer type-checks. |
| `onCreate() { this.onMessage("x", fn) }` | `messages = { x: fn }` as a class property | The declarative map is the primary API. `this.onMessage()` still works and returns an unbind function. |
| `@type("string") name: string` only | `schema({ name: t.string() }, "MyState")` | The builder is the primary style and needs no compiler flags. Decorators remain supported. |
| `client.id` | `client.sessionId` | `client.id` was **removed** in 0.18. |
| `this.setMetadata({ status: "x" })` expecting a merge | `this.setMetadata({ ...this.metadata, status: "x" })` | 0.18 **replaces** metadata wholesale instead of shallow-merging. Same for `setMatchmaking({ metadata })`. |
| `room.state.onChange(...)` | `const callbacks = Callbacks.get(room)`, then `callbacks(...)` | Callbacks go through a handler obtained from the SDK. |
| `this.setSimulationInterval(fn)` | `this.setTimestep(fn)` | Renamed. The old name forwards, so it is not broken, but new code should use `setTimestep()`. For predicted games use `setFixedTimestep(step, tickRate)`. |
| A `Schema` with 64+ fields | Move the overflow onto a nested child `Schema` | 0.18 caps a `Schema` at 63 fields and throws at definition time. Inherited fields count. |
| `@colyseus/fossil-delta-serializer` | `@colyseus/schema` | Removed in 0.18. |

Two more that bite in production:

- **Playground** returns 404 on production mounts (`NODE_ENV=production`) unless
  you pass a guard middleware via its `use` option.
- **`@colyseus/auth` password hashing changed** in 0.18. Pre-0.18 hashes cannot
  be verified by the new hasher, so existing email/password users must reset
  their passwords. The legacy `scrypt` hasher and its global `AUTH_SALT` are
  deprecated.

## Step 3: the canonical 0.18 shapes

Server entry (`src/index.ts`):

```ts
import { listen } from "@colyseus/tools";
import app from "./app.config.js";

listen(app);
```

Server config (`src/app.config.ts`):

```ts
import { defineServer, defineRoom } from "colyseus";
import { MyRoom } from "./rooms/MyRoom.js";

const server = defineServer({
    rooms: {
        my_room: defineRoom(MyRoom),
    },
});

export default server;
```

State (`src/rooms/schema/MyRoomState.ts`):

```ts
import { schema, t, type SchemaType } from "@colyseus/schema";

export const MyRoomState = schema({
    mySynchronizedProperty: t.string().default("Hello world"),
});
export type MyRoomState = SchemaType<typeof MyRoomState>;
```

`schema()` returns a real class, so `new MyRoomState()` and `instanceof` both
work. The `export type` alias makes the same name usable as a type. Raw string
field types (`"string"`) throw at definition time in schema 5; they remain valid
as collection child types, e.g. `t.array("string")`.

Room (`src/rooms/MyRoom.ts`):

```ts
import { Room, Client, CloseCode } from "colyseus";
import { MyRoomState } from "./schema/MyRoomState.js";

export class MyRoom extends Room {
    maxClients = 4;
    state = new MyRoomState();

    messages = {
        move: (client: Client, message: { x: number; y: number }) => {
            // mutate this.state here. Only the server may write to it
        },
        // return a value to answer a client's room.request()
        "get-profile": async (client: Client, { userId }: { userId: number }) => {
            return await db.profiles.findById(userId);
        },
    };

    onCreate(options: any) {}
    onJoin(client: Client, options: any) {}
    onLeave(client: Client, code: CloseCode) {}
    onDispose() {}
}
```

Client:

```ts
import { Client, Callbacks } from "@colyseus/sdk";

const client = new Client("http://localhost:2567");
const room = await client.joinOrCreate("my_room");

const callbacks = Callbacks.get(room);

room.send("move", { x: 1, y: 2 });
const profile = await room.request("get-profile", { userId: 42 });
```

## Step 4: where to look

Read the bundled reference before writing non-trivial code in that area. They
are the full documentation text, generated from the site.

| Task | Read |
| --- | --- |
| Defining or changing synchronized state, collections, filtering | `references/schema.md` |
| Room lifecycle, messages, request/response, timers, disposal | `references/room.md` |
| Connecting, joining, sending, state callbacks on the client | `references/client-sdk.md` |
| Client prediction, reconciliation, lag compensation, fixed timestep | `references/netcode.md` |

Everything else is on the site, and every page is available as raw markdown by
appending `.md`:

| Topic | URL |
| --- | --- |
| Matchmaking, lobby, queue, visibility | `https://docs.colyseus.io/matchmaker.md` |
| Authentication | `https://docs.colyseus.io/auth.md` |
| Database and persistence | `https://docs.colyseus.io/database.md` |
| Server config, transport, presence, driver | `https://docs.colyseus.io/server.md` |
| Scaling out | `https://docs.colyseus.io/scalability.md` |
| Deployment | `https://docs.colyseus.io/deployment.md` |
| Colyseus Cloud | `https://docs.colyseus.io/cloud.md` |
| Playground, monitor, load testing, unit testing | `https://docs.colyseus.io/tools.md` |
| Upgrading from an earlier version | `https://docs.colyseus.io/migrating/0.18.md` |
| Full index of every page | `https://docs.colyseus.io/llms.txt` |

When a task needs a page that is neither bundled nor listed, fetch
`https://docs.colyseus.io/llms.txt` and pick from it rather than guessing a URL.

## Working rules

- The server is authoritative. Client code must never be the only thing
  enforcing a rule; validate in the room.
- Only the server mutates `state`. Clients send messages or input and react to
  callbacks.
- Prefer the declarative `messages` map over imperative `onMessage()` calls, and
  validate payloads. Never trust a `sessionId` supplied in a message body; use
  the `client` argument the handler receives.
- Keep synchronized state small. It is broadcast on every patch, so anything a
  client does not need belongs outside the schema.
- After changing state structure, check the 63-field cap on every affected
  class, parents included.
