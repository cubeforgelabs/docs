# usePreloadAudio

Preloads multiple audio files in parallel and tracks loading progress. Decoded buffers are stored in the shared cache — subsequent `useSound` calls with the same paths reuse the already-decoded data without fetching again.

## Signature

```ts
function usePreloadAudio(srcs: string[]): PreloadAudioResult
```

## Result

| Property | Type | Description |
|---|---|---|
| `loaded` | number | Number of files that have finished loading (success or failure) |
| `total` | number | Total number of files requested |
| `progress` | number | Progress 0–1. Reaches 1 when all files have settled |
| `isReady` | boolean | `true` when all files have finished loading |
| `errors` | string[] | Paths that failed to fetch or decode |

## Examples

### Loading screen before the game starts

```tsx
import { usePreloadAudio } from 'cubeforge'

const AUDIO_ASSETS = [
  '/sfx/jump.wav',
  '/sfx/explosion.wav',
  '/sfx/coin.wav',
  '/music/theme.ogg',
]

function App() {
  const { progress, isReady } = usePreloadAudio(AUDIO_ASSETS)

  if (!isReady) {
    return (
      <div className="loading">
        <div style={{ width: `${progress * 100}%` }} className="bar" />
        <p>Loading audio… {Math.round(progress * 100)}%</p>
      </div>
    )
  }

  return <Game>{/* your game */}</Game>
}
```

### Show errors for missing files

```tsx
const { isReady, errors } = usePreloadAudio(['/sfx/jump.wav', '/sfx/missing.wav'])

if (errors.length > 0) {
  console.warn('Failed to load:', errors)
}
```

### Combined with other asset loading

```tsx
const audio = usePreloadAudio(AUDIO_ASSETS)
const sprites = usePreload(['/sprites/player.png', '/sprites/tileset.png'])

const isReady = audio.isReady && sprites.isReady
const progress = (audio.progress + sprites.progress) / 2
```

## Notes

- Buffers are loaded once — duplicate paths in the array are deduplicated automatically.
- Files already in the cache (from a previous `usePreloadAudio` or `useSound` call) are skipped without fetching again.
- Failed files are counted as settled — `isReady` becomes `true` when all files have completed, whether or not they errored.
- The `srcs` array is joined to a key string internally. The effect only re-runs when the set of paths actually changes, not when the array reference changes.

## See also

- [useSound](/api/use-sound) — play preloaded sounds
- [useMusic](/api/use-music) — global music player with crossfade
- [useStreamedMusic](/api/use-streamed-music) — stream large audio files without preloading
