# Title

App camera preview window

## Summary

Implement the "From Camera…" flow: a preview window with live `AVCaptureVideoPreviewLayer`,
device picker (built-in + Continuity iPhone), Capture/Cancel controls, feeding the
standard pipeline.

## Context

DESIGN §9.3. Reuses `CameraSource`/`CameraDiscovery` (Issue 16) for discovery, auth,
and the still-capture backend; this issue adds the AppKit/SwiftUI window UX that a CLI
cannot offer.

## Scope

- `App/ScanMD/Camera/CameraWindowController.swift`, `CameraPreviewView.swift`,
  `CameraViewModel.swift`; `CaptureCoordinator` `.camera` kind wiring.

## Detailed Requirements

1. Window: titled, non-resizable 720×540, centered, `NSWindow.level = .floating`,
   closes on Esc (Cancel) and on capture completion. Only one instance (invoking while
   open brings it to front — coordinator busy-rejection already prevents double
   pipelines; opening the window is `acquiring` state).
2. Content: preview layer filling the top area (`videoGravity =
   .resizeAspect`), bottom bar with: device `Picker` (from `CameraDiscovery.devices()`,
   refreshed on `AVCaptureDevice.wasConnectedNotification`/`wasDisconnected` so a
   late-appearing iPhone shows up), `Capture` button (default, Return key), `Cancel`
   button (Esc).
3. Session lifecycle: `AVCaptureSession` starts when the window appears, stops on close
   (never keep the camera running in the background — privacy). Switching device swaps
   the session input without tearing down the window. Auth denied → window never opens;
   coordinator shows the §8.6 camera remediation alert (deep link to the Camera pane).
4. Capture: button triggers the same `AVCapturePhotoOutput` path as Issue 16 (share the
   session — refactor `CameraSource` so the capture routine accepts an
   externally-managed session; CLI path behavior unchanged, its tests must stay green).
   Captured `CGImage` → window closes → pipeline continues (recognizing → delivering).
5. Config: `camera.device` preselects the picker; the picker choice is session-local
   (not persisted — persistence is a Settings concern, v2).
6. No countdown in the app (live preview replaces it).
7. Tests: view-model unit tests with fakes (device list refresh on connect
   notification, selection swap, capture→close sequence, denied-auth path). UI itself is
   manual.
8. Manual checklist: preview shows built-in camera · iPhone appears in picker within a
   few seconds (Continuity requirements noted: same Apple ID, BT/Wi-Fi on) · switching
   devices live works · Return captures, Esc cancels, window closes, camera indicator
   light goes OFF after close in all paths · captured whiteboard photo yields Markdown
   on clipboard.

## Acceptance Criteria

- [ ] Full manual checklist evidence (both device kinds) in the PR.
- [ ] Camera never runs while the window is closed (checked via the macOS camera
      indicator; auto-verified by asserting `session.isRunning == false` after close in
      a hosted test if feasible).
- [ ] CLI camera tests from Issue 16 remain green after the session refactor.
- [ ] View-model tests green.

## Validation

Manual checklist + `swift test` + app screen recording or step log in PR.

## Dependencies

Issues 16, 19, 20. Issue 20's real `CaptureCoordinator` is a hard dependency because
`.camera` follows the same busy-rejection, pipeline, and delivery path.

## Non-goals

Continuity Camera document-scan sheet (`importFromDevice:`, v2 — DESIGN §13), video,
burst/multi-page scanning, device persistence.

## Design References

DESIGN §9.3, §9.4, §7.5; research/macos-capture-ocr-apis.md (camera facts).
