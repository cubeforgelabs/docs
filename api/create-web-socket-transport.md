# createWebSocketTransport

Wraps the native browser `WebSocket` API into a `BinaryNetTransport`. Supports text and binary (ArrayBuffer) messages, optional auto-reconnect with exponential backoff, and multiple handlers per event.

## Signature

```ts
function createWebSocketTransport(
  url: string,
  opts?: WebSocketTransportOptions
): BinaryNetTransport
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `reconnect` | boolean | `false` | Automatically reconnect on disconnect with exponential backoff |
| `reconnectBaseDelayMs` | number | `500` | Base reconnect delay in ms. Doubles each attempt, capped at 16 s |
| `maxReconnectAttempts` | number | `Infinity` | Maximum reconnect attempts before giving up |

## Returns

A `BinaryNetTransport` with these methods:

| Method | Description |
|---|---|
| `send(data)` | Send a string message (only when connected) |
| `sendBinary(data)` | Send an `ArrayBuffer` message (only when connected) |
| `onMessage(handler)` | Register a handler for incoming text messages |
| `onBinaryMessage(handler)` | Register a handler for incoming binary (`ArrayBuffer`) messages |
| `onConnect(handler)` | Register a handler for connection open. Fires immediately if already connected |
| `onDisconnect(handler)` | Register a handler for disconnection |
| `close()` | Close the connection. Cancels any pending reconnect |

## Examples

### Basic connection

```ts
import { createWebSocketTransport, Room } from 'cubeforge'

const transport = createWebSocketTransport('wss://your-server.com/game')
const room = new Room({ transport })

room.onMessage((msg) => console.log(msg))
room.send({ type: 'hello', payload: null })
```

### Auto-reconnect

```ts
const transport = createWebSocketTransport('wss://your-server.com/game', {
  reconnect: true,
  reconnectBaseDelayMs: 500,   // 500ms → 1s → 2s → 4s … up to 16s
  maxReconnectAttempts: 10,
})
```

### Binary messages

```ts
const transport = createWebSocketTransport('wss://your-server.com/game')

// Send binary
const buf = new ArrayBuffer(8)
new Float32Array(buf).set([playerX, playerY])
transport.sendBinary(buf)

// Receive binary
transport.onBinaryMessage((buf) => {
  const [x, y] = new Float32Array(buf)
  console.log('remote position:', x, y)
})
```

## Reconnect behaviour

When `reconnect: true`, the transport uses exponential backoff after each disconnect:

```
attempt 1 → wait 500ms
attempt 2 → wait 1000ms
attempt 3 → wait 2000ms
…
capped at  → 16000ms
```

The reconnect loop stops when:
- `maxReconnectAttempts` is reached
- `close()` is called manually

## Notes

- The transport connects immediately on creation — no separate `connect()` call needed.
- Multiple handlers can be registered for each event type via repeated `onMessage` / `onBinaryMessage` calls.
- `binaryType` is set to `'arraybuffer'` — binary messages always arrive as `ArrayBuffer`, not `Blob`.
- `send` and `sendBinary` are silent no-ops when the socket is not in the `OPEN` state.

## See also

- [Multiplayer Guide](/guide/multiplayer) — full setup walkthrough
- [Room](/api/room) — structured JSON message routing on top of a transport
- [createWebRTCTransport](/api/create-web-rtc-transport) — P2P DataChannel transport
