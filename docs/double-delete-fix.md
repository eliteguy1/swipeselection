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

PENDING — macOS VM unreachable (`ssh` timeout). Not built, no Logos/clang output yet.

## 5. Artifact Integrity

PENDING — no `.deb` produced yet.

## 6. Impact & Regression Risks

- SpringBoard/keyboard hosts: filter unchanged (UIKit everywhere) — same injection footprint as 2.0.
- Single-tap delete: expected 1 char (was 2). If 0 chars appear on-device, the system path assumption is wrong for that app — report immediately (would mean re-adding a guarded manual call).
- Hold-delete repeat: expected unchanged continuous delete. Watch for: repeat stopping after 1 char (would indicate the removed block was load-bearing — unlikely per guard analysis).
- Swipe-cursor / shift-select / more-key suppression: untouched code paths, but verify cursor movement + 2-finger word skip still work since the file was retranscribed.
- On-device observation: type `abcd`, tap backspace once → expect `abc`; hold backspace → continuous delete; swipe on keyboard → cursor moves, no characters deleted.
