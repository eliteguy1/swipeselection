# SwipeSelection 2.0.1 — tap-delete removes 2 chars (double-delete fix)

## 1. Pre-check & Analysis

- Device: iPhone 11 / iOS 18.7.1 / Relaxin + RootHide (arm64e). Same environment as keyboard project.
- Package: `com.icraze.swipeselection` 2.0 (iCraze rebuild of Kyle Howells' SwipeSelection), filter `com.apple.UIKit`, FAT arm64+arm64e, 151,664-byte dylib.
- Symptom: every tap on backspace deletes exactly 2 characters.
- Root cause (from real sources, not guesses):
  - Kyle Howells' original `touchesEnded` (VERIFIED via `kylehowells/SwipeSelection` repo, `SwipeSelection/Tweak.mm`): calls `%orig` first, then MANUALLY calls `[kb handleDelete]` when the touch lifts on delete and `!isLongPressed`. Its `handleDelete` hook swallows the tap-time delete (`if (!isLongPressed && isDeleteKey) {}` empty branch). On iOS <= 12 this nets exactly 1 char (swallowed system + 1 manual).
  - On iOS 18 the system performs the tap-delete itself through a modern path the old code does not swallow (layout `deleteAction` / `handleDeleteWithNonZeroInputCount` chain — both selectors VERIFIED present in the 2.0 binary's `__objc_methname`). Static disassembly (capstone, arm64e slice, 22 functions) confirms the 2.0 binary keeps the original hook set (`touchesBegan/Moved/Ended`, `handleDelete`, `handleDeleteAsRepeat:executionContext:`, pan gesture) plus an added word-delete counter in the repeat hook.
  - Result: 1 char (system self-delete) + 1 char (legacy manual re-fire) = exactly 2 per tap. The symptom itself proves the system path is live.
- Fix safety arithmetic: removing the manual re-fire leaves exactly the 1 system char. Hold-to-repeat is unaffected (manual call was guarded by `!isLongPressed`; repeat flows through `handleDeleteAsRepeat:executionContext:` with `%orig`). The `handleDelete` swallow is left untouched (harmless either way).
- Claim status: `system self-deletes on iOS 18 tap` = HYPOTHESIS (strong, forced by symptom arithmetic); `manual re-fire adds exactly 1` = VERIFIED (original source); `removal leaves 1 char` = VERIFIED by arithmetic; `repeat unaffected` = VERIFIED (guard in source).

## 2. Executed Commands

Local (Windows, `D:\llama agent\tweak\swipeselection`):
- `New-Item -ItemType Directory` for extracted/src/build/mcp/logs/temp/docs — ok.
- `.deb` split via custom Python `ar` parser (`arx.py`) — ok (`debian-binary`, `control.tar.zst`, `data.tar.zst`); `tar -xf` ok except zero-byte `.roothidepatch` (Windows-invalid name, irrelevant).
- Mach-O parsing scripts (`symchk.py`, `disx.py`, `secmap.py`, `stubmap.py`, `annotate.py`): FAT arm64+arm64e confirmed; 39 symtab entries; 22 functions disassembled (`disasm.txt`, 2444 lines); 38-entry stub map resolved (`_MSHookMessageEx`, `objc_retain/release`, ...); selrefs/classrefs are zero-on-disk (chained fixups) so selector args are not statically resolvable — pivoted to original source after 2 failed resolution attempts per workspace loop rule.
- GitHub MCP `github_get_file_contents` (kylehowells/SwipeSelection) + `webfetch` raw `Tweak.mm`/`Makefile`/`SwipeSelection.plist` — ok.
- `ssh csw@192.168.1.21` — Connection timed out (VM unreachable this session); build NOT run.

## 3. Code Changes Detail

- `src/Tweak.x` (new, from verified original `Tweak.mm`, converted to `.x` — no C++ constructs present): single functional change in `UIKeyboardLayoutStar touchesEnded`: removed the manual `[kb handleDelete]` / `handleDeleteAsRepeat:` / `handleDeleteWithNonZeroInputCount` re-fire block and its now-unused key lookup. Everything else byte-identical in logic to the original.
- `src/Makefile` (new): `THEOS_PACKAGE_SCHEME=rootless`, `ARCHS=arm64 arm64e`, `TARGET=iphone:clang:latest:15.0`, `-fno-objc-arc` (source uses manual retain/release — VERIFIED `release` calls in source).
- `src/control` (new): same package id/name/meta, `Version: 2.0.1`, `Architecture: iphoneos-arm64` (rootless).
- `src/SwipeSelection.plist` (new): identical UIKit-bundles filter.
- Known tradeoff: iCraze 2.0's extra word-delete-counter logic inside `handleDeleteAsRepeat:executionContext:` (seen in disasm, not in original source) is NOT carried over — source unavailable, not reinvented per workspace rules. Core swipe-cursor/select preserved; delete-words-on-swipe-from-delete may differ from 2.0 behavior (OPEN ISSUE to verify on-device).

## 4. Build Diagnostics & Warnings

- VM `macOS 14.vmx` was down; started via `vmrun -T player start nogui`, SSH up after boot.
- `make package` round 1: 7 errors. 1 real (transcription dropped the whole `@interface UIKeyboardTaskQueue` block — restored from verified original source) + 6 `-Werror` deprecation errors under SDK 26.0 (`setTargetRect:inView:`, `setMenuVisible:animated:`, `UI_USER_INTERFACE_IDIOM()`, `UITextWritingDirectionRightToLeft`, plus 2 cascade errors from the missing interface). Fix: `-Wno-error=deprecated-declarations` in `SwipeSelection_CFLAGS` (warnings kept, logic untouched).
- Round 2: compiles clean (0 errors; 4 deprecation warnings, harmless — same calls as original). Link ok both slices. Signing failed: `ldid: command not found` — resolved via `export PATH='/usr/local/bin:/opt/homebrew/bin:...'` (`ldid` lives in `/usr/local/bin`, missing from non-interactive ssh PATH).
- Scheme: first built rootless (`iphoneos-arm64`, `-1+debug`); switched to `THEOS_PACKAGE_SCHEME=roothide` + `FINALPACKAGE=1` to match original 2.0 packaging (`iphoneos-arm64e`, release). Theos does not emit `.roothidepatch`, so a zero-byte marker was added via `src/layout/` to replicate the original package (VERIFIED present in 2.0 deb).
- Linker: `ld: warning: -multiply_defined is obsolete` (benign, theos default flag).

## 5. Artifact Integrity

- `build/com.icraze.swipeselection_2.0.1_iphoneos-arm64e.deb`, 7,730 bytes.
- SHA256 `9690553196380cc74387bc7bbb83e96476bb8e09fb663c8f5813fe41192c0586` — identical local vs VM (VERIFIED).
- FAT dylib: `arm64 arm64e` (VERIFIED via `lipo -info`).
- `data.tar.*` members: `SwipeSelection.dylib`, `SwipeSelection.dylib.roothidepatch` (0-byte), `SwipeSelection.plist` — matches original 2.0 layout.
- control: `Version: 2.0.1`, `Architecture: iphoneos-arm64e`, UIKit-bundles filter unchanged.

## 6. Impact & Regression Risks

- SpringBoard/keyboard hosts: filter unchanged (UIKit everywhere) — same injection footprint as 2.0.
- Single-tap delete: expected 1 char (was 2). If 0 chars appear on-device, the system path assumption is wrong for that app — report immediately (would mean re-adding a guarded manual call).
- Hold-delete repeat: expected unchanged continuous delete. Watch for: repeat stopping after 1 char (would indicate the removed block was load-bearing — unlikely per guard analysis).
- Swipe-cursor / shift-select / more-key suppression: untouched code paths, but verify cursor movement + 2-finger word skip still work since the file was retranscribed.
## 7. 2.0.1 Safe-Mode Regression (2026-10-15) + 2.0.2

- 2.0.1 installed -> respring -> SpringBoard safe mode. Uninstalled; phone recovered. The `apt-get` crash log the user provided is a SEPARATE event (package-manager abort inside `DoInstall`/`writeData:` while REMOVING 2.0.1, bug_type 309) — not the SpringBoard cause.
- 2.0's EXACT hook set recovered from static analysis (VERIFIED: capstone disasm of arm64e slice + `dyld_info -fixups` selref->`__objc_methname` mapping + cstring cross-check):
  - `UIKeyboardImpl`: `initWithFrame:`, `initWithFrame:forCustomInputView:` (NEW vs Kyle), `isLongPress`, `handleDelete`, `handleDeleteAsRepeat:executionContext:`, plus `%new` `SS_KeyboardGestureDidPan:`, `SS_revealSelection:` and an `SS_pan` associated property (`setSS_pan:`/`SS_pan`, `class_addProperty`).
  - `UIKeyboardLayoutStar`: `touchesBegan/Moved/Cancelled/Ended:withEvent:`, plus `%new` `SS_shouldSelect`, `SS_disableSwipes`, `SS_isKanaKey`.
  - `_UIKeyboardTextSelectionInteraction`: `gestureRecognizerShouldBegin:` (NEW vs Kyle — yields our pan to Apple's selection interaction).
  - NOT hooked (neither 2.0 nor Kyle): `handleDeleteAsRepeat:` (2-arg), `handleDeleteWithNonZeroInputCount`, `deleteAction`, `stopAutoDelete`.
- Consequence: 2.0.1's hook set is a SUBSET of working 2.0's. A missing hook cannot crash; therefore the safe-mode cause is NOT the hook selection. Remaining suspects (OPEN ISSUE, needs device evidence): toolchain/packaging delta (SDK 26.0 vs 16.2, theos `@rpath` install_name vs absolute, dm.pl tar layout) or device state (the later apt abort suggests possible disk/dpkg distress around the same window).
- 2.0.2 changes (minimal deltas toward 2.0 fidelity): `+initWithFrame:forCustomInputView:` hook (pan install on the modern init path — without it the gesture may never install on iOS 16+), restored `Pre-Depends: rootless-compat(>= 0.9)` in control. The iCraze extras (word-delete counter, TextSelectionInteraction yield, kana check, SS_pan property) intentionally NOT reimplemented (touch-time features, unrelated to load crash; HYPOTHESIS).
- Artifact: `build/com.icraze.swipeselection_2.0.2_iphoneos-arm64e.deb`, SHA256 `bf9caa59b04de80edb218b39e30812358e6ba82491f6e64e0e4a3caf91f8eb56` (VERIFIED local vs VM). 0 errors, same 4 deprecation warnings. On-device test PENDING.
- If 2.0.2 also safe-modes: pull `SpringBoard-*.ips` (faulting frame decides ctor-hook vs packaging) and check free storage + `dpkg --configure -a` state.
