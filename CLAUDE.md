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

### Updating Workflow

To pull latest upstream updates while preserving custom modifications:

```bash
# Update main from upstream
git checkout main
git fetch upstream
git merge --ff-only upstream/main

# Rebase custom-build on updated main
git checkout custom-build
git rebase main
```

This keeps your custom-build branch with all upstream features plus the license/update modifications.
