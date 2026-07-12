# Title

CameraSource: AVFoundation still capture (CLI path)

## Summary

Implement `CameraSource: ScanSource` (AVFoundation single still capture with device
discovery incl. Continuity Camera), the CLI Info.plist embedding for the camera usage
string, and validate the KU-2 TCC attribution question first.

## Context

Camera is the riskiest source (KU-2): a terminal-spawned CLI may not get a camera prompt
attributed correctly even with an embedded Info.plist. This issue validates that FIRST
and carries the documented fallback. The app path (Issue 21) reuses this source.

## Scope

- `Sources/ScanMDKit/Sources/CameraSource.swift`, `CameraDiscovery.swift`.
- `Sources/scanmd/Info.plist` + `Package.swift` linker flags.
- Fake-based unit tests + manual hardware checklist.

## Detailed Requirements

0. **KU-2 spike gate (do this before the rest)**: build a 30-line scratch target with the
   `-sectcreate __TEXT __info_plist` embedding and call
   `AVCaptureDevice.requestAccess(.video)` from Terminal on macOS 14/15. Record in the PR:
   which app the prompt names, whether grant persists, `tccutil reset Camera` behavior.
   If attribution fails → implement the fallback: `scanmd camera` exits 3 with message
   `camera capture from the CLI is not supported on this macOS version; use ScanMD.app
   (menu bar) camera capture instead`, the rest of this issue still lands (the source is
   app-consumed), and ISSUE_PLAN KU-2 is updated with the finding.
1. Info.plist embedding (CLI): `Sources/scanmd/Info.plist` with
   `NSCameraUsageDescription = "scanmd captures a single photo to convert its text to
   Markdown."`, `CFBundleIdentifier = dev.saber5656.scanmd.cli`, `CFBundleName = scanmd`,
   version keys templated at release. `Package.swift` scanmd target:
   `linkerSettings: [.unsafeFlags(["-Xlinker","-sectcreate","-Xlinker","__TEXT",
   "-Xlinker","__info_plist","-Xlinker","Sources/scanmd/Info.plist"])]`.
2. `CameraDiscovery.devices()` → `[CameraDevice { index, name, deviceType,
   isContinuity }]` via `AVCaptureDevice.DiscoverySession(deviceTypes:
   [.builtInWideAngleCamera, .continuityCamera, .external], mediaType: .video,
   position: .unspecified)`, stable ordering (built-in first, then alphabetical), with a
   3 s discovery timeout guard (KU: continuity devices can appear late — document actual
   behavior).
3. Selection: `--device <index|substring>` semantics implemented here as
   `select(devices:, query: String?) -> CameraDevice` — index if the query is an
   integer, else case-insensitive name substring (ambiguous → `.usage` listing matches);
   nil query → config `camera.device` (same semantics) → first device; empty list →
   `.noInput("no camera device found")`.
4. Capture: preflight `authorizationStatus(for: .video)` (`.denied/.restricted` →
   `.permissionDenied(.camera)`; `.notDetermined` → `requestAccess` await).
   `AVCaptureSession` (`.photo` preset) + `AVCapturePhotoOutput`; warm up
   `camera.warmupMs`; capture one photo (no flash); overall deadline
   `limits.captureTimeoutSec` → `.timeout`. Convert to CGImage (orientation upright),
   kind `.camera`, single payload. Countdown/stderr UX belongs to the CLI (Issue 18) —
   the source exposes `onCountdownTick` callback hooks instead.
5. Abstraction for tests: `CameraSessionRunning` protocol; fakes simulate discovery
   lists, auth states, capture success/timeout. No real camera in CI.
6. Manual checklist (real hardware, paste results): built-in FaceTime camera capture ·
   iPhone Continuity capture (same Apple ID, Wi-Fi/BT on) · `--list-devices` shows both ·
   denied permission → exit 3 with §8.6 message · warm-up produces non-black exposure.

## Acceptance Criteria

- [ ] KU-2 spike result documented in PR + ISSUE_PLAN updated (either outcome).
- [ ] Fake-based tests: selection semantics (index/substring/ambiguous/config/default/
      empty), auth mapping, timeout mapping.
- [ ] `strings scanmd-binary | grep NSCameraUsageDescription` finds the usage text
      (embedding verified in CI via a build step).
- [ ] Manual hardware checklist completed (or fallback path documented + tested).

## Validation

`swift test --filter CameraSourceTests`; embedding check script output; manual evidence
in PR.

## Dependencies

Issues 04, 05, 06.

## Non-goals

Preview UI (app window — Issue 21), document-scan sheet (v2), video/multi-shot modes.

## Design References

DESIGN §7.5, §10.1-B2, §10.3; research/macos-capture-ocr-apis.md (camera facts);
ISSUE_PLAN KU-2.
