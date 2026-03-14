# Audio Settings Persistence

`saveAudioSettings` and `loadAudioSettings` persist master and group volumes to `localStorage` so player preferences survive page reloads.

## Functions

```ts
import { saveAudioSettings, loadAudioSettings } from 'cubeforge'
```

| Function | Description |
|---|---|
| `saveAudioSettings()` | Serialise current master and all group volumes to `localStorage` |
| `loadAudioSettings()` | Restore previously saved volumes from `localStorage` |

## Usage

Call `loadAudioSettings()` **once on startup** before any audio plays, then call `saveAudioSettings()` **whenever the player changes a volume setting**.

```tsx
import {
  loadAudioSettings,
  saveAudioSettings,
  setMasterVolume,
  setGroupVolume,
} from 'cubeforge'
import { useEffect } from 'react'

function App() {
  // Restore saved volumes on first render
  useEffect(() => {
    loadAudioSettings()
  }, [])

  return <Game>{/* ... */}</Game>
}
```

### Volume settings panel

```tsx
function AudioSettings() {
  const handleMaster = (e: React.ChangeEvent<HTMLInputElement>) => {
    setMasterVolume(Number(e.target.value))
    saveAudioSettings() // persist immediately
  }

  const handleMusic = (e: React.ChangeEvent<HTMLInputElement>) => {
    setGroupVolume('music', Number(e.target.value))
    saveAudioSettings()
  }

  const handleSFX = (e: React.ChangeEvent<HTMLInputElement>) => {
    setGroupVolume('sfx', Number(e.target.value))
    saveAudioSettings()
  }

  return (
    <div>
      <label>Master <input type="range" min={0} max={1} step={0.01} onChange={handleMaster} /></label>
      <label>Music  <input type="range" min={0} max={1} step={0.01} onChange={handleMusic} /></label>
      <label>SFX    <input type="range" min={0} max={1} step={0.01} onChange={handleSFX} /></label>
    </div>
  )
}
```

## What is persisted

Every group volume set via `setGroupVolume` and the master volume set via `setMasterVolume` are saved under the key `cubeforge:audio` in `localStorage`.

```json
{ "master": 0.7, "music": 0.4, "sfx": 0.9, "ambient": 0.5 }
```

`loadAudioSettings` reads this object and calls `setMasterVolume` / `setGroupVolume` for each entry, restoring the audio graph to the saved state.

## Notes

- Both functions are **silent no-ops** when `localStorage` is unavailable (server-side rendering, private browsing, quota exceeded).
- `loadAudioSettings` ignores corrupt JSON and non-numeric values — the audio graph stays at defaults if the saved data is invalid.
- Only volumes are persisted — mute state, effects, and per-sound volumes are not.
- Calling `saveAudioSettings` before any volume has been explicitly set saves all groups at their default value of `1`.

## See also

- [Audio Groups](/api/audio-groups) — `setGroupVolume`, `setMasterVolume`, and group management
- [useSound](/api/use-sound) — load and play sounds
