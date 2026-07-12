# ADR-004: Region capture shells out to `/usr/sbin/screencapture -i` in v1

- Status: Accepted
- Date: 2026-07-12
- Decider: Fable (design), conservative default — reversible behind `ScanSource`

## Context

macOS provides no public API for the interactive "drag a region" selection UI.
Options: (a) spawn the system `screencapture -i` binary, which brings its own selection UI;
(b) implement a custom overlay (transparent `NSWindow` across displays + crosshair +
`SCScreenshotManager` capture), as tools like TRex do.

## Decision

v1 uses **(a)**: `ScreenRegionSource` spawns `/usr/sbin/screencapture` with argv
`["-i", "-x", <tmpPath>]` via `posix_spawn` (no shell), applying the subprocess controls in
DESIGN §7.3/§10.1-B3 (absolute path, minimal env, 0600 temp file, deferred cleanup, timeout
+ kill, cancel detection via exit status / missing file).

A custom overlay is explicitly deferred to v2 and slots in as a new `ScanSource`
implementation with no changes elsewhere.

## Consequences

- v1 gets the exact selection UX users already know (drag, Esc to cancel, space for window
  mode) with near-zero UI code and no multi-display math.
- TCC attribution: when invoked from the CLI, Screen Recording permission attaches to the
  invoking terminal app; from `ScanMD.app`, to the app itself. Remediation messages must
  name "the app that runs scanmd" (DESIGN §8.6).
- We depend on undocumented-but-stable `screencapture` behavior for cancel signaling; the
  wrapper treats *either* nonzero exit *or* absent/empty file as cancel (KU-5 tracks
  verifying this matrix across macOS 14/15).
- Window-picker mode (space key) comes for free inside `-i`.

## Alternatives considered

- Custom overlay + ScreenCaptureKit in v1: best-in-class UX (dim overlay, magnifier) but
  adds the largest single chunk of UI code, multi-display and Spaces edge cases, and direct
  ScreenCaptureKit permission handling. Deferred, not rejected.
- `CGWindowListCreateImage` era APIs: deprecated paths; rejected.
