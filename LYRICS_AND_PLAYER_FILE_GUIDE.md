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
