# createWebRTCTransport

Creates a peer-to-peer WebRTC DataChannel transport implementing the same `BinaryNetTransport` interface as `createWebSocketTransport`. Drop it in wherever a WebSocket transport is used. Signaling (SDP + ICE exchange) is left to you — use your existing `Room` WebSocket connection to shuttle the handshake.

## Signature

```ts
function createWebRTCTransport(config?: WebRTCTransportConfig): WebRTCTransport
```

## Config

| Option | Type | Default | Description |
|---|---|---|---|
| `iceServers` | RTCIceServer[] | Google STUN | STUN/TURN servers for ICE negotiation |
| `ordered` | boolean | `false` | `false` = unreliable + unordered (UDP-like, best for game state). `true` = reliable + ordered (TCP-like, better for chat/events) |
| `maxRetransmits` | number | `0` | For unreliable channels: max retransmissions per packet. Ignored when `ordered: true` |

## WebRTCTransport methods

In addition to all `BinaryNetTransport` methods (`send`, `sendBinary`, `onMessage`, `onBinaryMessage`, `onConnect`, `onDisconnect`, `close`):

| Method | Description |
|---|---|
| `createOffer()` | Create an SDP offer. Call on the **initiating peer** and send the result via signaling |
| `handleAnswer(sdp)` | Apply a remote SDP answer. Call on the **initiating peer** after the responder replies |
| `handleOffer(sdp)` | Apply a remote SDP offer and return a local answer. Call on the **responding peer** |
| `addIceCandidate(candidate)` | Apply an ICE candidate received from the remote peer via signaling |
| `onIceCandidate(handler)` | Register a handler for locally generated ICE candidates — forward these via signaling |

## Setup — complete example

WebRTC requires a signaling channel to exchange SDP and ICE candidates before the peer connection opens. Use your existing `Room` WebSocket for this.

### Initiating peer

```ts
import { createWebRTCTransport, createWebSocketTransport, Room } from 'cubeforge'

// 1. Existing WebSocket room (used as signaling channel)
const wsTransport = createWebSocketTransport('wss://your-server.com/game')
const signalingRoom = new Room({ transport: wsTransport })

// 2. Create the WebRTC transport (UDP-like by default)
const rtc = createWebRTCTransport({ ordered: false })

// 3. Forward local ICE candidates via signaling
rtc.onIceCandidate((candidate) => {
  signalingRoom.send({ type: 'rtc:ice', payload: candidate })
})

// 4. Listen for answer and remote ICE candidates
signalingRoom.onMessage(async (msg) => {
  if (msg.type === 'rtc:answer') await rtc.handleAnswer(msg.payload as RTCSessionDescriptionInit)
  if (msg.type === 'rtc:ice' && msg.payload) await rtc.addIceCandidate(msg.payload as RTCIceCandidateInit)
})

// 5. Create offer and send it to the remote peer
const offer = await rtc.createOffer()
signalingRoom.send({ type: 'rtc:offer', payload: offer })

// 6. Once connected, use rtc like any other transport
rtc.onConnect(() => console.log('P2P connected'))
rtc.onMessage((data) => console.log('received:', data))
```

### Responding peer

```ts
const rtc = createWebRTCTransport({ ordered: false })

rtc.onIceCandidate((candidate) => {
  signalingRoom.send({ type: 'rtc:ice', payload: candidate })
})

signalingRoom.onMessage(async (msg) => {
  if (msg.type === 'rtc:offer') {
    const answer = await rtc.handleOffer(msg.payload as RTCSessionDescriptionInit)
    signalingRoom.send({ type: 'rtc:answer', payload: answer })
  }
  if (msg.type === 'rtc:ice' && msg.payload) await rtc.addIceCandidate(msg.payload as RTCIceCandidateInit)
})
```

### Using the P2P transport with Room

Once connected, wrap it in a `Room` for structured messaging — identical to WebSocket usage:

```ts
rtc.onConnect(() => {
  const p2pRoom = new Room({ transport: rtc })
  p2pRoom.send({ type: 'hello', payload: null })
})
```

## Unreliable vs reliable channels

| | `ordered: false` (default) | `ordered: true` |
|---|---|---|
| Reliability | Unreliable | Reliable |
| Ordering | Unordered | Ordered |
| Best for | Game state (latest packet wins) | Chat, events, game commands |
| Analogy | UDP | TCP |

For game state synchronisation, unreliable + unordered is almost always better — stale state packets are worthless, so dropping them is correct.

## Notes

- WebRTC requires a signaling channel to bootstrap — the library does not provide one. Use your existing WebSocket `Room` as shown above.
- The default STUN server is `stun:stun.l.google.com:19302`. For production games with players behind symmetric NATs, add a TURN server.
- `close()` closes both the data channel and the underlying `RTCPeerConnection`.
- `onConnect` fires when the data channel opens (after ICE negotiation completes), not when the `RTCPeerConnection` connects.
- Binary messages sent via `sendBinary` arrive as `ArrayBuffer` on the receiving end.

## See also

- [Multiplayer Guide](/guide/multiplayer) — full walkthrough including signaling, entity sync, and prediction
- [createWebSocketTransport](/api/create-web-socket-transport) — WebSocket transport (for the signaling channel)
- [InterpolationBuffer](/api/interpolation-buffer) — smooth remote entity movement
- [Room](/api/room) — structured message routing on top of any transport
