# Title

Global hotkey, Settings window, launch at login

## Summary

Add the configurable global hotkey (default ⌃⌥⌘M → Capture Region) via
KeyboardShortcuts, the three-tab Settings window bound to the shared config file with
live reload, and launch-at-login via `SMAppService`.

## Context

DESIGN §9.6 defines the Settings surface; ADR-006 puts all settings except the hotkey in
`config.json` (the hotkey lives in KeyboardShortcuts' own store). KU-4 tracks the
dependency pin.

## Scope

- `App/ScanMD/Settings/` (`GeneralTab.swift`, `OCRTab.swift`, `ShortcutsTab.swift`,
  `SettingsView.swift`), `HotkeyManager.swift`, `ConfigStore.swift` (observable bridge +
  file watcher), login item wiring.

## Detailed Requirements

1. Hotkey: `KeyboardShortcuts.Name("captureRegion")` with default `⌃⌥⌘M`;
   `HotkeyManager` registers on launch and calls
   `CaptureCoordinator.shared.request(.region)`. Busy state → coordinator's beep-drop
   already applies. Recording UI = `KeyboardShortcuts.Recorder` in the Shortcuts tab.
   Pin the package `exact` to a version verified against macOS 14 (record the verified
   version in the PR → resolves KU-4; if incompatible, STOP and open the Carbon-wrapper
   follow-up issue per ISSUE_PLAN KU-4 contingency before proceeding).
2. `ConfigStore` (`@MainActor ObservableObject`):
   - loads via `ConfigLoader` (Issue 06); publishes typed bindings for the §9.6 controls;
   - saves through `ConfigLoader.save` (atomic, 0600) with 500 ms debounce;
   - watches the file via `DispatchSource.makeFileSystemObjectSource` (vnode write/
     delete/rename on the containing directory to survive atomic replaces); external
     change → reload + republish (last-writer-wins, no merge — ADR-006);
   - reload storms guarded (ignore events within 200 ms after own save).
3. General tab: copy-to-clipboard toggle **disabled+on** with caption "always on in
   v1" · auto-save toggle + folder picker (`NSOpenPanel.canChooseDirectories`, path
   stored in `output.directory`) · notification toggle (`app.showNotification`) ·
   launch-at-login toggle → `SMAppService.mainApp.register()/unregister()`, reflecting
   `status` on appear (handle `.requiresApproval` by showing a hint linking to Login
   Items settings).
4. OCR tab: language priority editable list seeded from `ocr.languages` (reorder via
   drag, add via validated text field using Issue 08's `validate(languages:)`, remove;
   minimum one language) · auto-detect toggle · accurate/fast picker.
5. Shortcuts tab: the Recorder + a static hint that CLI users can bind
   `scanmd` in Raycast/Shortcuts too.
6. Captures pick up changed config immediately (Issue 20 reads per capture — verify).
7. Tests: `ConfigStore` round-trip (edit → debounce save → file content), external-edit
   reload (rewrite file in test → published values change), storm guard, login-item
   calls behind a `SMAppServicing` fake. UI tabs are manual.
8. Manual checklist: hotkey fires from any app (incl. full-screen apps — record
   result) · re-recording the shortcut takes effect immediately and survives relaunch ·
   every General/OCR control round-trips to `config.json` (show file diff) · editing
   `config.json` in a text editor while Settings is open updates the UI · launch at
   login works after logout/login (or `.requiresApproval` path documented) · CLI sees
   the same config changes (`scanmd config path` + a capture honoring changed
   languages).

## Acceptance Criteria

- [ ] Hotkey default ⌃⌥⌘M works and is re-recordable; KU-4 resolved with a pinned,
      verified version.
- [ ] Settings tabs match DESIGN §9.6 exactly (controls, bindings, disabled states).
- [ ] Config file remains the single source of truth (no shadow UserDefaults except
      KeyboardShortcuts' own store) — code-reviewed grep for `UserDefaults`.
- [ ] Store tests green; manual checklist evidence attached.

## Validation

`swift test --filter ConfigStoreTests` + manual checklist + config-diff transcript.

## Dependencies

Issues 06, 19.

## Non-goals

Per-capture-kind hotkeys (v2), menu customization, localization (v2), notification
content settings beyond the toggle.

## Design References

DESIGN §9.5, §9.6, §8.7; ADR-006; ISSUE_PLAN KU-4.
