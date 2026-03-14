# InterpolationBuffer

A timestamped ring buffer that smooths remote entity movement by interpolating between received state snapshots. Samples at `renderTime - bufferMs` to trade a small amount of latency for visually smooth motion.

## Signature

```ts
new InterpolationBuffer(config?: InterpolationBufferConfig)
```

## Config

| Option | Type | Default | Description |
|---|---|---|---|
| `bufferMs` | number | `100` | How far behind real-time to sample, in milliseconds. Higher = smoother but more latency |
| `capacity` | number | `32` | Maximum number of snapshots to keep. Older entries are discarded |

## InterpolationState

Each snapshot pushed into the buffer has this shape:

```ts
interface InterpolationState {
  x: number
  y: number
  angle?: number
  [key: string]: number | undefined  // any additional numeric fields
}
```

## Methods

| Method | Description |
|---|---|
| `push(timestamp, state)` | Add a new snapshot. `timestamp` is in milliseconds (e.g. `performance.now()`) |
| `sample(renderTime)` | Sample the buffer at `renderTime - bufferMs`. Returns an interpolated `InterpolationState` or `null` if the buffer is empty |

## Example

### Smooth remote player movement

```ts
import { InterpolationBuffer } from 'cubeforge'

const buffer = new InterpolationBuffer({ bufferMs: 100 })

// On receiving a state update from the network:
room.onMessage((msg) => {
  if (msg.type === 'entity:state') {
    const { x, y, angle } = msg.payload
    buffer.push(performance.now(), { x, y, angle })
  }
})

// In your game loop, apply the smoothed position:
function update(dt: number) {
  const state = buffer.sample(performance.now())
  if (state) {
    transform.x = state.x
    transform.y = state.y
    if (state.angle !== undefined) transform.angle = state.angle
  }
}
```

### Custom state fields

Any numeric field beyond `x`, `y`, `angle` is interpolated automatically:

```ts
buffer.push(performance.now(), { x: 100, y: 200, hp: 80, shield: 1 })

const state = buffer.sample(performance.now())
// state.x, state.y, state.hp, state.shield are all interpolated
```

### Adjust buffer size for your network conditions

```ts
// Low-latency local network — small buffer
const buffer = new InterpolationBuffer({ bufferMs: 50 })

// High-latency internet — larger buffer for smoother playback
const buffer = new InterpolationBuffer({ bufferMs: 150 })
```

## How it works

`sample(renderTime)` computes `sampleTime = renderTime - bufferMs`, then finds the two snapshots that bracket `sampleTime` and linearly interpolates between them:

```
snapshots:  A ──────── B ──────── C
                         ↑
                    sampleTime
                    (lerps between B and C)
```

**Angle interpolation** uses the shortest path — it automatically wraps around `±π` so a sprite rotating from 170° to -170° doesn't spin the long way around.

**Edge cases:**
- Buffer empty → returns `null`
- `sampleTime` is before all snapshots → returns the first known state (no extrapolation)
- `sampleTime` is after all snapshots → holds the last known state (no drift)

## Choosing `bufferMs`

| Scenario | Recommended `bufferMs` |
|---|---|
| Local LAN / same machine | 50 ms |
| Average internet (50–100 ms RTT) | 100 ms |
| High-latency / mobile (150+ ms RTT) | 150–200 ms |

A good rule of thumb: set `bufferMs` to roughly your expected one-way latency. Too low and you'll see jitter; too high and movement feels laggy.

## Notes

- All numeric fields in `InterpolationState` are interpolated with simple linear interpolation (`lerp`). For values that should not be interpolated (e.g. health, flags), apply them directly from the latest snapshot rather than the buffer.
- The buffer capacity (`capacity`) controls memory use. At 20 Hz network updates and `capacity: 32`, the buffer holds 1.6 seconds of history — enough for most conditions.
- `push` is safe to call at any frequency. The buffer keeps only the most recent `capacity` snapshots.

## See also

- [Multiplayer Guide](/guide/multiplayer) — full walkthrough
- [createWebRTCTransport](/api/create-web-rtc-transport) — P2P transport
- [syncEntity](/api/sync-entity) — broadcast and receive component state
