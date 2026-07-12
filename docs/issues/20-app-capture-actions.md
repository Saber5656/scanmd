# Title

App capture actions and state machine

## Summary

Implement the app's four file-less capture actions — Capture Region, From Clipboard,
From Image File…, From PDF… — through the shared pipeline, completing the
`CaptureCoordinator` state machine with busy-rejection.

## Context

Fills Issue 19's stubs using the Kit sources (Issues 12–15). Camera (Issue 21) and
delivery polish (Issue 23) are separate; in this issue delivery = clipboard write only
(the always-on baseline of DESIGN §9.5) so the flow is testable end-to-end.

## Scope

- `App/ScanMD/CaptureCoordinator.swift` (real implementation), `PipelineFactory.swift`,
  `OpenPanels.swift`.

## Detailed Requirements

1. `CaptureCoordinator.request(_ kind: CaptureKind)` (`@MainActor`):
   - if `state != .idle` → `NSSound.beep()`, drop the event (DESIGN §9.4), return.
   - transition per §9.4 driving a `Task` that runs acquire → pipeline → deliver with
     the heavy stages **off** the main actor; state updates back on main.
2. Source construction per kind:
   - `.region` → `ScreenRegionSource` (permission preflight inside the source; app is
     the TCC-responsible process here — remediation alert on `.permissionDenied` with an
     "Open System Settings" button using `remediationURL` from Issue 04).
   - `.clipboard` → `ClipboardSource` (with PDF delegation factory as in Issue 18).
   - `.imageFile` → `NSOpenPanel`: `allowedContentTypes = [.png, .jpeg, .heic, .tiff,
     .gif, .webP]`, single selection, panel runs modally from the menu action; cancel →
     back to idle silently.
   - `.pdf` → `NSOpenPanel` with `[.pdf]`; after selection, if the document is large
     (> 20 pages) show a determinate progress in the menu bar icon tooltip (simple:
     state stays `recognizing`; per-page progress via a `@Published pageProgress:
     (Int, Int)?` — rendered as menu item text `Recognizing page 3 of 12`, disabled).
3. Config: coordinator reads `ScanMDConfig` via `ConfigLoader` at each capture (cheap,
   avoids staleness before Issue 22's file watching lands).
4. Delivery (baseline): `ClipboardSink` always; errors → `NSAlert` for now
   (notifications arrive in Issue 23; keep the error text = `userMessage`).
   `.emptyResult` shows an alert "No text found" (temporary UX, replaced in 23).
5. Concurrency: exactly one capture task; task stored, `.cancelled` results return to
   idle without alerts. App termination while busy cancels the task cleanly.
6. Unit tests (extracted, UI-free): reducer covers §9.4 rows incl. busy-drop; a fake
   source + fake sink drive a full coordinator run asserting state sequence
   `idle→acquiring→recognizing→delivering→idle` and clipboard-sink invocation; error
   and cancel paths return to idle with the right effect enum.
7. Manual checklist: region capture end-to-end from the menu (Markdown lands on
   clipboard — paste into a text editor) · clipboard capture of a screenshot · image
   file via panel · 12-page mixed PDF shows page progress and completes · double-invoke
   while busy beeps and drops · TCC-denied region shows the settings-link alert.

## Acceptance Criteria

- [ ] All four actions work end-to-end on a dev build (manual checklist evidence).
- [ ] Reducer/state tests green; no capture path blocks the main thread (verify via
      Instruments hang detector once, note in PR).
- [ ] Busy-rejection behaves per §9.4 (beep + drop, no queueing).
- [ ] Permission alert deep-links to the Screen Recording pane.

## Validation

Manual checklist + unit tests + a short screen recording (or step log) attached to the
PR.

## Dependencies

Issues 12, 13, 14, 15, 19.

## Non-goals

Camera (21), notifications/auto-save (23), settings/hotkey (22).

## Design References

DESIGN §9.2, §9.4, §9.5 (baseline), §8.6 (remediation).
