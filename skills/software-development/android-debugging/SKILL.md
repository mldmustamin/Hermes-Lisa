---
name: android-debugging
description: "Runtime debugging for Android apps — connectivity issues, manifest/permission problems, DI crashes, and backend integration failures."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [android, debugging, troubleshooting, kotlin, connectivity, di, hilt]
    related_skills: [systematic-debugging]
---

# Android Runtime Debugging

## Overview

Common Android app runtime failures follow predictable patterns. Many are caused by missing manifest declarations, wrong API URLs, or incorrect Hilt DI usage — not by business logic bugs.

Check these before diving into code-level debugging.

## Quick Diagnostic Checklist

When a user reports "app doesn't work" or "app crashes on [screen]", check in this order:

1. **INTERNET permission** — Missing → `socket failed: EPERM`
2. **Network Security Config** — Cleartext blocked → HTTP calls fail
3. **HTTP cleartext** — Android 9+ blocks HTTP → `CLEARTEXT communication not permitted`
4. **Backend endpoint exists?** — 404 from API call → Add route to backend
5. **hiltViewModel() target** — Class is not a ViewModel? → Force close on navigation
6. **Session data in Compose** — Not reading from SessionManager? → Blank screens
7. **Room FK constraint** — User not inserted into Room after login? → `FOREIGN KEY constraint failed (code 787)`
8. **Login freeze/hang** — HttpClient no timeout → See `references/android-common-crashes.md` item 8
8. **SyncWorker not registered** — App runs, data created locally, never appears on server? → WorkManager periodic sync not scheduled in `Application.onCreate()`
9. **Role-based UI gating missing** — User sees buttons/features they shouldn't have? (e.g., FIELD_ENGINEER sees "Create Project") → ViewModel not observing SessionManager roles to drive `canCreate*` flags in UiState. See `references/android-rbac-ui-gating.md` for full pattern.
10. **Room DB schema mismatch / "Migration didn't properly handle"** — App opens, shows Dashboard but crashes with Expected vs Found table info. Root cause: DB version was bumped in a previous build but tables weren't actually created (user's device has the newer version number with old schema). Fix: Bump DB version again (e.g. 9→10), create a NEW migration with `CREATE TABLE IF NOT EXISTS` for all new tables. The migration won't re-run if version matches. See `references/room-migration-debugging.md`.

11. **App freezes/hangs on login (no error message)** — Login button pressed, spinner spins forever, no error toast → HttpClient (Ktor) has no timeout configured. Request hangs indefinitely. Fix: add `install(HttpTimeout)` in `NetworkModule` with `requestTimeoutMillis=15000`, `connectTimeoutMillis=8000`, `socketTimeoutMillis=15000`. After timeout, existing `onFailure` handler in ViewModel will display error. See `references/httpclient-timeout.md`.

12. **App freezes on login even WITH timeout** — After adding HttpTimeout, login still hangs 15-30 seconds before any response. Root cause: Device registration (`deviceRepository.registerDevice()`) is called INSIDE the login `onSuccess` handler, making another HTTP call that blocks the UI until it completes. The `isLoading = false` line only runs AFTER device registration finishes → total freeze time = login timeout + device registration timeout (30s). Fix: Move device registration to a separate `viewModelScope.launch` coroutine (`registerDeviceInBackground()`), set `isLoading = false` BEFORE the background registration, and wrap session save in try-catch so login succeeds even if local storage fails. Also improve error messages for specific failures: timeout → "Koneksi timeout. Server tidak merespon.", DNS failure → "Tidak dapat terhubung ke server.", Connection refused → "Server sedang sibuk." See `references/login-freeze-device-registration.md`.

13. **AVD headless debugging setup** — Need to test Android app on a server without GUI? Install Android SDK command-line tools, emulator, and system image. Start emulator with `-no-window -no-audio -gpu swiftshader_indirect`. Use `adb -s emulator-5554` for all commands. Note: Text input via `adb shell input text` may not work on Compose TextFields because IME is not connected in headless mode. Use `uiautomator dump` + XML parsing + coordinate taps instead. See `references/avd-headless-setup.md` for complete guide including input simulation for headless testing.

## Reference Documents

- `references/android-common-crashes.md` — detailed fix procedures for each crash pattern
- `references/android-rbac-ui-gating.md` — role-based navigation and screen visibility
- `references/room-migration-debugging.md` — Room DB schema mismatch + migration failures
- `references/login-freeze-debugging.md` — login hangs, timeout, background blocking
- `references/avd-headless-setup.md` — AVD on server without GUI
See `references/sync-worker-registration.md` for the SyncWorker WorkManager scheduling pattern.
See `references/android-rbac-ui-gating.md` for the role-based UI gating pattern (ViewModel observes roles, UiState boolean flags, conditional Compose rendering).
