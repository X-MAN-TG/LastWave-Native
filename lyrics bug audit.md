# LastWave Native: Lyrics and Player File Guide

This document explains which files control **Lyrics** and **Player**, from external APIs to what the user sees. Use it as a reference when preparing updates.

## Overall architecture

```text
External API / local files
        ↓
Data API classes
        ↓
LyricsRepository / MusicPlayer
        ↓
ViewModel + StateFlow
        ↓
Jetpack Compose UI
        ↓
What the user sees
```

---

# 1. Lyrics

## 1.1 Central lyrics model and orchestration

### Main file

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
```

This is the most important lyrics control file. It defines the internal lyric models and coordinates all providers.

### Main models

`LyricSyllable` represents word or syllable-level timing:

```kotlin
data class LyricSyllable(
    val timeMs: Long,
    val durationMs: Long,
    val text: String,
    val isBackground: Boolean = false,
    val appendToPrevious: Boolean = false,
)
```

`LyricLine` represents one lyric line:

```kotlin
data class LyricLine(
    val timeMs: Long,
    val durationMs: Long = 0L,
    val text: String,
    val syllables: List<LyricSyllable> = emptyList(),
    val transliteration: String? = null,
    val transliterationSyllables: List<LyricSyllable> = emptyList(),
)
```

The models support:

- Plain lyrics
- Line-synced lyrics
- Word-synced lyrics
- Syllable-level highlighting
- Transliteration
- Background vocals
- RTL languages such as Arabic, Hebrew, Urdu, and Persian
- Word fragments that should not receive an inserted space

`LyricsResult` is the result passed to the UI:

```text
LyricsResult.Success
LyricsResult.Empty
LyricsResult.Error
```

A successful result contains:

```text
lines
isSynced
isWordSynced
plainLyrics
isInstrumental
source
```

### Change this file when you need to update

- Provider priority
- Provider fallback behavior
- Caching
- Lyrics matching
- Duration matching
- Local/offline lyric lookup
- Word-synced versus line-synced preference
- Lyrics result conversion
- Adding a new provider to the repository

---

## 1.2 Lyrics API providers

### Apple Music provider

```text
app/src/main/java/com/lastwave/app/data/lyrics/AppleMusicLyricsApi.kt
```

Uses:

```text
https://itunes.apple.com/search
https://lyrics.paxsenix.org/apple-music/lyrics
```

Responsible for:

- Searching for the track through iTunes
- Resolving the track ID
- Fetching Apple Music-style lyrics
- Syllable timing
- Transliteration
- Converting the response to `LyricLine` and `LyricSyllable`

This is the default provider in the current settings configuration.

### LyricsPlus provider

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsPlusApi.kt
```

Endpoint:

```text
https://lyricsplus.prjktla.my.id/v2/lyrics/get
```

Supports:

- Word-synced lyrics
- Line-synced lyrics
- Album and duration matching
- Retry without duration
- Cleaned title and artist queries

### BetterLyrics provider

```text
app/src/main/java/com/lastwave/app/data/lyrics/BetterLyricsApi.kt
```

Endpoints:

```text
https://lyrics-api.boidu.dev/getLyrics
https://lyrics-api.boidu.dev/ttml/getLyrics
```

Responsible for word-synced lyrics and TTML lyric parsing.

### Kugou provider

```text
app/src/main/java/com/lastwave/app/data/lyrics/KugouLyricsApi.kt
```

Endpoints include:

```text
https://lyrics.kugou.com/search
https://lyrics.kugou.com/download
```

Responsible for:

- Kugou search
- Selecting a matching result
- Downloading lyric content
- Parsing KRC timing
- Converting KRC data to the internal lyric models

### LRCLIB provider

```text
app/src/main/java/com/lastwave/app/data/lyrics/LrclibLyricsApi.kt
```

Endpoints:

```text
https://lrclib.net/api/get
https://lrclib.net/api/search
```

Primarily provides line-synced or plain lyrics. It matches using:

- Track title
- Artist
- Album
- Duration
- Version markers such as Live, Remix, Acoustic, and Cover
- Duration tolerance

Title and artist cleanup logic is also located here.

---

## 1.3 Provider selection and fallback flow

The main orchestration is in:

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
```

General flow:

1. Read the selected provider from settings.
2. Build a cache key from artist, title, album, duration, word-by-word mode, and provider.
3. Check the in-memory cache.
4. Check downloaded/offline lyrics.
5. Try the selected provider.
6. If word-by-word mode is enabled, try word-synced providers, including parallel requests.
7. Fall back to LRCLIB.
8. Return a successful, empty, instrumental, or error result.

The repository also supports partial results:

```kotlin
onPartialResult: suspend (LyricsResult.Success) -> Unit
```

This allows the UI to show a line-synced result while the app continues looking for a better word-synced result.

### Change provider priority here

```text
LyricsRepository.kt
```

---

## 1.4 Downloaded and offline lyrics

Relevant files:

```text
app/src/main/java/com/lastwave/app/data/local/db/DownloadedTrackEntity.kt
app/src/main/java/com/lastwave/app/data/local/db/DownloadedTrackDao.kt
app/src/main/java/com/lastwave/app/data/download/TrackDownloadManager.kt
app/src/main/java/com/lastwave/app/data/download/AudioTagWriter.kt
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
```

The repository can read:

- Embedded plain lyrics
- Embedded synced lyrics
- Saved `.lrc` files
- Downloaded track metadata

Possible sources returned by the repository include:

```text
Downloaded Lyrics (LRC)
Downloaded Lyrics (Plain)
```

Update these files if you need to change how lyrics are embedded in downloaded files or saved as companion LRC files:

```text
AudioTagWriter.kt
TrackDownloadManager.kt
```

---

## 1.5 Lyrics settings and user controls

### Settings model

```text
app/src/main/java/com/lastwave/app/data/local/SettingsPreferences.kt
```

Important settings:

```text
lyricsUiVersion
lyricsAnimation
lyricsProvider
wordByWordLyrics
downloadLyrics
```

Provider options are declared by `LyricsProvider` in this file. Current options include:

```text
Apple Music
LyricsPlus
BetterLyrics
Kugou
LRCLIB
```

DataStore keys include:

```text
lw_lyrics_ui_version
lw_word_by_word_lyrics
lw_lyrics_animation
lw_lyrics_provider
lw_download_lyrics
```

### Settings ViewModel

```text
app/src/main/java/com/lastwave/app/ui/settings/SettingsViewModel.kt
```

Important methods:

```kotlin
setLyricsUiVersion(...)
setWordByWordLyrics(...)
setLyricsAnimation(...)
setLyricsProvider(...)
setDownloadLyrics(...)
```

### Settings UI

```text
app/src/main/java/com/lastwave/app/ui/settings/SettingsScreen.kt
```

This controls the visible settings for:

- Lyrics provider
- Lyrics UI version
- Lyrics animation
- Word-by-word lyrics
- Download lyrics

If adding a new provider or setting, normally update:

```text
SettingsPreferences.kt
SettingsViewModel.kt
SettingsScreen.kt
LyricsRepository.kt
```

---

## 1.6 Lyrics loading and UI state

Lyrics loading is controlled by `PlayerViewModel`, which is inside:

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
```

It injects `LyricsRepository` and exposes:

```kotlin
val lyricsState = _lyricsState.asStateFlow()
```

The ViewModel reloads lyrics when:

- The current track changes
- Word-by-word mode changes
- The user presses retry

Important methods:

```kotlin
loadLyrics(track: PlayableTrack, forceRefresh: Boolean = false)
retryLyrics()
publishLyrics(result: LyricsResult)
```

The ViewModel converts repository results into:

```text
LyricsUiState.Idle
LyricsUiState.Loading
LyricsUiState.Success
LyricsUiState.Empty
LyricsUiState.Error
```

It also:

- Cancels the previous request when the track changes
- Delays the loading spinner to avoid flashing it on quick cache hits
- Keeps previous lyrics visible briefly during transitions
- Publishes partial results
- Uses current track duration for API matching
- Supports forced refresh

---

## 1.7 Lyrics UI

### Standard/legacy lyrics UI

```text
app/src/main/java/com/lastwave/app/ui/player/LyricsView.kt
```

Defines:

```text
LyricsUiState
LyricsView
PlainLyricsView
EmptyLyricsView
```

Controls:

- Loading screen
- Error screen
- Empty lyrics screen
- Plain lyrics
- Line-synced lyrics
- Active-line highlighting
- Scrolling
- RTL layout
- Retry button
- Fullscreen controls

### Modern lyrics UI

```text
app/src/main/java/com/lastwave/app/ui/player/ModernLyricsView.kt
```

Main composable:

```kotlin
ModernLyricsPanel(...)
```

Controls:

- Animated state transitions
- Word-synced karaoke lyrics
- Line-synced lyrics
- Plain lyrics
- Instrumental state
- Fullscreen mode
- Retry action
- Smooth playback-position interpolation
- Active-line scrolling
- RTL layout
- Apple Music-specific sizing
- Syllable highlighting

It uses lyric components such as:

```text
KaraokeLyricsView
SyncedLyrics
KaraokeLine
KaraokeSyllable
```

If you want to change what the user sees while lyrics are playing, start with:

```text
ModernLyricsView.kt
LyricsView.kt
```

The lyrics panel is placed into the player screen from:

```text
PlayerHost.kt
```

---

## Lyrics update guide

### Change an API

Update the relevant provider file:

```text
data/lyrics/<Provider>LyricsApi.kt
```

Convert the API response into:

```text
LyricLine
LyricSyllable
LyricsResult.Success
```

### Change provider priority

Update:

```text
LyricsRepository.kt
```

### Change matching accuracy

Update:

```text
LrclibLyricsApi.kt
LyricsRepository.kt
```

### Change word-by-word timing

Update:

```text
LyricsRepository.kt
<provider API file>
ModernLyricsView.kt
```

### Change visual appearance

Update:

```text
ModernLyricsView.kt
LyricsView.kt
PlayerHost.kt
```

---

# 2. Player

The player has three main layers:

```text
Playback engine and state
        ↓
MusicPlayer.kt

Background and system playback
        ↓
MusicPlaybackService.kt

Visible in-app player
        ↓
PlayerHost.kt
```

---

## 2.1 Main playback engine

### Main file

```text
app/src/main/java/com/lastwave/app/playback/MusicPlayer.kt
```

This is the central player controller. It is a singleton and owns the process-wide Media3/ExoPlayer playback engine.

### Main state

`MusicPlayerState` contains fields such as:

```text
current
queue
currentIndex
isPlaying
isBuffering
positionMs
bufferedPositionMs
durationMs
shuffleEnabled
repeatMode
speed
bitrateKbps
audioCodec
isLossless
bitDepth
samplingRateKHz
sleepTimerRemainingMs
error
```

### Optimized state models

`PlaybackChromeState` is used by the mini-player and collapsed player UI. It avoids unnecessary updates caused by the frequent position ticker.

`PlaybackProgressState` is used by:

- Seek bar
- Current playback position
- Duration display
- Lyrics synchronization

---

## 2.2 Track model

Inside `MusicPlayer.kt`:

```kotlin
PlayableTrack
```

Contains:

```text
title
artist
album
artworkUrl
videoId
playbackUrl
playbackMimeType
```

This model is passed through search, albums, playlists, queue, player, lyrics, notifications, widgets, and Android Auto.

If you add a track property that must be available across all player surfaces, update this model and its serialization/conversion code.

---

## 2.3 Track playback flow

```text
User taps a track
        ↓
MusicPlayer.play(...)
        ↓
playQueue(...)
        ↓
Resolve audio stream
        ↓
Create MediaItem
        ↓
ExoPlayer.setMediaItems(...)
        ↓
ExoPlayer.prepare()
        ↓
ExoPlayer.play()
        ↓
MusicPlayerState updates
        ↓
PlayerHost and system controls update
```

Important methods in `MusicPlayer.kt` include:

```kotlin
play(...)
playQueue(...)
playDiscoverQueue(...)
playNext(...)
addToQueue(...)
resume()
pause()
togglePlayPause()
seekTo(...)
next()
previous()
setShuffleEnabled(...)
setRepeatMode(...)
seekToQueueItem(...)
```

---

## 2.4 Audio stream resolution

The player may need to resolve a playable stream URL before ExoPlayer can start.

Relevant files:

```text
app/src/main/java/com/lastwave/app/data/music/InnerTubeMusicApi.kt
app/src/main/java/com/lastwave/app/data/music/InnerTubeXStreamExtractor.kt
app/src/main/java/com/lastwave/app/data/music/YouTubeStreamExtractor.kt
```

Lossless playback:

```text
app/src/main/java/com/lastwave/app/data/lossless/LosslessMusicApi.kt
```

Plugin/module-based playback:

```text
app/src/main/java/com/lastwave/app/data/plugin/ModulePlaybackResolver.kt
app/src/main/java/com/lastwave/app/data/plugin/ModuleDrmFactory.kt
app/src/main/java/com/lastwave/app/data/plugin/SegmentedDashBridge.kt
```

The resolver selects the stream, detects MIME type, and passes the final URL to ExoPlayer.

For stream-resolution updates, inspect:

```text
MusicPlayer.kt
InnerTubeMusicApi.kt
InnerTubeXStreamExtractor.kt
YouTubeStreamExtractor.kt
LosslessMusicApi.kt
ModulePlaybackResolver.kt
```

---

## 2.5 ExoPlayer configuration

`MusicPlayer.kt` configures:

- ExoPlayer
- Audio attributes
- Buffering
- Data sources
- Cache
- Renderers
- DRM
- MIME types
- Network playback
- Audio sinks
- Retry behavior
- Crossfade
- Preloading

Important classes include:

```text
DefaultLoadControl
DefaultRenderersFactory
ExoPlayer
DefaultAudioSink
CacheDataSource
SimpleCache
```

Change `MusicPlayer.kt` for updates to:

- Buffer size
- Startup speed
- Rebuffering behavior
- Audio format handling
- Retry handling
- Playback cache
- Crossfade
- Preloading

---

## 2.6 Playback controls

### Play and pause

```kotlin
play(...)
resume()
pause()
togglePlayPause()
```

### Seeking

```kotlin
seekTo(positionMs)
```

Seeking affects both the progress UI and lyric synchronization.

### Queue controls

```kotlin
playQueue(...)
addToQueue(...)
playNext(...)
seekToQueueItem(...)
next()
previous()
```

### Shuffle and repeat

```kotlin
setShuffleEnabled(...)
setRepeatMode(...)
```

These values are exposed through `MusicPlayerState` and are also published to the system MediaSession.

---

## 2.7 Background playback service

### File

```text
app/src/main/java/com/lastwave/app/playback/MusicPlaybackService.kt
```

Responsible for:

- Background playback
- MediaSession
- Lock-screen controls
- Notification controls
- Bluetooth controls
- Headset media buttons
- Android Auto
- System media buttons
- Scrobbling hooks
- Widget updates
- Playback wake locks
- Notification updates

The service uses the same singleton `MusicPlayer`; it does not create a separate independent audio player.

---

## 2.8 MediaSession controls

`MusicPlaybackService.kt` maps system commands to `MusicPlayer` methods:

```text
onPlay() → musicPlayer.resume()
onPause() → musicPlayer.pause()
onSkipToNext() → musicPlayer.next()
onSkipToPrevious() → musicPlayer.previous()
onSeekTo() → musicPlayer.seekTo(...)
onStop() → musicPlayer.stopAndClear()
onSkipToQueueItem() → musicPlayer.seekToQueueItem(...)
onSetShuffleMode() → musicPlayer.setShuffleEnabled(...)
onSetRepeatMode() → musicPlayer.setRepeatMode(...)
```

This controls lock-screen, notification, Bluetooth, car, and Android Auto actions.

---

## 2.9 Android manifest service registration

```text
app/src/main/AndroidManifest.xml
```

The background player service is registered as:

```xml
<service
    android:name=".playback.MusicPlaybackService"
    android:exported="true"
    android:stopWithTask="false"
    android:foregroundServiceType="mediaPlayback">
```

If background playback or notification controls fail, inspect:

```text
AndroidManifest.xml
MusicPlaybackService.kt
```

---

## 2.10 Visible full player

### Main UI file

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
```

Main composable:

```kotlin
PlayerHost(...)
```

This consumes:

```text
PlayerViewModel
MusicPlayerState
PlaybackChromeState
PlaybackProgressState
LyricsUiState
```

It controls what the user sees:

- Album artwork
- Track title
- Artist
- Play/pause
- Previous/next
- Seek bar
- Current position
- Duration
- Shuffle
- Repeat
- Queue
- Lyrics
- Favorite button
- Add to playlist
- More options
- Audio quality badge
- Signal path
- Cast
- USB/DAC controls
- Fullscreen player
- Mini-player transitions

`PlayerHost` is reached from:

```text
app/src/main/java/com/lastwave/app/MainActivity.kt
```

---

## 2.11 Mini-player and player chrome

Relevant files:

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
app/src/main/java/com/lastwave/app/ui/common/TrackMiniTraySheet.kt
app/src/main/java/com/lastwave/app/ui/shell/MainShell.kt
```

The mini-player uses `PlaybackChromeState` for:

- Current artwork
- Current title
- Current artist
- Play/pause
- Open full player
- Queue indication

---

## 2.12 Seek bar

Custom seek bar:

```text
app/src/main/java/com/lastwave/app/ui/player/WavySeekBar.kt
```

The seek bar ultimately calls:

```kotlin
musicPlayer.seekTo(...)
```

Update these files for changes to:

- Seek bar appearance
- Waveform style
- Drag behavior
- Haptic feedback
- Seek accuracy
- Progress animation

```text
WavySeekBar.kt
PlayerHost.kt
MusicPlayer.kt
```

---

## 2.13 Artwork

Artwork data and rendering are handled by:

```text
app/src/main/java/com/lastwave/app/data/artwork/ArtworkRepository.kt
app/src/main/java/com/lastwave/app/data/artwork/ArtworkNormalizer.kt
app/src/main/java/com/lastwave/app/ui/common/ArtworkImage.kt
```

Player artwork rendering is mainly inside `PlayerHost.kt`.

The playback service also loads artwork for:

- Notifications
- MediaSession
- Android Auto
- Widgets

For artwork changes across all surfaces, inspect:

```text
MusicPlaybackService.kt
ArtworkRepository.kt
ArtworkImage.kt
PlayerHost.kt
```

---

## 2.14 Audio quality and signal path UI

Relevant files:

```text
app/src/main/java/com/lastwave/app/playback/QualityBadge.kt
app/src/main/java/com/lastwave/app/playback/SignalPath.kt
app/src/main/java/com/lastwave/app/ui/player/SignalPathDialog.kt
```

These display:

- Lossless status
- Bitrate
- Codec
- Bit depth
- Sampling rate
- USB output
- Exclusive USB mode
- DAC path
- Native audio path

Additional audio files:

```text
AudioEffectsEngine.kt
NativeAudioEngine.kt
NativePcmAudioProcessor.kt
NativeProcessingAudioSink.kt
UsbBitPerfectOutput.kt
UsbDacMonitor.kt
ExclusiveUsbOutput.kt
```

---

## 2.15 Native audio layer

C++ audio engine:

```text
app/src/main/cpp/AudioEngine.cpp
app/src/main/cpp/AudioEngine.h
app/src/main/cpp/NativeBridge.cpp
app/src/main/cpp/DspProcessor.cpp
app/src/main/cpp/DspProcessor.h
```

Kotlin bridge:

```text
app/src/main/java/com/lastwave/app/playback/NativeAudioEngine.kt
app/src/main/java/com/lastwave/app/playback/NativePcmAudioProcessor.kt
app/src/main/java/com/lastwave/app/playback/NativeProcessingAudioSink.kt
```

This layer is for:

- DSP
- PCM processing
- Audio effects
- Native output
- Low-level audio routing

It is not normally required for standard player UI updates.

---

# Most important files

## Lyrics API and behavior

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
app/src/main/java/com/lastwave/app/data/lyrics/AppleMusicLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/LyricsPlusApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/BetterLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/KugouLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/LrclibLyricsApi.kt
```

## Lyrics loading and state

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
```

Important section:

```text
PlayerViewModel
loadLyrics(...)
publishLyrics(...)
retryLyrics()
```

## Lyrics appearance

```text
app/src/main/java/com/lastwave/app/ui/player/LyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/ModernLyricsView.kt
```

## Lyrics settings

```text
app/src/main/java/com/lastwave/app/data/local/SettingsPreferences.kt
app/src/main/java/com/lastwave/app/ui/settings/SettingsViewModel.kt
app/src/main/java/com/lastwave/app/ui/settings/SettingsScreen.kt
```

## Core player

```text
app/src/main/java/com/lastwave/app/playback/MusicPlayer.kt
```

## Background and system controls

```text
app/src/main/java/com/lastwave/app/playback/MusicPlaybackService.kt
app/src/main/AndroidManifest.xml
```

## Player UI

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
app/src/main/java/com/lastwave/app/ui/player/WavySeekBar.kt
app/src/main/java/com/lastwave/app/ui/player/SignalPathDialog.kt
app/src/main/java/com/lastwave/app/ui/player/PlayerCastMenuRow.kt
```

## Audio stream resolution

```text
app/src/main/java/com/lastwave/app/data/music/InnerTubeMusicApi.kt
app/src/main/java/com/lastwave/app/data/music/InnerTubeXStreamExtractor.kt
app/src/main/java/com/lastwave/app/data/music/YouTubeStreamExtractor.kt
app/src/main/java/com/lastwave/app/data/lossless/LosslessMusicApi.kt
app/src/main/java/com/lastwave/app/data/plugin/ModulePlaybackResolver.kt
```

---

# Recommended update order

## For a lyrics update

```text
1. Provider API file
2. LyricsRepository.kt
3. PlayerViewModel in PlayerHost.kt
4. LyricsUiState, if needed
5. ModernLyricsView.kt
6. SettingsPreferences.kt, if adding a user option
7. SettingsScreen.kt
8. Lyrics tests
```

Existing relevant tests:

```text
app/src/test/java/com/lastwave/app/data/lyrics/LyricsRtlTest.kt
app/src/test/java/com/lastwave/app/ui/player/KaraokeLineSplitterTest.kt
```

## For a player update

```text
1. MusicPlayer.kt
2. MusicPlaybackService.kt if system/background controls are affected
3. PlayerHost.kt
4. WavySeekBar.kt if seek behavior is affected
5. Stream/API resolver files if audio URLs are affected
6. Artwork or signal-path files if those surfaces are affected
7. Player tests
```

Existing player-related tests:

```text
app/src/test/java/com/lastwave/app/playback/DspResponseTest.kt
app/src/test/java/com/lastwave/app/playback/QualityBadgeTest.kt
app/src/test/java/com/lastwave/app/playback/ExclusiveUsbSignalPathTest.kt
app/src/test/java/com/lastwave/app/ui/player/KaraokeLineSplitterTest.kt
```

---

# Quick summary

- **`LyricsRepository.kt`** controls where lyrics come from and how fallback works.
- **`PlayerViewModel` inside `PlayerHost.kt`** connects lyrics data to the current track.
- **`ModernLyricsView.kt`** controls how lyrics look and synchronize visually.
- **`MusicPlayer.kt`** controls actual playback, queue, position, stream resolution, shuffle, repeat, and state.
- **`MusicPlaybackService.kt`** controls background playback, notification, lock screen, Bluetooth, and Android Auto.
- **`PlayerHost.kt`** controls the player visible inside the app.

LYRICS_PLAYER_VISUAL_BUG_AUDIT.md
+1257
# Lyrics and Player Visual Bug Audit

## Scope

This audit focuses on the code that controls the in-app full player, modern lyrics, legacy lyrics, progress synchronization, and player controls.

Primary files inspected:

```text
app/src/main/java/com/lastwave/app/ui/player/ModernLyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/LyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
app/src/main/java/com/lastwave/app/playback/MusicPlayer.kt
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
app/src/main/java/com/lastwave/app/data/local/SettingsPreferences.kt
```

This is a code-level audit. Some issues are definite from the implementation; visual severity should still be verified on small phones, large phones, tablets, gesture navigation, 60 Hz, 120 Hz, long titles, RTL lyrics, and external display/font-scale settings.

---

# Executive summary

The largest user-visible risks are:

1. **Long word-sync lines can still be clipped or visually spill outside the viewport.** The modern renderer estimates library padding and focused-line scale instead of measuring the actual rendered row. `KaraokeLyricsView` is still responsible for the final layout.
2. **Pre-splitting long lines can damage lyric semantics and timing.** A wrapped visual sub-line is represented as a new timed lyric line, which can cause incorrect active-line highlighting, wrong auto-scroll targets, duplicated line timing, and awkward seek behavior.
3. **The karaoke renderer has no explicit safe bottom inset for all device configurations.** The fixed `offset = 84.dp`, weighted content, top badge, and bottom controls can cause the last visible line to sit beneath controls or be cut by navigation/gesture insets.
4. **The current smooth clock is not a true audio-position clock.** It advances from `SystemClock.elapsedRealtime()` and uses a 15% drift correction. During buffering, stalls, speed changes, crossfade, or playback-rate changes, lyrics can drift from audio.
5. **The library and custom renderers do not share one canonical line layout.** Modern mode uses `KaraokeLyricsView`; legacy mode uses `LazyColumn`/`FlowRow`. Fixes applied to one mode will not automatically fix the other.
6. **The player screen has several fixed dimensions and nested weighted layouts.** Artwork, tabs, controls, lyrics, seekbar, fullscreen mode, and system insets compete for vertical space, especially on short phones and landscape.
7. **The legacy word renderer creates one Compose `Text` per syllable.** This makes wrapping, spacing, baseline alignment, scaling, and RTL behavior less reliable than measuring/drawing one line as one layout.
8. **Line timing has fallback assumptions that are not always musically correct.** Several places use fixed 4.5–5 second durations or a minimum one second duration, which can keep a line highlighted too long or too short.

---

# 1. Lyrics rendering architecture

## 1.1 Data path

```text
Lyrics API
  ↓
LyricsRepository
  ↓
LyricsResult.Success
  ↓
PlayerViewModel.publishLyrics()
  ↓
LyricsUiState.Success
  ↓
PlayerHost
  ↓
ModernLyricsPanel or LyricsPanel
  ↓
KaraokeLyricsView or legacy SyncedLyricsList
```

The ViewModel is inside:

```text
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
```

The visual renderers are:

```text
app/src/main/java/com/lastwave/app/ui/player/ModernLyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/LyricsView.kt
```

---

# 2. Definite and probable lyrics bugs

## LYR-01 — Long modern word-sync lines can still be clipped

**Severity: P0 / critical visual bug**

### Code

```text
ModernLyricsView.kt:322-408
```

The code uses:

```kotlin
private val KaraokeHorizontalChrome = 112.dp

val wrapBudgetPx = (maxWidth - KaraokeHorizontalChrome).toPx() * 0.88f
```

Then it pre-splits lines before passing them to:

```kotlin
KaraokeLyricsView(...)
```

### Why this is fragile

The implementation guesses:

- Library outer padding
- Library inner padding
- Line padding
- Focused-line scale
- Scrim/padding behavior

Those values can change with:

- Library version
- Font scale
- Device density
- RTL layout
- Translation/phonetic rows
- Accompaniment rows
- Different font metrics
- Tablet width
- Library internal measurement

The final library layout is not measured by this code. Therefore a line can fit the estimated budget but still clip when the library lays it out or when focus animation scales it.

### User-visible result

- Last words disappear
- The right side of a long line is cut off
- Active line becomes wider than the screen during animation
- RTL lines can clip on the opposite edge
- A line appears to jump when it becomes active

### Correct fix

Do not rely on a guessed `112.dp` budget as the primary guarantee. Use one of these strategies:

1. **Prefer wrapping inside a custom renderer where the exact available width is known.**
2. If keeping `KaraokeLyricsView`, give it a measured width using `BoxWithConstraints`, account for actual content padding, and remove focused horizontal scaling. Use scale only for alpha/color/vertical translation, or scale around the center with enough explicit inset.
3. Use `TextMeasurer` with the actual resolved `TextStyle`, font resolver, letter spacing, and separator strings. Measure complete candidate rows, not a sum of individual syllables.
4. Add `softWrap = true` and explicit safe horizontal padding in the renderer/library component if supported.
5. Add a final width guard: if a line still exceeds available width, reduce its font size for that line rather than allowing clipping.

### Recommended design

Use a custom `KaraokeLine` composable that:

- Measures the full line using `TextLayoutResult`
- Keeps each word/syllable as an `AnnotatedString` range
- Uses a single paragraph layout
- Draws active progress with a shader/clip rectangle
- Wraps at word boundaries
- Preserves the timing range of each syllable

This is more reliable than converting to a third-party list and trying to predict its internal padding.

---

## LYR-02 — Pre-splitting creates fake lyric lines and can break timing

**Severity: P0 / synchronization and interaction bug**

### Code

```text
ModernLyricsView.kt:369-379
ModernLyricsView.kt:412-507
```

The code does:

```kotlin
lines.flatMap { line ->
    line.splitKaraokeToFit(wrapBudgetPx) { ... }
}
```

The resulting display lines are then converted into `SyncedLyrics`.

### Why this is dangerous

A visual wrap is not necessarily a new lyric event. If a long lyric line is split into two display items, the second visual row may retain timing that was intended for the original full line. This can cause:

- Both visual rows to become active together
- The second row to be auto-centered as if it were a new lyric
- A click on a wrapped row to seek to the wrong place
- Highlight transitions to restart or appear duplicated
- Incorrect progress when syllables span the wrap boundary
- Scroll jumps because the library sees extra timed rows

### Correct fix

Separate **visual layout lines** from **semantic timed lyric lines**.

Use a model similar to:

```kotlin
data class RenderedLyricLine(
    val sourceLineIndex: Int,
    val visualPartIndex: Int,
    val parts: List<RenderedSyllable>,
    val startMs: Long,
    val endMs: Long,
    val seekTargetMs: Long,
    val isContinuation: Boolean,
)
```

All wrapped parts should:

- Belong to one source line
- Share one line timing range
- Have one active-line identity
- Scroll as one group
- Seek to the source line start

If the third-party library cannot represent that, replace it for modern karaoke rendering or render a custom grouped item.

---

## LYR-03 — Final line and bottom controls can overlap

**Severity: P0 / common on short screens**

### Code

```text
ModernLyricsView.kt:151-163
ModernLyricsView.kt:306-318
ModernLyricsView.kt:391-408
PlayerHost.kt:1853-1897
```

The modern panel uses a weighted content area and fixed bottom controls. The karaoke library receives:

```kotlin
offset = 84.dp
```

The content also has:

```kotlin
.padding(horizontal = 12.dp)
```

### Why this is risky

The actual available height is affected by:

- Header and tab controls in `PlayerHost`
- Top system inset
- Bottom system inset
- Fullscreen mode
- Lyrics controls
- Seekbar height
- Navigation mode
- Font scale
- Transliteration/phonetic lines

A fixed `84.dp` offset does not guarantee that the last item can scroll above the bottom controls.

### User-visible result

- Last lyric line hidden beneath seekbar/playback controls
- Bottom line cannot be fully centered
- First line begins behind the header
- Fullscreen and non-fullscreen modes have different clipping

### Correct fix

Pass an explicit content inset from the parent:

```kotlin
lyricsContentPadding = PaddingValues(
    top = headerHeight + 24.dp,
    bottom = controlsHeight + navigationBars + 32.dp,
    start = horizontalSafePadding,
    end = horizontalSafePadding,
)
```

Use:

```kotlin
WindowInsets.safeDrawing
WindowInsets.navigationBars
WindowInsets.ime
```

Measure actual header/control heights with `onSizeChanged` or `LayoutCoordinates` rather than hardcoding `84.dp`.

Add a spacer item at the end of the lyric list whose height equals the bottom inset. Test that the last lyric can be positioned at the visual center.

---

## LYR-04 — Smooth lyric position can drift from audio

**Severity: P0 / timing quality**

### Code

```text
ModernLyricsView.kt:115-149
LyricsView.kt:135-168
```

The code advances position using:

```kotlin
smoothedPositionMs + dt
```

and corrects drift by only 15% per frame:

```kotlin
nextPos += (drift * 0.15f).toLong()
```

### Problems

1. It uses wall-clock elapsed time, not the actual renderer/audio position.
2. `coerceAtLeast(smoothedPositionMs)` prevents backward movement during a normal correction. This can delay correction after a seek or clock discrepancy.
3. Playback speed is not included in the manual `dt` increment.
4. Buffering/underrun can advance lyrics while audio is not advancing.
5. Crossfade or player handoff can make the position source and visual track differ.
6. The legacy effect is keyed on `state.isPlaying`, not `track`, so its frame clock can survive a track switch longer than intended.

### Correct fix

Use a position sample containing:

```kotlin
data class PlaybackClock(
    val positionMs: Long,
    val sampledAtRealtimeMs: Long,
    val speed: Float,
    val isPlaying: Boolean,
    val isBuffering: Boolean,
    val trackKey: String,
)
```

At each frame calculate:

```text
predicted = sampledPosition + (now - sampledAt) * speed
```

Do not advance while buffering or when the player is not rendering. On every seek, track change, pause, resume, buffering transition, speed change, and player handoff, reset the clock.

For best accuracy, expose a player clock from `MusicPlayer` backed by Media3's current position and playback parameters, rather than duplicating timing logic in two composables.

---

## LYR-05 — Legacy word renderer can wrap words independently and break visual order

**Severity: P1**

### Code

```text
LyricsView.kt:665-785
```

Each syllable is emitted as a separate `Text` inside a `FlowRow`:

```kotlin
FlowRow { line.syllables.forEach { Text(...) } }
```

### Problems

- A word can be separated from its visual punctuation or continuation fragment.
- A separator is manually appended to each syllable, affecting measurement.
- A scaled active syllable can collide with adjacent syllables.
- FlowRow may wrap a syllable in a way that differs from the timing model.
- Baselines can look uneven across different scripts/fonts.
- RTL ordering plus per-child layout direction is difficult to guarantee.

### Correct fix

Replace per-syllable child layout with a single measured paragraph per lyric line. Maintain syllable ranges and draw highlight independently from line wrapping.

If retaining FlowRow as an interim fix:

- Group syllables into words before layout.
- Keep `appendToPrevious` fragments in the same child.
- Use a measured separator token rather than adding literal spaces blindly.
- Prevent scale from affecting layout by applying a clipped draw-layer highlight instead of child scale.
- Add explicit `maxLines`/overflow behavior only after verifying it never truncates.

---

## LYR-06 — Syllable duration fallback is too short and inconsistent

**Severity: P1**

### Code

```text
ModernLyricsView.kt:432-437
LyricsView.kt:682-686
```

Fallback duration is often:

```kotlin
150L
```

when the provider does not return a duration.

### Problem

A syllable can remain active for the time until the next syllable, but the legacy renderer uses a fixed 150 ms. This can cause:

- Tiny gaps between highlighted words
- Words appearing inactive before the next word begins
- Fast lyrics failing to highlight correctly
- Slow vocal syllables not remaining highlighted long enough

### Correct fix

When duration is absent, calculate:

```text
nextSyllable.timeMs - currentSyllable.timeMs
```

Then clamp to safe bounds only:

```text
minDuration = 40 ms
maxDuration = lineEnd - syllableStart
```

Use provider duration first, next syllable start second, line end third, and only use a small final fallback if all timing is missing.

---

## LYR-07 — Line duration logic can keep wrong lines active

**Severity: P1**

### Code

```text
LyricsView.kt:289-317
ModernLyricsView.kt:412-416
ModernLyricsView.kt:487-507
```

The implementation uses fallback durations around 4.5–5 seconds and a minimum one second for syllable lines.

### Problems

- A line with a large gap before the next line may be active too long or too short.
- A line with late syllables can use a duration that does not include the actual end correctly.
- Overlapping provider timestamps are not normalized.
- Duplicate timestamps may produce unstable active-line choice.

### Correct fix

Normalize all lyrics once in the repository or a dedicated `LyricsTimelineNormalizer`:

1. Sort by line start time.
2. Remove or merge duplicate empty lines.
3. Clamp negative durations.
4. Derive line end from explicit duration, last syllable end, next line start, or track duration.
5. Never let a line end after the next meaningful line start unless overlap is intentional.
6. Keep a stable source index for seeking and accessibility.

Then make both renderers consume the normalized timeline instead of recalculating different durations.

---

## LYR-08 — `isWordSynced` and actual syllable availability can disagree

**Severity: P1**

### Code

```text
ModernLyricsView.kt:207-208
```

The modern UI decides:

```kotlin
val isWordSynced = targetState.isWordSynced ||
    targetState.lines.any { it.hasSyllables }
```

### Problem

A provider can mark a result as word-synced while returning sparse or malformed syllables, or return syllables only for some lines. The entire view then selects word-sync styling and splitting behavior.

### Correct fix

Calculate a quality score:

```text
word-synced if meaningful lines with valid syllables >= threshold
```

Validate:

- Syllables are within line/track bounds
- Times are monotonic after normalization
- Text coverage is sufficient
- Empty syllables are removed
- At least a defined percentage of vocal lines have timing

Fall back per line to line-sync rendering where syllables are missing.

---

## LYR-09 — Stale lyrics can be shown during a track transition

**Severity: P1 / confusing UX**

### Code

```text
PlayerHost.kt:339-355
PlayerHost.kt:378-417
```

The implementation intentionally keeps previous lyrics visible while new lyrics load. This reduces flashing, but it creates a stale-content window.

### User-visible result

The new song artwork/title may appear while lyrics from the previous song are still visible for a short time.

### Correct fix

Keep the old lyrics only if the header also indicates the old track, or show a transition state:

- Fade old lyrics quickly
- Show new track title and a lightweight skeleton
- Do not allow old lines to be clicked or auto-scrolled
- Associate every `LyricsUiState` with a `trackKey`
- Reject results whose key is not the currently displayed track

Add:

```kotlin
val trackKey: String
```

to `LyricsUiState.Success` and verify it before publishing.

---

## LYR-10 — Loading/error state loses useful error information

**Severity: P2**

### Code

```text
ModernLyricsView.kt:189-194
LyricsView.kt:211-216
```

Both `Empty` and `Error` render the same empty/no-lyrics view. The error message is not surfaced to the user.

### Correct fix

Render separate states:

- No lyrics found: “No synced lyrics found”
- Network failure: “Couldn’t connect” + retry
- Provider failure: “Lyrics source unavailable” + switch source/retry
- Partial result: show available line-sync lyrics with a subtle upgrade indicator

Do not expose raw exception text directly; log it for diagnostics and show a readable message.

---

## LYR-11 — Plain lyric text has limited safe-area behavior

**Severity: P2**

### Code

```text
ModernLyricsView.kt:517-552
LyricsView.kt:818-853
```

Plain lyrics use fixed padding:

```kotlin
.padding(top = 24.dp, bottom = 90.dp, start = 16.dp, end = 16.dp)
```

### Problems

- `90.dp` may be insufficient under the player controls.
- No explicit navigation-bar inset is included.
- Very large font-scale settings may cause uncomfortable wrapping.
- RTL text uses `TextAlign.Start`, but mixed-script text may need paragraph direction handling.

### Correct fix

Use `WindowInsets.safeDrawing` and the measured controls height. Use `BasicText`/`Text` with explicit paragraph direction and selectable/copyable lyrics if desired.

---

## LYR-12 — Transliteration and phonetic content can cause vertical overflow

**Severity: P1 for long/translated songs**

### Code

```text
ModernLyricsView.kt:391-396
ModernLyricsView.kt:450-463
ModernLyricsView.kt:471-475
LyricsView.kt:642-660
LyricsView.kt:787-805
```

Modern mode asks the library to show both:

```kotlin
showTranslation = true
showPhonetic = true
```

But line sizing and safe vertical padding are based mainly on the primary lyric row. Long transliteration or accompaniment rows can increase item height and make the visible focus region unstable.

### Correct fix

- Include translation/phonetic height in measurement.
- Apply a maximum two-line or three-line secondary text policy.
- Use smaller alpha and font size for secondary text.
- Ensure the focused item has enough vertical space before and after it.
- Hide phonetic text by default on small screens, with a user option.

---

# 3. Player visual bugs and risks

## PLY-01 — Nested weighted layout can starve lyrics or controls

**Severity: P0 on short screens**

### Code

```text
ModernLyricsView.kt:151-163
ModernLyricsView.kt:304-318
PlayerHost.kt:1853-1860
PlayerHost.kt:1934-1944
```

`PlayerHost` uses a weighted tab content area, while the lyrics panel uses another weighted content area and fixed bottom controls. The header/tab row, system bars, fullscreen controls, lyrics controls, seekbar, and navigation insets all compete for height.

### User-visible result

- Lyrics area becomes too short
- Seekbar overlaps content
- Controls move or disappear on short devices
- Artwork gets squeezed or clipped in Now Playing
- Landscape layout looks unbalanced

### Correct fix

Create one explicit player layout model:

```text
Top bar measured height
Tab/header measured height
Content available height
Bottom controls measured height
System safe insets
```

Use `WindowInsets.safeDrawing` and one `Scaffold`/custom layout instead of nested, independent weights. Give each tab the same measured content bounds.

---

## PLY-02 — Artwork size calculation can be wrong in constrained layouts

**Severity: P1**

### Code

```text
PlayerHost.kt:1940-1955
```

The artwork size is based on:

```kotlin
(minOf(maxWidth, maxHeight) - 6.dp)
    .coerceAtLeast(0.dp)
    .coerceAtMost(370.dp)
```

### Problems

- `maxHeight` is the height available to the `BoxWithConstraints`, not necessarily the final usable artwork area after all controls.
- The glow is `artworkSize + 28.dp`, which can exceed the parent and be clipped.
- On short devices, artwork can become too small while controls still require fixed space.
- On tablets, the 370 dp cap may leave too much unused space.

### Correct fix

Use aspect-ratio constraints and a layout policy:

- Small portrait: reserve a minimum artwork size and reduce glow.
- Large portrait: cap artwork relative to width.
- Landscape/tablet: use a two-column layout with artwork on the left and controls/queue on the right.
- Measure the controls first, then allocate remaining height to artwork.

Avoid letting decorative glow determine the content bounds.

---

## PLY-03 — Fixed padding and control sizes are not fully inset-aware

**Severity: P1**

Player and lyrics controls use fixed padding such as:

```kotlin
.padding(bottom = 16.dp, top = 6.dp)
.padding(horizontal = 16.dp)
```

The full player also manages system bars separately through `PlayerHost` and window inset code.

### Problems

- Gesture navigation can cover bottom controls.
- Three-button navigation and gesture navigation have different effective safe areas.
- Cutouts and landscape insets can cover top controls.
- Keyboard/IME or external device layouts can change available height.

### Correct fix

Apply `WindowInsets.safeDrawing` at the top-level player container, then avoid applying competing bottom padding in child controls. Use one source of truth for insets.

---

## PLY-04 — Animated scale can clip artwork, lyric text, and buttons

**Severity: P1**

Several elements use `graphicsLayer` scaling and translation:

```text
PlayerHost.kt:1955-1960
PlayerHost.kt:1985 onward
ModernLyricsView.kt:698-781
LyricsView.kt:710-781
```

### Problem

Graphics-layer scale changes the visual bounds without changing layout bounds. A scaled active word or artwork glow can be clipped by a parent with finite bounds.

### Correct fix

- Add explicit clipping policy: do not clip active lyric rows, or add enough content padding.
- Use `transformOrigin = TransformOrigin.Center` for predictable growth.
- Prefer color/alpha/gradient progress over scale for long words.
- Use a small scale range, roughly 1.02–1.05, for dense long lines.
- Ensure parent containers do not unexpectedly clip transformed children.

---

## PLY-05 — Seek/progress updates can cause excessive recomposition

**Severity: P1 performance and visual smoothness**

The player exposes a high-frequency progress stream and lyrics create frame-level state updates. The code tries to separate chrome state from progress state, which is good, but large composable subtrees may still recompose if they read broad `MusicPlayerState` objects.

### Risk

- Dropped frames during word highlighting
- Stuttering animation on lower-end phones
- Delayed seekbar response
- Scroll animation fighting recomposition

### Correct fix

- Keep progress consumers isolated from chrome consumers.
- Pass only primitive fields or narrow state objects to each component.
- Use `derivedStateOf` for active syllable/line index.
- Avoid allocating lists, strings, styles, and animation specs per frame.
- Use stable immutable lyric models.
- Profile with Compose recomposition tracing and Macrobenchmark.

---

## PLY-06 — Player tab transitions can briefly preserve old content state

**Severity: P2**

### Code

```text
PlayerHost.kt:1853-1870
```

The player uses `AnimatedContent` for `currentTab`. The content is animated while the parent may also change fullscreen state and current track state.

### Risk

- Old lyrics remain visible during a tab transition
- The wrong tab receives focus/accessibility focus
- Scroll position is preserved or reset unexpectedly
- A player control is clickable during exit animation

### Correct fix

Use stable keys based on:

```text
currentTab + trackKey + lyricsFullscreen
```

Disable interaction during the transition or use `AnimatedContent`'s target/initial state carefully. Ensure the lyrics scroll state is remembered by track, not only by composable lifetime.

---

## PLY-07 — Long title/artist metadata can create header collisions

### Code

```text
PlayerHost.kt:1819-1828
```

The subtitle is restricted to one line with ellipsis:

```kotlin
maxLines = 1
overflow = TextOverflow.Ellipsis
```

This avoids overflow but can reduce context, especially in lyrics mode where the title and artist are important.

### Improvement

Use a two-line responsive header:

- Title: one line
- Artist: one line
- Use marquee only when focused and only after a delay
- Preserve content description with the full title
- Keep menu button in a fixed-width slot

---

# 4. Architecture changes recommended before polishing

## 4.1 Create a single canonical lyric timeline

Add a dedicated file such as:

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsTimelineNormalizer.kt
```

Responsibilities:

- Sort lines and syllables
- Normalize timestamps
- Infer missing durations
- Remove invalid/empty timing entries
- Preserve source indices
- Calculate line start/end
- Validate word-sync quality
- Handle overlaps and duplicate timestamps
- Produce render-ready immutable data

Both modern and legacy UI should consume this output.

## 4.2 Create a single playback clock

Add a clock in the playback layer, not separately in both UI files. It should account for:

- Current position
- Playback speed
- Paused state
- Buffering state
- Track key
- Seek generation
- Player handoff/crossfade

Lyrics and seekbar should use the same clock.

## 4.3 Replace guessed library padding with explicit layout ownership

The app should own:

- Horizontal content width
- Top inset
- Bottom inset
- Active-line center position
- Item spacing
- Focus animation bounds

A third-party library can still provide text-progress drawing, but it should not own an unknown outer layout if clipping is a priority.

## 4.4 Prefer grouped visual lines over fake semantic lines

A wrapped lyric line should remain one semantic item. Visual parts can be multiple rows inside that item.

This fixes:

- Wrong active-line changes
- Wrong seek targets
- Duplicate timing
- Scroll jumps
- Line highlight flicker

---

# 5. Best visual design improvements

These ideas can make the result feel more polished than a basic Spotify/YouTube Music clone while remaining readable and performant.

## 5.1 Use a focused center lane

Keep the active line near the visual center, with:

- Active line: full white/accent color
- Previous lines: medium alpha
- Future lines: lower alpha
- Nearby lines: slightly larger than distant lines

Avoid making every word scale independently; it produces collisions on long lines.

## 5.2 Use progressive focus instead of aggressive scale

Recommended focus treatment:

```text
Active line: 1.00 scale, alpha 1.00
Previous/next line: 0.96 scale, alpha 0.70
Far lines: 0.92 scale, alpha 0.35
```

For word sync:

- Use a left-to-right active fill gradient
- Use a soft glow behind the active syllable
- Use minimal 1.02–1.04 scale
- Do not scale the whole long line beyond its measured bounds

## 5.3 Add a progress fill rather than per-word popping

For a karaoke line, render the full text in a muted color and overlay an accent-colored clipped copy according to current syllable progress. This gives a continuous fill similar to premium music apps and avoids thousands of animated child composables.

## 5.4 Use adaptive type sizes

Calculate type size based on:

- Available width
- Number of words
- Average glyph width
- Script/font
- Whether translation/phonetic text is shown
- Device width class

Do not use one fixed 22/28 sp style for every long line. A safe range might be:

```text
Large screens: 28–34 sp
Normal phones: 22–28 sp
Very long line: 18–24 sp
Secondary translation: 14–18 sp
```

Use a minimum readable size and wrap before reducing too far.

## 5.5 Add a lyrics display preference

Useful options:

- Word sync on/off
- Translation on/off
- Transliteration on/off
- Compact/comfortable text size
- Reduce motion
- Keep screen awake while lyrics are open
- Tap line to seek on/off
- Center active line on/off

## 5.6 Improve manual scrolling behavior

Recommended behavior:

1. User scroll immediately pauses auto-scroll.
2. Show a small “Jump to current line” button.
3. Auto-scroll resumes after a longer quiet period, such as 4–6 seconds.
4. Tapping the button jumps to the active line and hides itself.
5. Do not fight user scrolling during fast flicks.

The current implementation uses about 2.2 seconds before auto-scroll resumes in legacy mode. This may feel too aggressive for users reading lyrics.

## 5.7 Add a subtle sync status indicator

Use a compact, non-distracting indicator:

```text
Word synced · Apple Music
Line synced · LRCLIB
Plain lyrics
```

Make it accessible through a settings/info action rather than permanently taking vertical space on small screens.

## 5.8 Handle missing word timing gracefully

If only some lines have syllables:

- Word-highlight lines with karaoke fill
- Render missing lines with line-level highlight
- Do not downgrade the entire screen
- Avoid visibly switching typography between lines

## 5.9 Better accessibility

Add:

- Content descriptions for player buttons
- TalkBack announcement for current lyric line
- Adjustable text size
- Reduce motion support
- Minimum contrast validation
- Tap targets of at least 48 dp
- Correct RTL semantics
- Full lyric text in accessibility nodes, not only individual syllable fragments

## 5.10 Landscape and tablet layout

For wide screens:

```text
Left: artwork / player controls
Right: lyrics or queue
```

For portrait:

```text
Header
Lyrics center lane
Seekbar and controls
```

Do not simply stretch the phone layout to tablet width; it causes long lines to become visually sparse and makes controls harder to reach.

---

# 6. Testing plan

## 6.1 Device matrix

Test at minimum:

- 320 dp width phone
- 360–411 dp phone
- Large phone
- Tablet portrait
- Tablet landscape
- Short-height landscape phone
- Gesture navigation
- Three-button navigation
- Font scale 1.0, 1.3, 1.5, and 2.0
- 60 Hz and 120 Hz
- Light and dark themes if both are supported
- RTL locale

## 6.2 Lyric content matrix

Test:

- Very long English line
- Long Arabic/Urdu/Hebrew line
- CJK lyrics with no spaces
- Lyrics containing punctuation
- Apple Music `part` continuation fragments
- Background vocals
- Transliteration
- Translation
- Missing syllable durations
- Duplicate timestamps
- Overlapping lines
- Empty musical-note lines
- Final lyric line near track end
- Instrumental track
- Plain lyrics only

## 6.3 Interaction tests

Verify:

- Tap any line seeks correctly
- Wrapped line seeks to the source line, not a fake sub-line
- Drag seekbar updates lyric highlight immediately
- Pause freezes word highlight
- Resume continues from actual audio position
- Buffering does not advance lyrics
- Playback speed changes timing correctly
- Next/previous track never shows stale lyrics
- Manual scroll does not fight auto-scroll
- Jump-to-current-line works
- Fullscreen does not hide first/last lines

## 6.4 Automated tests to add

Recommended new tests:

```text
LyricsTimelineNormalizerTest.kt
LyricsWrapLayoutTest.kt
LyricsPlaybackClockTest.kt
LyricsTrackIdentityTest.kt
KaraokeSyllableTimingTest.kt
```

Test invariants:

- Every visual part maps to one source line
- No rendered line exceeds available width
- Last line can scroll above bottom controls
- Syllables are monotonic
- Missing duration uses next timestamp correctly
- Seek target always equals source line start
- Old-track results are rejected
- Clock does not advance while buffering

## 6.5 Performance tests

Use Compose recomposition tracing and Macrobenchmark to measure:

- Frame time while word highlighting is active
- Recomposition count per progress tick
- Scroll jank during auto-scroll
- Memory allocations for a long song
- CPU usage with 100+ lyric lines and 20+ syllables per line

---

# 7. Recommended implementation order

## Phase 1 — Stop clipping and wrong scrolling

1. Add explicit safe top/bottom content insets.
2. Make the final lyric line scroll above the controls.
3. Remove or reduce horizontal focused-line scaling.
4. Replace guessed width budget with actual measured layout width.
5. Add long-line and small-device screenshot tests.

## Phase 2 — Fix timing correctness

1. Add `LyricsTimelineNormalizer`.
2. Preserve source-line identity through wrapping.
3. Fix missing syllable duration calculation.
4. Create one shared playback clock.
5. Pause visual time during buffering.
6. Add track-key validation to lyric results.

## Phase 3 — Improve rendering quality

1. Replace per-syllable FlowRow children with one measured paragraph per line.
2. Implement clipped active fill/highlight.
3. Add adaptive font size and safe wrapping.
4. Improve translation/phonetic layout.
5. Add manual-scroll pause and jump-to-current control.

## Phase 4 — Improve player layout

1. Consolidate system insets at the player root.
2. Replace nested weights with measured player regions.
3. Add tablet and landscape two-column layout.
4. Make artwork allocation depend on remaining height.
5. Profile progress-driven recompositions.

## Phase 5 — Premium polish

1. Progressive focus lane.
2. Smooth active fill gradient.
3. Reduced-motion mode.
4. Accessibility announcements.
5. User-controlled translation, phonetic text, text size, and animation.
6. Sync diagnostics for provider/timing quality.

---

# 8. File-by-file change map

## Highest priority

```text
app/src/main/java/com/lastwave/app/ui/player/ModernLyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/LyricsView.kt
app/src/main/java/com/lastwave/app/ui/player/PlayerHost.kt
```

These govern the visual bugs users currently see.

## Add or refactor

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsTimelineNormalizer.kt
app/src/main/java/com/lastwave/app/playback/PlaybackClock.kt
```

These would centralize timing and remove duplicated UI calculations.

## Data validation and provider correctness

```text
app/src/main/java/com/lastwave/app/data/lyrics/LyricsRepository.kt
app/src/main/java/com/lastwave/app/data/lyrics/AppleMusicLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/LyricsPlusApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/BetterLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/KugouLyricsApi.kt
app/src/main/java/com/lastwave/app/data/lyrics/LrclibLyricsApi.kt
```

## Player engine and source clock

```text
app/src/main/java/com/lastwave/app/playback/MusicPlayer.kt
app/src/main/java/com/lastwave/app/playback/MusicPlaybackService.kt
```

## Player controls and surfaces

```text
app/src/main/java/com/lastwave/app/ui/player/WavySeekBar.kt
app/src/main/java/com/lastwave/app/ui/player/SignalPathDialog.kt
app/src/main/java/com/lastwave/app/ui/player/PlayerCastMenuRow.kt
app/src/main/java/com/lastwave/app/ui/common/TrackMiniTraySheet.kt
app/src/main/java/com/lastwave/app/ui/shell/MainShell.kt
```

---

# Final recommendation

The best path is not simply increasing padding or reducing font size. The core fix is to separate:

1. **Semantic lyrics timing** — the actual source line and syllable timeline.
2. **Visual layout** — how that line wraps and appears on the current screen.
3. **Playback clock** — the authoritative current audio position.
4. **Player geometry** — measured safe space for header, lyrics, artwork, controls, and system insets.

Once those four concerns are separated, long lyrics can wrap without losing timing, the last line can never 
