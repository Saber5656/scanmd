# Title

App result delivery: clipboard, notifications, auto-save

## Summary

Replace Issue 20's temporary alerts with the final delivery UX: success/empty/error
notifications via `UNUserNotificationCenter` (with settings deep-link actions), plus
optional auto-save through `FileSink`.

## Context

DESIGN §9.5 defines delivery; Issue 04 provides `remediationURL`; Issue 11 provides
`FileSink`. KU-7 flags notification reliability for unsigned dev builds.

## Scope

- `App/ScanMD/Delivery/NotificationCenter+ScanMD.swift`, `DeliveryHandler.swift`;
  `CaptureCoordinator` delivering-phase rewrite.

## Detailed Requirements

1. Authorization: if `app.showNotification` is true, request `UNUserNotificationCenter`
   authorization (`.alert`) lazily — on first delivery, not at launch (DESIGN §10.3
   spirit). Denied → fall back to a brief `NSAlert`-free path: menu bar icon flashes
   success (SF Symbol swap for 1.5 s) and errors fall back to `NSAlert` (errors must never
   be silent). If `app.showNotification` is false, do not request authorization and do not
   post success/empty notifications; clipboard writes and auto-save still run, and errors
   still surface via the non-notification fallback.
2. Success: clipboard write (always, via `ClipboardSink`) → notification title
   `Copied as Markdown`, body = first 120 chars of the markdown (sanitized to a single
   line, ellipsis) — note: notification content is user-visible by design; do NOT log it
   (p1 still applies to logs).
3. `.emptyResult`: title `No text found`, body names the source kind.
4. Errors: title `Capture failed`, body = `userMessage`. If `remediationURL != nil`,
   attach a notification category with action button `Open System Settings` that opens
   the URL (`UNNotificationAction`; handle in the delegate; app is `LSUIElement` so set
   the delegate on launch).
5. Auto-save (`app.autoSave && output.directory != nil`): run `FileSink` in directory
   mode after the clipboard write; when notifications are enabled, success appends
   ` · saved as <basename>` to the notification body; failure sends a **separate** error
   notification (clipboard content is intact — say so in the body:
   `Result is still on the clipboard`). When notifications are disabled, autosave success
   is silent and autosave failure uses the same non-notification error fallback as other
   errors.
6. Notification identifiers: one mutable delivery notification per capture
   (`scanmd.capture.<uuid>`); remove delivered ones older than the last 5 (no center
   spam).
7. Tests: `DeliveryHandler` behind `UserNotifying` + sink fakes — matrix: success/
   empty/error×(remediation yes/no)×(autosave off/on/success/fail) asserting
   title/body/action/category and sink calls. Delegate action-routing unit-tested with a
   fake response.
8. Manual checklist (KU-7): notifications appear for success/empty/error on a **signed
   dev build** (ad-hoc ok; if unsigned build drops notifications, record it per KU-7
   and validate on the signed build) · action button opens the exact Settings pane ·
   auto-save writes the templated file and the body shows the name · auto-save failure
   (read-only folder) keeps clipboard + shows the follow-up error · Focus/DND behavior
   noted.

## Acceptance Criteria

- [ ] Delivery matrix tests green; alerts fully removed from success paths (grep:
      `NSAlert` remains only in denied-notification error fallback).
- [ ] Notification bodies never exceed one line/120 chars; identifiers pruned to 5.
- [ ] Manual checklist evidence incl. KU-7 finding recorded in ISSUE_PLAN if relevant.
- [ ] Clipboard always holds the result even when auto-save fails (tested).

## Validation

`swift test --filter DeliveryTests` + manual checklist with screenshots of each
notification kind.

## Dependencies

Issues 11, 20.

## Non-goals

Capture history UI, "Save As…" notification action (v2), sounds/badges.

## Design References

DESIGN §9.5, §8.6, §10.2-p1, §10.3; ISSUE_PLAN KU-7.
