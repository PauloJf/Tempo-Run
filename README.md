# 🏃 Tempo Run

> **Tempo Run** is a Flutter running music player that connects to self-hosted music servers (Navidrome or Jellyfin) and automatically adjusts song playback speed to match your running cadence (steps per minute).

[![Flutter Version](https://img.shields.io/badge/Flutter-3.3.0+-blue)](https://flutter.dev)
[![Dart Version](https://img.shields.io/badge/Dart-3.3.0+-blue)](https://dart.dev)
[![Platform](https://img.shields.io/badge/Platform-Android%20%7C%20iOS-green)](https://flutter.dev)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

## 📖 Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Core Concepts](#core-concepts)
- [Configuration Guide](#configuration-guide)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## ✨ Features

### Player Screen
- **Live BPM Display**: Large animated display showing the conversion chain: native BPM → multiplier → playback rate → target BPM
- **Dynamic Album Art**: Album cover with palette-extracted glow that pulses to the beat
- **BPM Control**: Quick adjustment buttons (±1, ±5 BPM) to change your running cadence on the fly
- **Cadence Locking**: Toggle between locked (cadence-matched playback) and unlocked (normal speed) modes
- **Transport Controls**: Seekable progress bar, play/pause, skip-next buttons
- **Tap Tempo**: Set target BPM by tapping to your running rhythm
- **Sensor Cadence**: Real-time accelerometer reading showing your actual running step rate

### Intelligent Queue System
- Fetches 50 random songs and intelligently scores each by BPM fit
- Considers multiplier options (×0.5, ×1.0, ×2.0) to find songs within tolerance
- Auto-categorizes songs into preferred (BPM-matched) and fallback buckets
- Automatically refills when queue drops below 3 songs
- Reloads entire queue when server credentials change

### Settings Hub (4 Pages)
- **Server**: Navidrome/Jellyfin URL, credentials, connection testing, demo library access
- **Queue**: Configure BPM multipliers, tolerance percentage, BPM data requirement, queue preload size (3-50 songs)
- **Display**: Toggle glow pulse animation, Tap Tempo visibility, Sensor Cadence visibility
- **BPM Overrides**: Per-song BPM corrections with persistent storage

### Pace Log (History)
- Complete playback history: song title/artist, target BPM, song BPM, playback rate, BPM multiple used
- Grouped by date (TODAY / YESTERDAY / YYYY-MM-DD)
- Color-coded adjustment percentages: green ≤5%, amber ≤15%, red >15%
- Persisted in SQLite, auto-pruned to 500 most-recent entries
- Share and clear functionality

### Background Playback (Android)
- Native foreground service with persistent notification
- Shows song title, artist, target BPM, and cover art
- Hardware/notification controls fully integrated via MethodChannel

---

## 🛠 Tech Stack

### Core Framework
- **Flutter**: 3.3.0+
- **Dart**: 3.3.0+
- **Target Platforms**: Android 21+ (APK), iOS 11+ (IPA)

### State Management & Data
- **flutter_riverpod** (^2.5.1): Reactive state management with NotifierProviders
- **sqflite** (^2.3.3): Local SQLite database for BPM overrides and pace history
- **sqflite_common_ffi** (^2.3.0): FFI support for desktop testing
- **shared_preferences** (^2.2.3): Lightweight key-value storage for settings
- **flutter_secure_storage** (^9.2.2): Encrypted storage for server credentials

### Audio & Media
- **just_audio** (^0.9.40): Audio playback with tempo stretching via playback rate control
- **audio_session** (^0.1.21): Background audio configuration (iOS AVAudioSession, Android AudioFocus)
- **palette_generator** (^0.3.3): Dynamic color extraction from album artwork

### Networking & API
- **dio** (^5.4.3): HTTP client for Subsonic API communication
- **crypto** (^3.0.3): MD5 hashing for Subsonic authentication

### Sensors & Hardware
- **sensors_plus** (^4.0.2): Accelerometer access for cadence detection

### UI & Assets
- **google_fonts** (^6.2.1): Custom typography
- **cupertino_icons** (^1.0.8): iOS-style icons
- **share_plus** (^10.0.0): Share pace log functionality
- **package_info_plus** (^8.0.0): App version information

---

## 📚 Core Concepts

### BPM Matching Algorithm

The queue intelligently selects songs based on how well their BPM fits your running cadence:

1. **Target Cadence**: You set your running BPM (e.g., 160 steps/minute)
2. **Song Retrieval**: 50 random songs fetched from server
3. **Scoring**: For each song, try multipliers (×0.5, ×1.0, ×2.0):
   - Calculate `abs(songBpm × multiple - targetBpm)`
   - If within tolerance (default ±15%), score it
4. **Bucket Sorting**:
   - Preferred bucket: Songs with BPM data and good fit
   - Fallback bucket: Songs without BPM data or out-of-tolerance
5. **Playback**: Queue pops from preferred bucket first

**Example**:
- Target: 160 BPM
- Song: 82 BPM (±15% tolerance = 69.7–94.3 BPM)
- Match at ×2.0: 82 × 2.0 = 164 BPM ✓ (within range)
- Playback rate: 160 / (82 × 2.0) = 0.976x

### Cadence Detection (Accelerometer)

`CadenceDetector` (`sensors/cadence_detector.dart`):
- Reads device accelerometer (Z-axis acceleration)
- Applies high-pass filter to isolate vertical motion
- Detects peaks (steps) via amplitude thresholding
- Outputs steps per minute (BPM equivalent)
- Updates in real-time on PlayerScreen

### Tempo Stretching

**just_audio** applies time-stretching (changes playback speed without changing pitch):
- Range: 0.5x to 2.0x speed
- Native plugins handle audio quality
- No beat distortion or robotic sound

### BPM Multipliers

Songs don't always have BPM in 1:1 match with your cadence:
- **×0.5**: Slow ballads (e.g., 80 BPM song → 160 BPM at 0.5x)
- **×1.0**: Songs near your cadence (e.g., 160 BPM song → 160 BPM at 1.0x)
- **×2.0**: Fast songs (e.g., 80 BPM song → 160 BPM at 2.0x)

Configurable in Settings → Queue.

### Pace Log

Every song played is logged with:
- Timestamp, song metadata, BPM data
- Target cadence and playback rate applied
- Sensor cadence reading at play time
- Color-coded adjustment % for analysis

Useful for tracking running sessions and musical patterns.

---

## ⚙️ Configuration Guide

### Initial Setup Wizard

On first launch, you'll see the settings wizard. You **must** configure a music server:

1. **Choose Server Type**: Navidrome or Jellyfin
2. **Enter Server URL**: Base URL (e.g., `https://navidrome.example.com`)
3. **Enter Credentials**: Username and password
4. **Test Connection**: The app validates connectivity
5. **Proceed**: You're ready to listen

**Demo Server Option**: Click "Demo Library" to test with a public Navidrome instance.

### Server Settings

**Path**: Settings Hub → Server

| Setting | Description |
|---------|-------------|
| Server Type | Navidrome or Jellyfin |
| Base URL | Server address (e.g., `https://music.example.com`) |
| Username | Subsonic API username |
| Password | Subsonic API password |
| Test Connection | Validates credentials and connectivity |
| Demo Library | Switch to public demo server for testing |

### Queue Settings

**Path**: Settings Hub → Queue

| Setting | Description | Default |
|---------|-------------|---------|
| BPM Multipliers | Enable ×0.5, ×1.0, ×2.0 | All enabled |
| Tolerance % | Acceptable BPM range (±%) | 15% |
| Require BPM Data | Only queue songs with BPM metadata | Off |
| Queue Size | Number of songs to preload | 15 songs |

### Display Settings

**Path**: Settings Hub → Display

| Setting | Description |
|---------|-------------|
| Glow Pulse Animation | Animates album art glow to beat |
| Tap Tempo Button | Show/hide tap tempo control |
| Sensor Cadence Display | Show/hide accelerometer reading |

### BPM Overrides

**Path**: Settings Hub → BPM Overrides

Manually correct the BPM of songs the server provides incorrectly:
1. Tap a song to view/edit its BPM
2. Enter corrected BPM
3. Saved to SQLite and applied globally

---

## 🆘 Troubleshooting

### BPM Not Detected

**Cause**: Server returned songs without BPM metadata
**Fix**:
1. Disable "Require BPM Data" in Queue settings (allows fallback songs)
2. Manually set BPM in BPM Overrides screen
3. Check Navidrome/Jellyfin library has BPM data

### Server Connection Failed

**Check**:
1. Server URL is correct (e.g., `https://navidrome.example.com`, not `https://navidrome.example.com/`)
2. Credentials are correct
3. Server is online and reachable
4. Try "Test Connection" in Server settings
5. Check firewall/VPN if accessing remotely

### Slow Queue Loading

**Reduce queue size** in Settings → Queue (fewer preloaded songs = faster fetch)

**Disable animations** in Settings → Display (Glow Pulse) to improve performance

### Accelerometer Not Working

**Android**:
- Device must have accelerometer hardware

**iOS**:
- Grant motion permission when prompted

---

## 📜 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- **just_audio**: Outstanding audio playback library with tempo stretching
- **Flutter & Dart**: Powerful cross-platform framework
- **Navidrome & Jellyfin**: Self-hosted music server projects
- **Riverpod**: Elegant reactive state management
- All contributors and users providing feedback

---

**Made with ❤️ for runners who love their music**
