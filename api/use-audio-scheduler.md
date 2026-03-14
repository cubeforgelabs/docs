# useAudioScheduler

BPM-synced beat scheduler using Web Audio API lookahead timing. Beat callbacks receive the exact `AudioContext` timestamp at which each beat falls, so you can schedule sounds with **sample-accurate precision** — no jitter from `setTimeout` alone.

## Signature

```ts
function useAudioScheduler(opts: AudioSchedulerOptions): AudioSchedulerControls
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `bpm` | number | — | **Required.** Beats per minute |
| `beatsPerBar` | number | `4` | Time signature numerator (beats per bar) |
| `lookaheadSec` | number | `0.1` | How far ahead in seconds to schedule each tick |
| `intervalMs` | number | `25` | How often the scheduler runs in milliseconds. Lower = tighter timing at the cost of CPU |

## AudioSchedulerControls

| Method / Property | Description |
|---|---|
| `start()` | Start the scheduler. Resets beat and bar counters to 0 |
| `stop()` | Stop the scheduler. Handlers are preserved for the next `start()` |
| `onBeat(handler)` | Register a callback fired for every beat. Returns an unsubscribe function |
| `onBar(handler)` | Register a callback fired at beat 0 of every bar. Returns an unsubscribe function |
| `isRunning` | Whether the scheduler is currently running |
| `currentBeat` | Current beat index within the bar (0 to `beatsPerBar - 1`) |
| `currentBar` | Current bar number from start (0-indexed) |

### Beat handler signature

```ts
type BeatHandler = (beat: number, bar: number, time: number) => void
```

- `beat` — 0-indexed within the bar (0 to `beatsPerBar - 1`)
- `bar` — bar number from start (0-indexed)
- `time` — **AudioContext timestamp** at which the beat occurs — pass directly to `AudioBufferSourceNode.start(time)` for accurate scheduling

### Bar handler signature

```ts
type BarHandler = (bar: number, time: number) => void
```

## Examples

### Play a metronome click on every beat

```tsx
import { useAudioScheduler, useSound } from 'cubeforge'

function Metronome() {
  const scheduler = useAudioScheduler({ bpm: 120, beatsPerBar: 4 })
  const { play } = useSound('/sfx/click.wav', { group: 'sfx' })

  scheduler.onBeat((_beat, _bar, _time) => {
    play()
  })

  return (
    <button onClick={() => scheduler.isRunning ? scheduler.stop() : scheduler.start()}>
      {scheduler.isRunning ? 'Stop' : 'Start'}
    </button>
  )
}
```

### Accent beat 0 of each bar

```tsx
const clickHi = useSound('/sfx/click-hi.wav')
const clickLo = useSound('/sfx/click-lo.wav')

scheduler.onBeat((beat) => {
  if (beat === 0) clickHi.play()
  else clickLo.play()
})
```

### Switch music section every 4 bars

```tsx
const sections = ['/music/intro.ogg', '/music/verse.ogg', '/music/chorus.ogg']

scheduler.onBar((bar) => {
  if (bar % 4 === 0) {
    const section = Math.floor(bar / 4) % sections.length
    music.crossfadeTo(sections[section], 0.5)
  }
})
```

### Sync particle burst to the beat

```tsx
import { useAudioScheduler } from 'cubeforge'

function BeatParticles() {
  const scheduler = useAudioScheduler({ bpm: 140 })

  scheduler.onBeat((beat) => {
    if (beat === 0) emitter.burst(20) // big burst on bar start
    else emitter.burst(5)
  })

  useEffect(() => {
    scheduler.start()
    return () => scheduler.stop()
  }, [])

  return null
}
```

### Unsubscribing a handler

```tsx
const unsub = scheduler.onBeat((beat) => {
  console.log('beat', beat)
})

// Later, when you no longer want the callback:
unsub()
```

## How it works

The scheduler uses the **lookahead pattern** — every `intervalMs` milliseconds it checks `AudioContext.currentTime` and fires callbacks for all beats that fall within the next `lookaheadSec` seconds. This decouples scheduling from the main thread's timer precision, giving you stable timing even under heavy CPU load.

```
Wall clock timer (every 25ms)
  └── check AudioContext.currentTime
  └── schedule all beats within lookahead window
        └── fire onBeat with the AudioContext timestamp
              └── play sound at that exact timestamp
```

## Notes

- `start()` is a no-op if the scheduler is already running — safe to call repeatedly.
- `stop()` does not clear registered handlers. Call the unsubscribe function returned by `onBeat` / `onBar` to remove a specific handler.
- The scheduler cleans up its internal `setInterval` automatically when the component unmounts.
- The `time` parameter in beat handlers is in AudioContext seconds. Pass it to `AudioBufferSourceNode.start(time)` or use it to schedule `GainNode.gain.setValueAtTime` for sample-accurate automation.

## See also

- [useSound](/api/use-sound) — load and play sounds, supports scheduled start via `play({ delay })`
- [useAudioAnalyser](/api/use-audio-analyser) — real-time frequency and beat detection
- [Audio Groups](/api/audio-groups) — control volume per group
