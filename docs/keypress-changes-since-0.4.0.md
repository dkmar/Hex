# Keypress Handling Changes Since Version 0.4.0

**Analysis Date:** November 22, 2025
**Current Version:** 0.6.2
**Base Version:** 0.4.0 (January 28, 2025)

## Executive Summary

Between versions 0.4.0 and 0.6.2 (69 commits over ~10 months), keypress handling in Hex underwent significant improvements focused on **reliability**, **macOS Sequoia compatibility**, and **documentation**. The core `HotKeyProcessor` state machine evolved from 321 lines to 522 lines (+62%) with extensive inline documentation, extracted constants, and refined edge-case handling.

---

## 1. Major Architectural Changes

### 1.1 Extracted Timing Constants (v0.5.13 - commit 083513c)

**Before (v0.4.0):**
```swift
public static let doubleTapThreshold: TimeInterval = 0.3
public static let pressAndHoldCancelThreshold: TimeInterval = 1.0
```

**After (v0.5.13+):**
All timing values moved to `HexCoreConstants.swift` with comprehensive documentation:
- `doubleTapWindow` = 0.3s (double-tap detection)
- `modifierOnlyMinimumDuration` = 0.3s (prevents OS shortcut conflicts)
- `pressAndHoldCancelWindow` = 1.0s (accidental key press threshold)
- `defaultMinimumKeyTime` = 0.2s (user-configurable base threshold)

**Impact:** Magic numbers eliminated, rationale documented, easier to maintain.

---

### 1.2 Enhanced Documentation (v0.5.13 - commit 083513c)

**File:** `HotKeyProcessor.swift:10-88`

Added **78 lines** of comprehensive doc comments covering:
- Architecture overview with 3-state machine description
- Double-tap detection algorithm
- Press-and-hold behavior
- Modifier-only hotkey specifics (0.3s threshold, mouse click handling)
- Dirty state & backsliding prevention
- ESC key handling
- Example interaction flows
- Related components

**File:** `docs/hotkey-semantics.md` (578 lines)

Created extensive user-facing documentation with:
- Quick reference tables for modifier-only vs regular hotkeys
- Timeline diagrams showing threshold behavior
- 46 passing test scenarios
- Decision trees and state machines
- Real-world examples (Option+click, Option+A for special characters)

---

### 1.3 RecordingDecisionEngine Extracted

**File:** `HexCore/Sources/HexCore/Logic/RecordingDecision.swift:1-78`

Separated recording decision logic from hotkey processing:

```swift
public struct RecordingDecisionEngine {
    public static let modifierOnlyMinimumDuration: TimeInterval = 0.3

    public enum Decision {
        case discardShortRecording
        case proceedToTranscription
    }

    public static func decide(_ context: Context) -> Decision {
        let effectiveMinimum = includesPrintableKey
            ? context.minimumKeyTime
            : max(context.minimumKeyTime, modifierOnlyMinimumDuration)
        // ...
    }
}
```

**Impact:** Clear separation of concerns, testable decision logic.

---

## 2. Modifier-Only Hotkey Improvements

### 2.1 Mouse Click Handling Refinement

**Location:** `HotKeyProcessor.swift:210-241`

**Behavior Changes:**

| Time Elapsed | Before (v0.4.0) | After (v0.6.2) |
|--------------|----------------|----------------|
| < 0.3s | `.discard` (silent) | `.discard` (silent) |
| ≥ 0.3s | ❌ Cancelled | ✅ **Ignored** (keeps recording) |

**Key Improvement:** After the 0.3s threshold, mouse clicks no longer cancel recordings. Only ESC can cancel, allowing users to click while recording long voice notes.

**Example:**
```swift
// User presses Option (modifier-only hotkey)
t=0.0s  Press Option       → START recording
t=0.5s  Click mouse        → IGNORED (keeps recording) ← NEW!
t=2.0s  Release Option     → STOP, TRANSCRIBE
```

**Code:**
```swift
// HotKeyProcessor.swift:228-236
if elapsed < effectiveMinimum {
    isDirty = true
    resetToIdle()
    return .discard
} else {
    // After threshold, ignore mouse clicks - let recording continue
    return nil  // ← Changed from .cancel
}
```

---

### 2.2 Extra Modifiers/Keys Handling

**Location:** `HotKeyProcessor.swift:381-393`

**Behavior Changes:**

For **modifier-only hotkeys** (e.g., Option):

| Time Elapsed | Action | Before (v0.4.0) | After (v0.6.2) |
|--------------|--------|----------------|----------------|
| < 0.3s | Press any key | `.stopRecording` | `.discard` (silent) |
| < 0.3s | Add modifier | `.stopRecording` | `.discard` (silent) |
| ≥ 0.3s | Press any key | 🟡 Mixed behavior | ✅ **Ignored** (keeps recording) |
| ≥ 0.3s | Add modifier | 🟡 Mixed behavior | ✅ **Ignored** (keeps recording) |

**Example:**
```swift
// Scenario: Recording while typing with Option held
t=0.0s  Press Option          → START recording
t=0.5s  Press A (Option+A)    → IGNORED (keeps recording) ← IMPROVED!
t=1.0s  Add Shift             → IGNORED (keeps recording) ← IMPROVED!
t=2.0s  Release Option        → STOP, TRANSCRIBE

// Old behavior would have cancelled at 0.5s
```

**Code:**
```swift
// HotKeyProcessor.swift:381-393
if hotkey.key == nil {
    let effectiveMinimum = max(minimumKeyTime, RecordingDecisionEngine.modifierOnlyMinimumDuration)

    if elapsed < effectiveMinimum {
        // Within threshold => discard silently (accidental trigger)
        isDirty = true
        resetToIdle()
        return .discard  // ← Changed to silent discard
    } else {
        // After threshold => ignore extra modifiers/keys, keep recording
        return nil  // ← NEW: Ignore instead of cancel
    }
}
```

---

## 3. macOS Sequoia Compatibility (Critical Fixes)

### 3.1 Input Monitoring Permission Handling

**Commits:**
- `9cd2757` - Fix hotkey monitoring on macOS Sequoia (fixes #122, #124)
- `471310c` - Fix Input Monitoring permission enforcement
- `7f6c5db` - Request Input Monitoring permission
- `7e325ad` - Fix Sequoia hotkey deadlock

**Issues Addressed:** #122, #124 (Hotkeys stop working on macOS Sequoia 15.7.1)

**Root Cause:** macOS Sequoia split Accessibility and Input Monitoring into separate permissions. Hex was only checking Accessibility.

**Solution:**

**File:** `Hex/Clients/KeyEventMonitorClient.swift`

```swift
// Added proper Input Monitoring checks throughout event-tap lifecycle
AXIsProcessTrustedWithOptions(options)  // Now used consistently
```

**File:** `Hex/Info.plist`

```xml
<!-- Added missing usage description -->
<key>NSInputMonitoringUsageDescription</key>
<string>Hex needs to monitor keyboard input to detect your hotkey...</string>
```

**Behavior:**
- Trust watchdog now polls same API that Settings UI uses
- Proper permission dialogs shown on Sequoia
- Event tap creation triggers permission prompt naturally
- Runtime checks aligned with Sequoia 15.7.1 behavior

---

### 3.2 Trust Watchdog & Tap Recovery

**Commit:** `df540e8` - Improve KeyEventMonitor reliability with trust watchdog and tap recovery

**Enhancement:** Added automatic recovery when CGEventTap gets disabled by the OS:
- Monitors accessibility trust status continuously
- Recovers from tap disabled events automatically
- Prevents hotkey failures during permission changes

**Changelog Entry (v0.5.0):**
> "Improve hotkey reliability with accessibility trust monitoring and automatic recovery from tap disabled events (#89, #81, #87)."

---

### 3.3 Voice Escape Hatch

**Commits:**
- `3560bdb` - Keep hotkeys alive on Sequoia and add voice force-quit (#122 #124)
- `7e325ad` - Re-add 'force quit Hex now' voice escape hatch

**Feature:** Added spoken command "force quit Hex now" to recover if permissions clobber input.

**Rationale:** If Input Monitoring permission breaks hotkeys, users can speak to force quit and reset.

---

## 4. ESC Key Improvements

### 4.1 LLM Processing Interruptibility (v0.6.1 - commit 804651c)

**Changelog Entry:**
> "Make LLM processing interruptible by pressing Escape"

**Enhancement:** ESC key now cancels not just recordings, but also in-progress LLM transformations.

**Impact:** Users can quickly cancel long-running AI operations.

---

### 4.2 ESC Handling Consistency

**Location:** `HotKeyProcessor.swift:164-172`

**Behavior:** Unchanged since v0.4.0, but now thoroughly documented:

```swift
// 1) ESC => immediate cancel
if keyEvent.key == .escape {
    let currentState = state
    hotKeyLogger.notice("ESC pressed while state=\(String(describing: currentState))")
}
if keyEvent.key == .escape, state != .idle {
    isDirty = true
    resetToIdle()
    return .cancel  // Plays cancel sound
}
```

**Works in:**
- `.pressAndHold` state
- `.doubleTapLock` state
- Sets dirty flag to prevent immediate re-triggering

---

## 5. Double-Tap Lock Enhancements

### 5.1 Key+Modifier Double-Tap Support (New in v0.4.0+)

**Location:** `HotKeyProcessor.swift:299-307`, `HotKeyProcessor.swift:343-358`

**Feature:** `useDoubleTapOnly` mode now properly supports key+modifier combinations.

**Before:** Only modifier-only hotkeys had reliable double-tap behavior.

**After:** Key+modifier hotkeys (e.g., Cmd+A) can use double-tap lock:

```swift
// handleMatchingChord()
if useDoubleTapOnly && hotkey.key != nil {
    // Record the timestamp but don't start recording
    lastTapAt = now
    return nil  // Wait for double-tap
}

// handleNonmatchingChord()
if useDoubleTapOnly && hotkey.key != nil &&
   chordIsFullyReleased(e) &&
   lastTapAt != nil {
    if let prevTapTime = lastTapAt,
       now.timeIntervalSince(prevTapTime) < Self.doubleTapThreshold {
        state = .doubleTapLock
        return .startRecording  // Locked!
    }
}
```

---

### 5.2 Full Release Detection

**Location:** `HotKeyProcessor.swift:411-414`

**Improvement:** Double-tap lock now properly requires full keyboard release to stop:

```swift
case .doubleTapLock:
    // For key+modifier combinations in doubleTapLock mode, require full key release to stop
    if useDoubleTapOnly && hotkey.key != nil && chordIsFullyReleased(e) {
        resetToIdle()
        return .stopRecording
    }
```

**Prevents:** Accidental stops from partial releases or modifier changes.

---

## 6. Dirty State Improvements

### 6.1 Backsliding Prevention

**Location:** `HotKeyProcessor.swift:50-56` (documentation)

**Behavior:** Unchanged logic, but now explicitly documented:

```
# Dirty State & Backsliding Prevention

After cancellation or with extra modifiers, processor enters "dirty" state:
- All input ignored until full release (key:nil, modifiers:[])
- Prevents accidental re-triggering during complex key combinations
- User cannot "backslide" into hotkey by releasing extra modifiers
```

**Example:**
```swift
User: Hold Option (0.1s) → Add Shift → Release Shift → Press Option again
      ↓
  START → DISCARD (dirty=true) → (ignored) → (ignored)

  User must release EVERYTHING (∅) to clear dirty

  → Release all keys → Now Option works again
```

---

### 6.2 Dirty Flag Preservation

**Location:** `HotKeyProcessor.swift:517-520`

**Documentation Added:**
```swift
/// Resets processor to idle state, clearing active recording state.
///
/// Preserves `isDirty` flag if caller has set it, allowing dirty state
/// to persist across state transitions for proper input blocking.
```

**Impact:** Clearer contract for state reset behavior.

---

## 7. Code Quality & Testing Improvements

### 7.1 Inline Documentation Expansion

**Statistics:**
- v0.4.0: 321 lines total
- v0.6.2: 522 lines total (+201 lines, +62%)
- Documentation: ~250 lines of comments/docs (~48% of file)

**Coverage:**
- Every public method documented with usage examples
- State machine transitions documented with flowcharts
- Edge cases explained with rationale
- Related components cross-referenced

---

### 7.2 Test Suite Expansion

**File:** `HexCore/Tests/HexCoreTests/HotKeyProcessorTests.swift`

**Changelog Entry (v0.5.13):**
> "Document Version: 2.0
> Last Updated: 2025-11-14
> **Total Tests: 46 passing ✅**"

**Test Coverage:**
- Modifier-only hotkey behavior
- Regular hotkey behavior
- Double-tap detection
- Mouse click handling
- Dirty state transitions
- ESC cancellation
- Threshold edge cases

---

### 7.3 Logging Improvements

**Commit:** `1deda2a` - use swift logging

**Enhancement:** Migrated to `swift-log` framework with HexLog categories:

```swift
private let hotKeyLogger = HexLog.hotKey

hotKeyLogger.notice("ESC pressed while state=\(String(describing: currentState))")
```

**Categories:**
- `.hotKey` - Hotkey processing events
- `.transcription` - Transcription pipeline
- `.recording` - Audio recording
- `.settings` - Settings changes

**Commit:** `6c2f1bd` - Add comprehensive permissions logging for improved debugging

**Impact:** Better diagnostics for Sequoia permission issues (#122, #124).

---

## 8. Changelog Integration

### 8.1 Changeset Workflow (v0.4.0+)

**Commit:** `e50478d` - Adopt Changesets for SemVer + changelog management

**Enhancement:** Added `.changeset/*.md` fragments for tracking changes:

```bash
bun run changeset:add-ai patch "Your summary here"
bun run changeset:add-ai minor "Add new feature"
bun run changeset:add-ai major "Breaking change"
```

**Commit:** `1ee452a` - Add non-interactive changeset creation for AI agents

**Impact:** Better release notes, clear version history, GitHub issue linking.

---

### 8.2 Notable Changelog Entries

**v0.6.2:**
> "Fix Sequoia hotkey deadlock by removing Input Monitoring guard that prevented CGEventTap creation. (#122 #124)"

**v0.6.1:**
> "Make LLM processing interruptible by pressing Escape"

**v0.5.13:**
> "Add comprehensive documentation to HotKeyProcessor and extract magic numbers into named constants (HexCoreConstants)"

**v0.5.12:**
> "Fix Input Monitoring permission enforcement for hotkey reliability"

**v0.5.8:**
> "Let the hotkey tap start even when Input Monitoring is missing so Sequoia users get prompts again (#122 #124). Add a spoken 'force quit Hex now' escape hatch."

**v0.5.4:**
> "Fix hotkey monitoring on macOS Sequoia 15.7.1 by properly handling Input Monitoring permissions (#122, #124)"

**v0.5.0:**
> "Improve hotkey reliability with accessibility trust monitoring and automatic recovery from tap disabled events (#89, #81, #87)."

**v0.4.0:**
> "Fn and other modifier-only hotkeys respect left/right side selection, ignore phantom arrow events, and stop firing when combined with other keys (#89, #81, #87)."

---

## 9. Summary of Key Behavioral Changes

### 9.1 Modifier-Only Hotkeys (e.g., Option)

| Scenario | Time | v0.4.0 | v0.6.2 |
|----------|------|--------|--------|
| **Quick tap** | < 0.3s | Discard | Discard (unchanged) |
| **Mouse click** | < 0.3s | Discard | Discard (unchanged) |
| **Mouse click** | ≥ 0.3s | ❌ Cancel | ✅ **Ignore** (keeps recording) |
| **Extra key** | < 0.3s | Stop | **Discard** (silent) |
| **Extra key** | ≥ 0.3s | 🟡 Mixed | ✅ **Ignore** (keeps recording) |
| **Extra modifier** | ≥ 0.3s | 🟡 Mixed | ✅ **Ignore** (keeps recording) |
| **ESC key** | Any time | Cancel | Cancel (unchanged) |

---

### 9.2 Regular Hotkeys (e.g., Cmd+A)

| Scenario | Time | v0.4.0 | v0.6.2 |
|----------|------|--------|--------|
| **Quick tap** | < minimumKeyTime | Discard | Discard (unchanged) |
| **Different key** | 0.2s - 1.0s | Stop | Stop (unchanged) |
| **Different key** | > 1.0s | Ignore | Ignore (unchanged) |
| **Extra modifier** | 0.2s - 1.0s | Stop | Stop (unchanged) |
| **ESC key** | Any time | Cancel | Cancel (unchanged) |

---

### 9.3 New Capabilities

1. **LLM Processing Cancellation** (v0.6.1)
   - ESC now cancels in-progress AI transformations

2. **Voice Force Quit** (v0.5.8, v0.6.2)
   - Speak "force quit Hex now" if permissions break hotkeys

3. **Better Permission Prompts** (v0.5.4+)
   - Proper Input Monitoring dialogs on Sequoia
   - Automatic recovery from permission changes

4. **Enhanced Logging** (v0.5.11)
   - Export logs from Advanced → Export Logs
   - Diagnose Sequoia permission bugs locally

---

## 10. Related Issues Fixed

| Issue # | Description | Fixed in Version |
|---------|-------------|------------------|
| #122 | Hotkeys stop working on Sequoia 15.7.1 | v0.5.4, v0.5.6, v0.5.8, v0.6.2 |
| #124 | Input Monitoring permission not requested | v0.5.4, v0.5.6, v0.5.8, v0.6.2 |
| #119 | Auto-send keyboard command feature | v0.6.0 |
| #117 | Microphone access retained when cancelled | v0.5.0 |
| #113 | Recording latency improvements | v0.4.0 |
| #89, #81, #87 | Fn/modifier hotkey reliability | v0.4.0, v0.5.0 |
| #69, #42 | Paste reliability (panel apps) | v0.4.0 |

---

## 11. Migration Impact

### Breaking Changes

**None.** All changes are backward-compatible improvements.

### User-Visible Changes

1. **Modifier-only hotkeys more forgiving:**
   - Can click/type after 0.3s without canceling recording
   - Only ESC cancels after threshold

2. **Better Sequoia compatibility:**
   - Proper permission dialogs
   - Hotkeys work reliably on macOS 15.7.1+

3. **ESC key more powerful:**
   - Cancels recordings (existing)
   - Cancels LLM processing (new in v0.6.1)

### Developer-Visible Changes

1. **Constants extracted:**
   - Import `HexCoreConstants` for timing values
   - No more magic numbers in code

2. **RecordingDecisionEngine separated:**
   - Testable decision logic
   - Clear separation of concerns

3. **Extensive documentation:**
   - Inline docs explain every method
   - `docs/hotkey-semantics.md` for behavior specs

---

## 12. Future Considerations

### Potential Improvements

1. **User-configurable thresholds:**
   - Currently `modifierOnlyMinimumDuration` (0.3s) is hardcoded
   - Could expose in Advanced Settings

2. **Per-hotkey thresholds:**
   - Different minimums for different hotkeys
   - Power users could fine-tune

3. **Adaptive thresholds:**
   - Learn user's typing patterns
   - Adjust thresholds based on usage

### Known Limitations

1. **0.3s modifier-only minimum:**
   - Cannot be lowered below 0.3s
   - Prevents OS shortcut conflicts
   - Documented in `HexCoreConstants.swift:24-40`

2. **macOS Sequoia permissions:**
   - Still requires manual permission grant
   - Cannot programmatically enable

---

## Conclusion

Keypress handling in Hex evolved significantly from v0.4.0 to v0.6.2:

- **Reliability:** macOS Sequoia compatibility fixed (#122, #124)
- **Usability:** Modifier-only hotkeys less prone to accidental cancellation
- **Quality:** 200+ lines of documentation, 46 test cases
- **Maintainability:** Extracted constants, separated concerns
- **Observability:** Swift logging, exportable diagnostics

The changes maintain backward compatibility while fixing critical bugs and improving user experience. The codebase is now well-documented, thoroughly tested, and ready for future enhancements.

---

**Report Generated:** November 22, 2025
**Analyzed Commits:** 69 commits from `3ac91f3` (v0.4.0) to `37ec257` (v0.6.2)
**Files Examined:**
- `HexCore/Sources/HexCore/Logic/HotKeyProcessor.swift`
- `HexCore/Sources/HexCore/Logic/RecordingDecision.swift`
- `HexCore/Sources/HexCore/Constants.swift`
- `docs/hotkey-semantics.md`
- `CHANGELOG.md`
- `Hex/Clients/KeyEventMonitorClient.swift`
