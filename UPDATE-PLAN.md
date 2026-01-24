# Update Plan: whisper.cpp & VoiceInk

**Created:** 2026-01-24
**Purpose:** Update both forks from upstream while preserving custom modifications

---

## Overview

### Repositories
| Repo | Location | Branch Strategy |
|------|----------|-----------------|
| whisper.cpp | `~/Tools/whisper.cpp` | `master` (upstream) + `james` (custom) |
| VoiceInk | `~/Tools/VoiceInk` | `main` (upstream) + `custom-build` (custom) |

### Current Status
- **whisper.cpp:** 283 commits behind upstream (last synced: Dec 10, 2025)
- **VoiceInk:** 218 commits behind upstream (last synced: Oct 26, 2025)

### Dependency Chain
```
~/.zshrc alias "whispercpp"
    → ~/Tools/whisper.cpp/whispercpp-convert.sh
        → ~/Tools/whisper.cpp/build/bin/whisper-cli

VoiceInk.app
    → ~/Tools/whisper.cpp/build-apple/whisper.xcframework
```

---

## Custom Modifications to Preserve

### whisper.cpp (james branch)
| File | Purpose |
|------|---------|
| `whispercpp-convert.sh` | Wrapper script: auto-converts formats, defaults to large-v3-turbo, 12 threads, txt output |
| `CLAUDE.md` | Documentation for Claude Code |
| `WHISPER-USAGE-GUIDE.md` | Usage documentation |

### VoiceInk (custom-build branch)
| File | Modification |
|------|--------------|
| `VoiceInk/Models/LicenseViewModel.swift` | Forces `licenseState = .licensed`, `canUseApp = true` |
| `VoiceInk/VoiceInk.swift` | Disables Sparkle auto-updates |

---

## Model Configuration

**Current model:** `ggml-large-v3-turbo.bin` (1.5 GB)

**Performance on M4 Pro:**
- JFK sample (11 sec audio): 1.2 sec transcription
- Uses Core ML + Metal acceleration
- 12 threads configured

**Recommendation:** Keep `large-v3-turbo` - it's optimal for your setup. Pre-converted, CoreML-ready, and excellent performance.

---

## Execution Plan

### Phase 1: Documentation Updates (CLI)
Tasks Claude can do automatically.

#### Task 1.1: Update whisper.cpp CLAUDE.md
Add:
- `whispercpp` alias documentation
- VoiceInk xcframework dependency info
- Full rebuild steps after upstream sync

#### Task 1.2: Update VoiceInk CLAUDE.md
Add:
- Note that whisper.xcframework must be rebuilt first
- Xcode signing setup for local builds
- Exact files modified for license/update bypass

---

### Phase 2: Update whisper.cpp (CLI)
All tasks can be done via command line.

#### Task 2.1: Update master from upstream
```bash
cd ~/Tools/whisper.cpp
git checkout master
git fetch upstream
git merge upstream/master
git push origin master
```

#### Task 2.2: Rebase james branch
```bash
git checkout james
git rebase master
# Resolve conflicts if any
git push origin james --force-with-lease
```

#### Task 2.3: Verify custom files
Check that these files still exist and are correct:
- `whispercpp-convert.sh`
- `CLAUDE.md`
- `WHISPER-USAGE-GUIDE.md`

#### Task 2.4: Rebuild whisper-cli
```bash
cmake -B build -DGGML_METAL=1
cmake --build build --config Release
```

#### Task 2.5: Test whispercpp alias
```bash
whispercpp -f ~/Tools/whisper.cpp/samples/jfk.wav
```
Expected: Successful transcription of JFK audio.

#### Task 2.6: Rebuild xcframework (for VoiceInk)
```bash
./build-xcframework.sh
```
**Note:** Takes 10-20 minutes. Builds for iOS, macOS, visionOS, tvOS.

---

### Phase 3: Update VoiceInk (CLI + Manual)
Most tasks via CLI, some require manual Xcode interaction.

#### Task 3.1: Handle uncommitted changes
```bash
cd ~/Tools/VoiceInk
git stash
```

#### Task 3.2: Update main from upstream
```bash
git checkout main
git fetch upstream
git merge --ff-only upstream/main
```

#### Task 3.3: Rebase custom-build
```bash
git checkout custom-build
git rebase main
# Resolve conflicts if any - likely in LicenseViewModel.swift
```

#### Task 3.4: Verify license bypass
Check `VoiceInk/Models/LicenseViewModel.swift` contains:
```swift
@Published private(set) var licenseState: LicenseState = .licensed
// In loadLicenseState(): licenseState = .licensed
// canUseApp always returns true
```

#### Task 3.5: Verify auto-update disabled
Check `VoiceInk/VoiceInk.swift` bypasses Sparkle framework.

#### Task 3.6: Clean and rebuild
```bash
xcodebuild clean -project VoiceInk.xcodeproj -scheme VoiceInk
xcodebuild -project VoiceInk.xcodeproj -scheme VoiceInk -configuration Release build
```

#### Task 3.7: [MANUAL] Xcode signing setup
**You must do this if CLI build fails on signing:**
1. Open `VoiceInk.xcodeproj` in Xcode
2. Select VoiceInk target → Signing & Capabilities
3. Set Team to "None" or your personal team
4. Set Signing Certificate to "Sign to Run Locally"
5. Build with Cmd+B

#### Task 3.8: [MANUAL] Test the app
1. Launch VoiceInk from Xcode or `~/Library/Developer/Xcode/DerivedData/`
2. Verify it opens without license prompts
3. Test recording and transcription
4. Verify no update prompts appear

---

## Potential Issues & Solutions

### Rebase Conflicts
**whisper.cpp:** Unlikely - custom files are separate from upstream code.

**VoiceInk:** Possible in `LicenseViewModel.swift` if upstream changed it.
- Solution: Manually re-apply the license bypass after resolving conflicts.

### xcframework Build Failure
If `build-xcframework.sh` fails:
- Check Xcode version: `xcodebuild -version`
- Ensure Command Line Tools: `xcode-select --install`
- Clean previous builds: `rm -rf build-*`

### VoiceInk Signing Issues
If xcodebuild fails with signing errors:
- Must configure signing in Xcode GUI (Task 3.7)
- One-time setup, persists across builds

---

## Rollback Plan

If something goes wrong:

### whisper.cpp
```bash
git checkout james
git reset --hard origin/james  # Before push
# or
git reflog  # Find previous commit
git reset --hard <commit>
```

### VoiceInk
```bash
git checkout custom-build
git reset --hard origin/custom-build
git stash pop  # Restore stashed changes
```

---

## Post-Update Verification Checklist

- [ ] `whispercpp -f samples/jfk.wav` works
- [ ] whisper.cpp james branch is clean with custom commits on top
- [ ] `build-apple/whisper.xcframework` exists and is recent
- [ ] VoiceInk launches without license prompts
- [ ] VoiceInk transcription works
- [ ] No auto-update prompts in VoiceInk
- [ ] Both CLAUDE.md files are updated with complete documentation
