# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About VoiceInk

VoiceInk is a native macOS application (macOS 14.0+) that transcribes voice to text using local AI models. It's built with Swift and SwiftUI, focusing on privacy-first offline transcription with 99% accuracy. This is a custom fork with personal modifications.

## Building and Running

### Prerequisites
- macOS 14.0 or later
- Xcode (latest version)
- whisper.cpp XCFramework (must be built separately)

### Building whisper.cpp Framework
```bash
cd ~/Tools/whisper.cpp
./build-xcframework.sh
```
This creates `build-apple/whisper.xcframework`.

### Building VoiceInk
```bash
# Open project in Xcode
open VoiceInk.xcodeproj

# Or build from command line
xcodebuild -project VoiceInk.xcodeproj -scheme VoiceInk -configuration Debug build
```

**Important**: The whisper.xcframework must be added to "Frameworks, Libraries, and Embedded Content" in project settings.

### Running Tests
```bash
# Run all tests
xcodebuild test -project VoiceInk.xcodeproj -scheme VoiceInk

# Run specific test target
xcodebuild test -project VoiceInk.xcodeproj -scheme VoiceInk -only-testing:VoiceInkTests
```

### Clean Build
```bash
# Clean build folder
xcodebuild clean -project VoiceInk.xcodeproj -scheme VoiceInk

# Or in Xcode: Cmd+Shift+K
```

## Architecture

### Core Components

**WhisperState** (`VoiceInk/Whisper/WhisperState.swift`)
- Central state manager for the entire app
- Manages recording state machine: `.idle` → `.recording` → `.transcribing` → `.enhancing`
- Coordinates between recorder, transcription services, and UI
- Handles model loading, switching, and lifecycle

**TranscriptionService Protocol** (`VoiceInk/Services/TranscriptionService.swift`)
- Unified interface for all transcription providers
- Implementations:
  - `LocalTranscriptionService`: Local whisper.cpp models
  - `CloudTranscriptionService`: Groq, Deepgram, Mistral, etc.
  - `ParakeetTranscriptionService`: Parakeet models via FluidAudio
  - `NativeAppleTranscriptionService`: Apple's native speech recognition

**Recorder** (`VoiceInk/Recorder.swift`)
- Handles audio capture using AVAudioRecorder
- Audio format: 16kHz, mono, 16-bit PCM
- Manages audio device selection and dynamic device switching
- Monitors audio levels and controls media playback

**PowerMode** (`VoiceInk/PowerMode/`)
- Context-aware transcription settings
- Auto-applies configurations based on active app/URL
- `PowerModeSessionManager`: Saves/restores app state when entering/exiting power mode
- `ActiveWindowService` + `BrowserURLService`: Detect current context

### Model System

**TranscriptionModel Protocol** (`VoiceInk/Models/TranscriptionModel.swift`)
- Unified interface for all model types
- Provider types: `.local`, `.parakeet`, `.groq`, `.elevenLabs`, `.deepgram`, `.mistral`, `.gemini`, `.custom`, `.nativeApple`
- Implementations:
  - `LocalModel`: whisper.cpp GGML models
  - `ParakeetModel`: Parakeet v1/v2/v3 models
  - `CloudModel`: Cloud API-based models
  - `CustomCloudModel`: User-defined OpenAI-compatible endpoints
  - `ImportedLocalModel`: User-imported .bin files

**Model Management** (`VoiceInk/Whisper/WhisperState+ModelManagement.swift`)
- Downloads models from HuggingFace
- Tracks download progress per model
- Auto warm-up feature for downloaded models
- Model storage: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/WhisperModels/`

### UI Structure

**ContentView** (`VoiceInk/Views/ContentView.swift`)
- Main app navigation using sidebar pattern
- View types: Dashboard, Transcribe Audio, History, AI Models, Enhancement, Power Mode, Permissions, Audio Input, Dictionary, Settings, License
- Dynamic sidebar that adapts to feature flags (e.g., PowerMode UI)

**Recorder Views**
- `MiniRecorderView`: Compact floating recorder
- `NotchRecorderView`: Recorder that integrates with MacBook notch
- Controlled by `recorderType` preference ("mini" or "notch")

### Services

**AIEnhancementService** (`VoiceInk/Services/AIEnhancementService.swift`)
- Post-transcription AI processing
- Integrates with AIService for LLM providers
- Screen capture context awareness
- Custom prompt system with templates

**AudioDeviceManager** (`VoiceInk/Services/AudioDeviceManager.swift`)
- Manages audio input device selection
- Monitors device changes and reconnects automatically
- Persists last used device

**DictionaryContextService** (`VoiceInk/Services/DictionaryContextService.swift`)
- Custom word recognition and text replacement
- Feeds custom vocabulary to transcription prompt

### Data Model

**Transcription** (`VoiceInk/Models/Transcription.swift`)
- SwiftData model for transcription history
- Stores audio file URL, text, duration, status
- Managed by `ModelContext` throughout the app

## Key Patterns

### State Machine
Recording follows strict state transitions:
```swift
.idle → .recording → .transcribing → .enhancing → .idle
```
Never transition directly between non-adjacent states.

### Service Selection
Model provider determines which TranscriptionService implementation to use:
```swift
switch model.provider {
case .local: LocalTranscriptionService
case .parakeet: ParakeetTranscriptionService
case .nativeApple: NativeAppleTranscriptionService
default: CloudTranscriptionService
}
```

### Context Loading
WhisperState reuses loaded models when possible to avoid reload overhead. LocalTranscriptionService checks if the requested model is already loaded in WhisperState before creating a new context.

## Package Dependencies

Dependencies are managed via Swift Package Manager:
- **Sparkle**: Auto-updates (2.7.1)
- **KeyboardShortcuts**: Global hotkey management (2.3.0)
- **LaunchAtLogin**: Launch at login functionality (main branch)
- **MediaRemoteAdapter**: Media playback control (master branch)
- **FluidAudio**: Parakeet model support (main branch)
- **Zip**: Model archive extraction (2.1.2)

## File Locations

- **Models Directory**: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/WhisperModels/`
- **Recordings Directory**: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/Recordings/`
- **whisper.xcframework**: Should be at `../whisper.cpp/build-apple/whisper.xcframework` relative to project

## Custom Build Branch

This repository has two branches:

**`main`** - Tracks upstream (https://github.com/Beingpax/VoiceInk)
- Kept in sync with the original repository
- Use this as the base for merging upstream updates

**`custom-build`** - Personal development branch
- Built on top of `main` with custom modifications
- Modifications in commit "Custom build modifications for personal use":
  - **License system disabled**: Always returns `.licensed` state, no trial period
  - **Auto-updates disabled**: Prevents custom build from being overwritten
- Use this branch for development and running the app

### Custom Modifications (Exact Changes)

**`VoiceInk/Models/LicenseViewModel.swift`:**
```swift
// Change 1: Initialize as licensed
@Published private(set) var licenseState: LicenseState = .licensed

// Change 2: In loadLicenseState(), always set licensed
private func loadLicenseState() {
    licenseState = .licensed
    // ... rest of function bypassed
}

// Change 3: canUseApp always returns true
var canUseApp: Bool {
    return true
}
```

**`VoiceInk/VoiceInk.swift`:**
- Sparkle auto-update framework initialization is bypassed/disabled
- CloudKit sync disabled for dictionary (change `cloudKitDatabase: .private(...)` to `.none`)

### Updating Workflow

**Important:** Before updating VoiceInk, you must first rebuild `whisper.xcframework` from the updated whisper.cpp repo.

```bash
# Step 0: Rebuild whisper.cpp xcframework first
cd ~/Tools/whisper.cpp
./build-xcframework.sh

# Step 1: Handle any uncommitted changes
cd ~/Tools/VoiceInk
git stash

# Step 2: Update main from upstream
git checkout main
git fetch upstream
git merge --ff-only upstream/main

# Step 3: Rebase custom-build on updated main
git checkout custom-build
git rebase main
# If conflicts occur in LicenseViewModel.swift, re-apply the license bypass manually

# Step 4: Restore stashed changes if any
git stash pop
```

### Verifying Modifications After Rebase

After rebasing, verify the custom modifications are still in place:

1. **License bypass:** Check `LicenseViewModel.swift` - `licenseState` should initialize to `.licensed`
2. **Auto-update disabled:** Check `VoiceInk.swift` - Sparkle initialization should be bypassed

### Building Without Apple Developer Account

This project can be built for local use without an Apple Developer account:

**Prerequisites (one-time setup):**

1. **Remove capabilities requiring paid account (in Xcode):**
   - Open `VoiceInk.xcodeproj` in Xcode
   - Select VoiceInk target → Signing & Capabilities tab
   - Delete **iCloud** capability (click trash icon)
   - Delete **Push Notifications** capability (click trash icon)
   - Set Team to your personal team

2. **Create VoiceInk scheme if missing:**
   - If only SPM package schemes appear (KeyboardShortcuts, SelectedTextKit), the VoiceInk scheme is missing
   - In Xcode: Product → Scheme → New Scheme → select VoiceInk target

**Build Release version:**
```bash
cd ~/Tools/VoiceInk
xcodebuild -project VoiceInk.xcodeproj -scheme VoiceInk -configuration Release -arch arm64 -allowProvisioningUpdates build
```

**Install to Applications:**
```bash
rm -rf /Applications/VoiceInk.app
cp -R ~/Library/Developer/Xcode/DerivedData/VoiceInk-*/Build/Products/Release/VoiceInk.app /Applications/
codesign --force --deep --sign "Apple Development: james.milton@me.com (QS8QDV4C85)" /Applications/VoiceInk.app
```

**Note:** The codesign step is required. Without it the app gets ad-hoc signed and macOS blocks it after every restart.

**Clean up after building (removes extra Spotlight entries):**
```bash
rm -rf ~/Library/Developer/Xcode/DerivedData/VoiceInk-*
```

### Troubleshooting

**"whisper.xcframework not found" error:**
- Ensure whisper.cpp is at `~/Tools/whisper.cpp` (sibling to VoiceInk)
- Rebuild xcframework: `cd ~/Tools/whisper.cpp && ./build-xcframework.sh`

**Signing/provisioning profile errors:**
- Remove iCloud and Push Notifications capabilities in Xcode (see Prerequisites above)

**App crashes on launch (CloudKit error):**
- Verify `VoiceInk.swift` has `cloudKitDatabase: .none` for the dictionary config (not `.private(...)`)

**No VoiceInk scheme available:**
- Create scheme: Product → Scheme → New Scheme → select VoiceInk target
- Or check `xcshareddata/xcschemes/VoiceInk.xcscheme` exists

**Multiple VoiceInks appear in Spotlight:**
- Clean up: `rm -rf ~/Library/Developer/Xcode/DerivedData/VoiceInk-*`

**License prompt appears:**
- Verify `LicenseViewModel.swift` modifications are in place after rebase

**App tries to auto-update:**
- Verify `VoiceInk.swift` Sparkle bypass is in place after rebase

**Slow transcription startup:**
- Normal behavior - Whisper models need to load into memory on first use
- Larger models (large-v3-turbo) take longer to initialize
