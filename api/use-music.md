# useMusic

Global singleton music player with automatic crossfade when the source changes. Only one music track plays at a time — switching `src` fades the old track out while the new one fades in.

## Signature

```ts
function useMusic(src: string, opts?: MusicOptions): MusicControls
```

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `volume` | number | `1` | Volume (0–1) |
| `crossfadeDuration` | number | `1` | Seconds to crossfade when the source changes |
| `group` | string | `'music'` | Audio group for this track |
| `autoplay` | boolean | `false` | Start playing immediately on mount (requires a prior user gesture) |

## MusicControls

| Method | Description |
|---|---|
| `play()` | Start or resume playback |
| `stop()` | Stop playback |
| `setVolume(v)` | Set volume (0–1) without affecting the group or master volume |
| `fadeIn(duration)` | Ramp from 0 to current volume over `duration` seconds |
| `fadeOut(duration)` | Ramp to 0 over `duration` seconds, then stop |

## Examples

### Simple background music

```tsx
import { useMusic } from 'cubeforge'
import { useEffect } from 'react'

function BackgroundMusic() {
  const music = useMusic('/music/theme.ogg', { volume: 0.5 })

  useEffect(() => {
    music.play()
    return () => music.stop()
  }, [])

  return null
}
```

### Switch tracks with crossfade

Because `useMusic` is a singleton, changing `src` automatically crossfades:

```tsx
function LevelMusic({ level }: { level: number }) {
  const track = level === 1 ? '/music/cave.ogg' : '/music/boss.ogg'

  // Changing 'track' crossfades from the previous track over 2 seconds
  useMusic(track, { crossfadeDuration: 2 })

  return null
}
```

### Fade in on mount, fade out on unmount

```tsx
function IntroMusic() {
  const music = useMusic('/music/intro.ogg', { group: 'music' })

  useEffect(() => {
    music.fadeIn(2)
    return () => music.fadeOut(1.5)
  }, [])

  return null
}
```

## Notes

- `useMusic` manages a global singleton — if two components both call `useMusic`, the second call takes over the player. Only one track plays at a time.
- For independent tracks playing simultaneously, use `useSound` with `loop: true` instead.
- For large files that should not be fully decoded into memory, use `useStreamedMusic`.

## See also

- [useSound](/api/use-sound) — multi-instance sound playback
- [useStreamedMusic](/api/use-streamed-music) — stream large audio files without decoding into memory
- [Audio Groups](/api/audio-groups) — control volume per group
