# useAudioAnalyser

Taps into an audio group's output in real time, exposing frequency and waveform data for visualizers, beat detection, and reactive effects. The analyser is a **passive listener** — it never interrupts the signal path.

## Signature

```ts
function useAudioAnalyser(opts?: AudioAnalyserOptions): AudioAnalyserControls
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `group` | string | `'master'` | Audio group to tap (`'sfx'`, `'music'`, `'master'`, or any custom name) |
| `fftSize` | number | `2048` | FFT size — controls frequency resolution. Must be a power of 2 between 32 and 32768. `frequencyBinCount = fftSize / 2` |
| `smoothingTimeConstant` | number | `0.8` | Smoothing 0–1. Higher = smoother but slower to react. Lower = snappier but noisier |

## AudioAnalyserControls

| Method / Property | Description |
|---|---|
| `getFrequencyData()` | Returns a `Uint8Array` of magnitude values (0–255) per frequency bin. Call every frame for live data |
| `getWaveformData()` | Returns a `Uint8Array` of time-domain (waveform) samples. 128 = silence |
| `getAverageFrequency()` | Average magnitude across all bins (0–255). Quick loudness/energy measure |
| `isBeat(threshold?)` | Returns `true` when bass-range energy exceeds `threshold` (default `180`). Useful for triggering effects on each beat |
| `analyserNode` | Direct access to the underlying `AnalyserNode`. `null` before mount |

## Examples

### Spectrum visualizer

```tsx
import { useAudioAnalyser } from 'cubeforge'
import { useRef, useEffect } from 'react'

function Visualizer() {
  const analyser = useAudioAnalyser({ group: 'music', fftSize: 512 })
  const canvasRef = useRef<HTMLCanvasElement>(null)

  useEffect(() => {
    const canvas = canvasRef.current!
    const ctx = canvas.getContext('2d')!
    let raf: number

    const draw = () => {
      raf = requestAnimationFrame(draw)
      const freq = analyser.getFrequencyData()
      ctx.clearRect(0, 0, canvas.width, canvas.height)
      const barW = canvas.width / freq.length
      for (let i = 0; i < freq.length; i++) {
        const h = (freq[i] / 255) * canvas.height
        ctx.fillStyle = `hsl(${i * 2}, 80%, 50%)`
        ctx.fillRect(i * barW, canvas.height - h, barW, h)
      }
    }
    draw()
    return () => cancelAnimationFrame(raf)
  }, [])

  return <canvas ref={canvasRef} width={400} height={100} />
}
```

### Beat detection — camera shake

```tsx
import { useAudioAnalyser, useCamera } from 'cubeforge'

function BeatReactor() {
  const analyser = useAudioAnalyser({ group: 'music' })
  const camera = useCamera()

  return (
    <Script
      update={() => {
        if (analyser.isBeat(160)) {
          camera.shake(4, 0.1) // shake 4px for 0.1s on each beat
        }
      }}
    />
  )
}
```

### Average loudness — scale UI element

```tsx
function LoudnessMeter() {
  const analyser = useAudioAnalyser({ group: 'sfx' })
  const [scale, setScale] = useState(1)

  useEffect(() => {
    let raf: number
    const tick = () => {
      raf = requestAnimationFrame(tick)
      const avg = analyser.getAverageFrequency() / 255
      setScale(1 + avg * 0.5)
    }
    tick()
    return () => cancelAnimationFrame(raf)
  }, [])

  return (
    <div style={{ transform: `scale(${scale})`, transition: 'transform 0.05s' }}>
      🔊
    </div>
  )
}
```

## Notes

- The analyser taps the group's gain node as a fan-out — audio still flows normally to the speakers.
- `getFrequencyData()` and `isBeat()` both internally call `getByteFrequencyData` — calling both in the same frame only reads the analyser once (they share the same backing buffer).
- Smaller `fftSize` values (e.g. 256) are faster and fine for beat detection. Larger values (e.g. 4096) give better frequency resolution for visualizers.
- The returned `Uint8Array` from `getFrequencyData()` and `getWaveformData()` is reused across calls — copy it if you need to compare frames.

## See also

- [Audio Groups](/api/audio-groups) — control volume per group
- [useSound](/api/use-sound) — load and play sounds
- [useAudioScheduler](/api/use-audio-scheduler) — BPM-synced beat scheduling
