# Multiplayer

`@cubeforge/net` provides transport-agnostic helpers for building multiplayer games. It does not dictate a networking backend — you bring your own WebSocket server, WebRTC signaling, or any custom transport.

## Concepts

| Helper | Purpose |
|---|---|
| `createWebSocketTransport` | Wraps a WebSocket into a `BinaryNetTransport` with optional auto-reconnect |
| `createWebRTCTransport` | Peer-to-peer DataChannel transport (UDP-like by default) |
| `Room` | Connection lifecycle, message routing, peer tracking, ping/RTT |
| `syncEntity` | Broadcasts component state from owner → receivers each tick with delta detection |
| `useNetworkInput` | Collects local key state, broadcasts at a configurable rate, accumulates remote inputs |
| `ClientPrediction` | Snapshot-based rollback for client prediction |
| `InterpolationBuffer` | Smooths remote entity movement by interpolating between received snapshots |

---

## Quick start — WebSocket entity sync

```tsx
import { createWebSocketTransport, Room, syncEntity } from 'cubeforge'
import { useEffect } from 'react'
import { useGame, useEntity } from 'cubeforge'

function NetworkedPlayer() {
  const { ecs } = useGame()
  const id = useEntity()

  useEffect(() => {
    const transport = createWebSocketTransport('wss://your-server.com/game', {
      reconnect: true,
    })
    const room = new Room({ transport })

    const sync = syncEntity({
      room,
      world: ecs,
      entityId: id,
      components: ['Transform', 'RigidBody'],
      owner: true,
      tickRate: 20,
    })

    sync.start()
    return () => {
      sync.stop()
      room.disconnect()
    }
  }, [])

  return null
}
```

Remote peers call `syncEntity` with `owner: false` to receive and apply state.

---

## Transports

### WebSocket

```ts
import { createWebSocketTransport } from 'cubeforge'

const transport = createWebSocketTransport('wss://your-server.com', {
  reconnect: true,           // auto-reconnect with exponential backoff
  reconnectBaseDelayMs: 500, // 500ms → 1s → 2s … capped at 16s
  maxReconnectAttempts: 10,
})
```

Supports both text (`send`) and binary (`sendBinary` / `onBinaryMessage`) messages.

### WebRTC (P2P)

WebRTC DataChannels are unreliable and unordered by default — similar to UDP, which is ideal for game state where only the latest packet matters.

```ts
import { createWebRTCTransport } from 'cubeforge'

const rtc = createWebRTCTransport({ ordered: false }) // UDP-like (default)
```

WebRTC requires a signaling channel to exchange SDP offers and ICE candidates. Use your existing WebSocket `Room` for this:

**Initiating peer:**
```ts
rtc.onIceCandidate((c) => signalingRoom.send({ type: 'rtc:ice', payload: c }))

const offer = await rtc.createOffer()
signalingRoom.send({ type: 'rtc:offer', payload: offer })

signalingRoom.onMessage(async (msg) => {
  if (msg.type === 'rtc:answer') await rtc.handleAnswer(msg.payload)
  if (msg.type === 'rtc:ice')    await rtc.addIceCandidate(msg.payload)
})
```

**Responding peer:**
```ts
rtc.onIceCandidate((c) => signalingRoom.send({ type: 'rtc:ice', payload: c }))

signalingRoom.onMessage(async (msg) => {
  if (msg.type === 'rtc:offer') {
    const answer = await rtc.handleOffer(msg.payload)
    signalingRoom.send({ type: 'rtc:answer', payload: answer })
  }
  if (msg.type === 'rtc:ice') await rtc.addIceCandidate(msg.payload)
})
```

Once the P2P channel opens, wrap it in a `Room` like any other transport:

```ts
rtc.onConnect(() => {
  const p2pRoom = new Room({ transport: rtc })
  // use p2pRoom exactly like a WebSocket room
})
```

### Custom transport

Any object satisfying `NetTransport` works:

```ts
interface NetTransport {
  send(data: string): void
  onMessage(handler: (data: string) => void): void
  onConnect(handler: () => void): void
  onDisconnect(handler: () => void): void
  close(): void
}
```

---

## Room

`Room` manages connection state, JSON message framing, peer tracking, and ping/RTT measurement.

```ts
import { Room } from 'cubeforge'

const room = new Room({
  transport,
  onPeerJoin: (id) => console.log('joined:', id),
  onPeerLeave: (id) => console.log('left:', id),
})

// Send a structured message (auto-stamped with seq number)
room.send({ type: 'shoot', payload: { x: 100, y: 200 } })
room.broadcast({ type: 'jump', payload: null })

// Receive messages
const unsub = room.onMessage((msg) => {
  console.log(msg.type, msg.payload, msg.seq)
})
unsub() // unsubscribe

// Peer tracking
console.log(room.peers) // Set<string> of connected peer IDs

// Ping / RTT
room.ping()
console.log(room.latencyMs) // last measured round-trip time in ms

// Connection state
console.log(room.isConnected)

room.disconnect()
```

### Peer messages

When the server forwards a message with a `peerId` field, `Room` routes it to `onPeerMessage`:

```ts
const room = new Room({
  transport,
  onPeerMessage: (peerId, msg) => {
    console.log(`${peerId} said:`, msg.type)
  },
})
```

---

## syncEntity

Broadcasts a set of ECS component fields from the owning peer, and applies incoming state on receivers. Uses **delta detection** — only changed components are sent each tick, reducing bandwidth.

```ts
import { syncEntity } from 'cubeforge'

const sync = syncEntity({
  room,
  world: ecs,
  entityId: id,
  components: ['Transform', 'RigidBody'],
  owner: true,   // false on remote peers
  tickRate: 20,  // Hz (default 20)
})

sync.start()
sync.stop()    // clears delta cache — next start() sends full state
```

Remote peers receive `entity:state` messages and apply them via `Object.assign` onto the existing ECS component objects — existing component references stay valid.

---

## InterpolationBuffer

Smooth remote entity movement by buffering and interpolating received snapshots. Samples `bufferMs` behind real time to absorb network jitter.

```ts
import { InterpolationBuffer } from 'cubeforge'

const buffer = new InterpolationBuffer({ bufferMs: 100 })

// On network update
room.onMessage((msg) => {
  if (msg.type === 'entity:state') {
    buffer.push(performance.now(), {
      x: msg.payload.Transform.x,
      y: msg.payload.Transform.y,
      angle: msg.payload.Transform.angle,
    })
  }
})

// In your game loop
function update() {
  const state = buffer.sample(performance.now())
  if (state) {
    transform.x = state.x
    transform.y = state.y
    if (state.angle !== undefined) transform.angle = state.angle
  }
}
```

**Choosing `bufferMs`:** roughly equal to your expected one-way latency. 100 ms is a good default for typical internet connections.

---

## useNetworkInput

Broadcasts local key state at a configurable rate and accumulates inputs from remote peers.

```tsx
import { useNetworkInput } from 'cubeforge'

const { localInput, remoteInputs } = useNetworkInput({
  room,
  keys: ['ArrowLeft', 'ArrowRight', 'Space'],
  tickRate: 20, // Hz (default 20)
})

// localInput.ArrowLeft — boolean, true while held
// remoteInputs['player-2'].Space — remote peer's key state
```

Pass your `InputManager` to use action bindings instead of raw key names:

```ts
const { localInput } = useNetworkInput({
  room,
  keys: ['jump', 'left', 'right'],
  input: inputManager, // from createInputMap / useInput
  tickRate: 20,
})
```

---

## ClientPrediction

Snapshot-based rollback using `world.getSnapshot()` / `world.restoreSnapshot()`:

```ts
import { ClientPrediction } from 'cubeforge'

const prediction = new ClientPrediction(world, 60) // 60-frame buffer

// In your game loop, save each frame:
prediction.saveFrame(frameNumber)

// On receiving a correction from the server:
prediction.applyCorrection(serverSnapshot, frameNumber)
// Rolls back to the corrected frame and re-simulates forward
```

---

## Architecture patterns

### Authoritative server (recommended)

```
Client A ──send inputs──▶ Server ──broadcast state──▶ Client A
                                                    ▶ Client B
```
- Server runs physics; clients display interpolated state via `InterpolationBuffer`
- Use `syncEntity` with `owner: false` on clients
- Add `room.ping()` / `room.latencyMs` to display connection quality

### Peer-to-peer

```
Client A ◀──────────────────────────────────────▶ Client B
```
- Each client owns their own entity, broadcasts its state
- Use `createWebRTCTransport` for lower latency than WebSocket relay
- Use `InterpolationBuffer` on each client for the remote entity

### Listen server

One client acts as authority; others connect through it:
- Host uses WebSocket to a relay; peers use WebRTC or the relay
- `room.peers` tracks who is connected

---

## Notes

- `@cubeforge/net` is transport-agnostic. Works with WebSockets, WebRTC, Partykit, or any service that can send and receive strings.
- `syncEntity` uses `Object.assign` on receivers — existing ECS component references stay valid.
- Every `send` / `broadcast` call auto-stamps a `seq` field, useful for ordering on the receiver.
- `Room.ping()` sends `room:ping` and measures time until the server echoes `room:pong`. Pong messages are not forwarded to `onMessage` handlers.
- `<Game deterministic />` mode makes rollback (`ClientPrediction`) more reliable by ensuring identical physics results across peers.
