# useStreamedMusic

Stream a long-form audio file via an HTML `<audio>` element connected into the Web Audio graph. The file is **never fully decoded into memory** — the browser streams it on demand. Use this for large music tracks, hour-long ambience loops, or any file too large to preload.

## Signature

```ts
function useStreamedMusic(src: string, opts?: StreamedMusicOptions): StreamedMusicControls
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `volume` | number | `1` | Initial volume (0–1) |
| `loop` | boolean | `false` | Loop the track |
| `group` | string | — | Audio group name (routes through `setGroupVolume`) |
| `autoplay` | boolean | `false` | Start playing on mount (requires a prior user gesture) |

## StreamedMusicControls

| Method / Property | Description |
|---|---|
| `play()` | Start or resume playback |
| `pause()` | Pause, preserving the current position |
| `stop()` | Stop and rewind to the beginning |
| `setVolume(v)` | Set volume (0–1) |
| `isPlaying` | Whether the track is currently playing |

## Examples

### Large background track

```tsx
import { useStreamedMusic } from 'cubeforge'

function AmbientLoop() {
  const music = useStreamedMusic('/music/ambient-forest.mp3', {
    loop: true,
    group: 'music',
    volume: 0.6,
  })

  return (
    <button onClick={() => music.isPlaying ? music.pause() : music.play()}>
      {music.isPlaying ? 'Pause' : 'Play'}
    </button>
  )
}
```

### Autoplay on mount

```tsx
function CinematicTrack() {
  useStreamedMusic('/cutscene/intro.ogg', { autoplay: true, volume: 0.8 })
  return null
}
```

### Smooth transition using group fade

```tsx
import { useStreamedMusic, setGroupVolumeFaded } from 'cubeforge'

function LevelComplete() {
  const music = useStreamedMusic('/music/victory.ogg', { group: 'music' })

  const handleVictory = () => {
    setGroupVolumeFaded('music', 0, 1) // fade out over 1s
    setTimeout(() => {
      music.play()
      setGroupVolumeFaded('music', 1, 1) // fade in
    }, 1000)
  }

  return <button onClick={handleVictory}>Complete Level</button>
}
```

## Streamed vs decoded audio

| | `useStreamedMusic` | `useSound` / `useMusic` |
|---|---|---|
| Memory usage | Low — browser streams on demand | Higher — full buffer decoded |
| Best for | Long tracks, large files | Short SFX, music needing crossfade |
| Crossfade | Via `setGroupVolumeFaded` | Native via `crossfadeTo` / `useMusic` |
| Looping | ✅ | ✅ |
| Seek to position | ✅ (via `audio.currentTime`) | ❌ |

## Notes

- The underlying `<audio>` element uses `preload="none"` — no data is fetched until `play()` is called.
- `isPlaying` reflects the actual play state from the `<audio>` element's `play` / `pause` / `ended` events.
- There is no sample-accurate crossfade — use `setGroupVolumeFaded` for smooth transitions between streamed tracks.
- Changing `src` (the first argument) creates a new `<audio>` element and stops the previous track.

## See also

- [useMusic](/api/use-music) — in-memory music player with automatic crossfade
- [useSound](/api/use-sound) — decoded buffer playback for SFX
- [Audio Groups](/api/audio-groups) — shared volume controls
