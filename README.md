# loom

Loom helps you send data around Roblox without manually packing bytes, unpacking bytes, or guessing what shape a payload is supposed to have.

You describe the data once, then Loom handles the binary encode and decode part for you.

That means:

- smaller, typed payloads
- fewer remote mistakes
- cleaner server and client code
- easier state syncing
- optional schema versioning when your data changes later

## what Loom is, in normal-person terms

Loom is really 3 things:

### `Loom`
Use this when you want to describe data.

Example:
- a player profile
- an inventory entry
- a combat event payload
- a save file
- a MessagingService packet

This is the part that gives you codecs like `struct`, `array`, `map`, `optional`, `union`, and `schema`.

### `LoomRemote`
Use this when you already have a `RemoteEvent` and just want it to send typed data cleanly.

Instead of manually serializing stuff before every fire, you wrap the remote once and use it normally.

### `LoomChannels`
Use this when you want a higher-level pattern already set up for you.

It gives you:
- `event(...)` for normal one-way messages
- `request(...)` for request/response
- `state(...)` for server-to-client state sync

If you do not know which one to pick:

- Need to describe data, use `Loom`
- Need a typed `RemoteEvent`, use `LoomRemote`
- Need event, request, or synced state helpers, use `LoomChannels`

---

## install

Put these in `ReplicatedStorage`:

- `Loom`
- `LoomRemote`
- `LoomChannels`

Module expectations:

- `LoomRemote` expects a sibling module named `Loom`
- `LoomChannels` expects sibling modules named `Loom` and `LoomRemote`

---

## the mental model

Most people will use Loom like this:

### 1. Describe the payload

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Loom = require(ReplicatedStorage.Packages.Loom)

local PlayerStats = Loom.struct({
    {"health", Loom.u16},
    {"stamina", Loom.u16},
    {"alive", Loom.bool},
    {"position", Loom.vec3},
})
```

### 2. Use that payload somewhere

You now have options:

- encode/decode it directly
- put it on a `RemoteEvent` with `LoomRemote`
- use it in a higher-level channel with `LoomChannels`

That is the whole library.

---

## quick start, 3 real ways to use it

## 1. Use `Loom` by itself

Good for:
- saving data
- MessagingService
- MemoryStore
- raw buffers
- anywhere you want compact typed data without remotes

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Loom = require(ReplicatedStorage.Packages.Loom)

local InventoryItem = Loom.struct({
    {"id", Loom.u32},
    {"name", Loom.str},
    {"count", Loom.u16},
    {"equipped", Loom.bool, false},
})

local packet = Loom.encodeRaw(InventoryItem, {
    id = 1001,
    name = "Iron Sword",
    count = 2,
    equipped = true,
})

local decoded = Loom.decodeRaw(InventoryItem, packet)
print(decoded.name, decoded.count)
```

Use this when both sides already agree on the shape and you just want raw binary encode/decode.

---

## 2. Use `LoomRemote` for a typed `RemoteEvent`

Good for:
- hit markers
- combat events
- UI notifications
- sending one payload shape repeatedly through one remote

### server

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomRemote = require(ReplicatedStorage.Packages.LoomRemote)

local remotes = ReplicatedStorage.Remotes

local DamageEvent = Loom.struct({
    {"targetId", Loom.u32},
    {"amount", Loom.u16},
    {"critical", Loom.bool},
})

local damageRemote = LoomRemote.new(remotes.DamageEvent, DamageEvent)

damageRemote:FireClient(player, {
    targetId = 25,
    amount = 18,
    critical = true,
})
```

### client

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomRemote = require(ReplicatedStorage.Packages.LoomRemote)

local remotes = ReplicatedStorage.Remotes

local DamageEvent = Loom.struct({
    {"targetId", Loom.u32},
    {"amount", Loom.u16},
    {"critical", Loom.bool},
})

local damageRemote = LoomRemote.new(remotes.DamageEvent, DamageEvent)

damageRemote:Connect(function(payload)
    print("damage", payload.targetId, payload.amount, payload.critical)
end)
```

Use this when you want the remote layer to stay simple, but you still want typed packets.

---

## 3. Use `LoomChannels.state` for synced state

Good for:
- health bars
- stamina
- ammo
- scoreboard rows
- current objective
- anything the server owns and the client should mirror

### server

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomChannels = require(ReplicatedStorage.Packages.LoomChannels)

local remotes = ReplicatedStorage.Remotes

local playerState = LoomChannels.state(remotes.PlayerState, {
    {"health", Loom.u16},
    {"stamina", Loom.u16},
    {"alive", Loom.bool},
    {"position", Loom.vec3},
})

playerState:Push(player, {
    health = 100,
    stamina = 50,
    alive = true,
    position = Vector3.new(0, 0, 0),
})

playerState:Push(player, {
    health = 92,
    stamina = 44,
    alive = true,
    position = Vector3.new(5, 0, 3),
})
```

### client

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomChannels = require(ReplicatedStorage.Packages.LoomChannels)

local remotes = ReplicatedStorage.Remotes

local playerState = LoomChannels.state(remotes.PlayerState, {
    {"health", Loom.u16},
    {"stamina", Loom.u16},
    {"alive", Loom.bool},
    {"position", Loom.vec3},
})

playerState:Connect(function(state, delta)
    print("full state", state.health, state.stamina, state.position)
    print("just changed", delta.health, delta.position)
end)
```

This is one of the nicest parts of Loom.

You push full state from the server.
The channel only sends what changed.
The client keeps a local cached copy for you.

---

## when to use each part

## use `Loom` when...

- you want a codec for a value shape
- you are saving or loading binary data
- you want base64 or string helpers
- you want versioned schemas
- you want to build packet formats once and reuse them everywhere

## use `LoomRemote` when...

- you already have a `RemoteEvent`
- you want typed payloads
- you do not need request/response or state cache helpers
- you want something very thin and direct

## use `LoomChannels.event` when...

- you want the same thing as `LoomRemote`, but with the channel-style API
- you conceptually think of it as an event channel

## use `LoomChannels.request` when...

- the caller expects a response
- the server needs to answer a question
- you want a typed wrapper around `RemoteFunction`

Examples:
- “can I buy this item?”
- “give me my inventory page”
- “what is my current rank?”

## use `LoomChannels.state` when...

- the server owns the truth
- the client just needs the latest mirrored state
- you do not want to manually diff fields yourself

Examples:
- player vitals
- weapon HUD state
- round state
- quest tracker data

---

## a simple request example

### server

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomChannels = require(ReplicatedStorage.Packages.LoomChannels)

local remotes = ReplicatedStorage.Remotes

local GetCoinsRequest = Loom.struct({
    {"userId", Loom.u32},
})

local GetCoinsResponse = Loom.struct({
    {"coins", Loom.u32},
})

local coinsChannel = LoomChannels.request(
    remotes.GetCoins,
    GetCoinsRequest,
    GetCoinsResponse
)

coinsChannel:Handle(function(player, request)
    if player.UserId ~= request.userId then
        error("user mismatch")
    end

    return {
        coins = 1250,
    }
end)
```

### client

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Loom = require(ReplicatedStorage.Packages.Loom)
local LoomChannels = require(ReplicatedStorage.Packages.LoomChannels)

local remotes = ReplicatedStorage.Remotes

local GetCoinsRequest = Loom.struct({
    {"userId", Loom.u32},
})

local GetCoinsResponse = Loom.struct({
    {"coins", Loom.u32},
})

local coinsChannel = LoomChannels.request(
    remotes.GetCoins,
    GetCoinsRequest,
    GetCoinsResponse
)

local response, err = coinsChannel:InvokeServer({
    userId = game.Players.LocalPlayer.UserId,
})

if err then
    warn("request failed", err)
    return
end

print("coins", response.coins)
```

---

## the most useful codecs

You do not need to learn everything up front. These are the ones most people will actually use first:

### primitives

- `Loom.u8`, `Loom.u16`, `Loom.u32`
- `Loom.i8`, `Loom.i16`, `Loom.i32`
- `Loom.f32`, `Loom.f64`
- `Loom.varint`, `Loom.svarint`
- `Loom.bool`
- `Loom.str`
- `Loom.buffer`
- `Loom.bounded_str(maxLen)`

### roblox types

- `Loom.vec2`
- `Loom.vec3`
- `Loom.color3`
- `Loom.udim`
- `Loom.udim2`
- `Loom.cframe`
- `Loom.cframe_net(...)`
- `Loom.roblox_enum(Enum.SomeEnum)`

### composite codecs

- `Loom.struct(...)`, named object
- `Loom.array(...)`, list of values
- `Loom.map(...)`, keyed table
- `Loom.optional(...)`, nil or value
- `Loom.tuple(...)`, fixed ordered values
- `Loom.union(...)`, one of several value shapes
- `Loom.enum(...)`, string enum
- `Loom.bitfield(...)`, packed booleans

### state and versioning helpers

- `Loom.delta_struct(...)`
- `Loom.tracked_struct(...)`
- `Loom.applyDelta(...)`
- `Loom.schema(...)`

---

## schema versioning, when you should care

Use a schema when the data may live longer than the current code version.

That usually means:
- save data
- datastore blobs
- cached payloads
- messages crossing deploy versions

If both sides are always updated together, a plain codec is often enough.

### example

```lua
local SaveProfile = Loom.schema({
    name = "SaveProfile",
    version = 2,
    codec = Loom.struct({
        {"coins", Loom.u32},
        {"level", Loom.u16},
        {"title", Loom.str, "Rookie"},
    }),
    migrations = {
        [1] = function(data)
            data.title = "Rookie"
            return data
        end,
    },
})
```

Rule of thumb:

- use `struct` for current in-memory payloads
- use `schema` for data that must survive format changes later

---

## state sync notes

`LoomChannels.state` is not magic, but it is very convenient.

What it does:
- the server compares the last pushed state to the new one
- only changed fields are sent
- if a field becomes `nil`, Loom marks that as a delete
- the client applies the delta into its cached state table

So you write code like you are pushing a whole state object, but the wire payload stays smaller when only a few fields changed.

---

## things that will save you pain

### 1. `struct` field order matters

Do not reorder or remove fields in a plain `struct` you already shipped.
If the shape needs to evolve safely, use a `schema`.

### 2. `state(...)` is for server-owned truth

If both client and server are trying to author the same state, your design is the problem, not the library.

### 3. `map(...)` is not the fastest option for hot paths

It sorts keys so the wire output is deterministic.
That is useful, but it adds overhead.
If performance matters a lot, an `array(struct(...))` layout is usually a better fit.

### 4. `tracked_struct(...)` only notices reference changes for nested tables

If you mutate a nested table in place, that may not count as changed.
Replace the nested table if you want the tracker to notice it.

### 5. use bounded strings when the input is untrusted

If players can send it, putting a size limit on strings is usually smart.

---

## recommended folder examples

Look in `/usages` for actual scenarios, not toy snippets.

Good examples are things like:

- a weapon fire event
- an inventory request
- a round state sync
- a save schema migration
- a MessagingService packet encoded to base64

If an example does not answer “when would I use this in a real game?”, it probably should not be there.

---

## license

MIT license. do whatever you want with it, just keep the attribution.
