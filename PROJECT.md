# PROJECT.md — swipeselection

## Identity

- Package: `com.icraze.swipeselection` (SwipeSelection, iCraze 2.0 rebuild of Kyle Howells' open-source tweak)
- Device: iPhone 11 / iOS 18.7.1 / Relaxin + RootHide, arm64e
- Original: `com.icraze.swipeselection_2.0_iphoneos-arm64e.deb` (root, immutable reference)

## Working Directory

`D:\llama agent\tweak\swipeselection\src`

## Current State (2026-10-15)

- `src/control` Version: **2.0.1** (was 2.0). One functional fix, patch increment.
- Bug: tap on backspace deletes exactly 2 chars on iOS 18.7.1.
- Cause: legacy manual `[kb handleDelete]` re-fire in `touchesEnded` duplicates the system's own tap-delete (modern layout/`handleDeleteWithNonZeroInputCount` path). Fix = removed the re-fire.
- `src/` built from verified original source (`kylehowells/SwipeSelection`, `Tweak.mm`) with the single fix. iCraze 2.0's extra word-delete-counter logic NOT carried over (no source, not reinvented) — verify on-device.
- Build: PENDING — macOS VM (`csw@192.168.1.21`) unreachable this session (`ssh` timeout). Run `make package` once VM is up.
- Details: `docs/double-delete-fix.md` (6-section report).

## Layout

- `extracted/`: original 2.0 dylib + plist + control (reference only)
- `src/`: `Tweak.x`, `Makefile`, `control`, `SwipeSelection.plist`
- `temp/`: deb split + disasm (`disasm.txt`, `disasm_ann.txt`, `stubmap.txt`, `selrefs.txt`, `strings.txt`) — disposable
- `docs/`: verified findings
