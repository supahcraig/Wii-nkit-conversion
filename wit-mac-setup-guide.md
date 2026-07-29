# Wiimms ISO Tools (`wit`) on Apple Silicon Mac — Setup & Usage Guide

## Why this is needed

Wii disc images (`.iso`) are often larger than **4GB**, which is the hard per-file
size limit on **FAT32** — the filesystem your SD card / USB drive needs to use for
the Wii to read it. You can't just `cp` a 4.5GB ISO onto a FAT32 card.

**Wiimms ISO Tools (`wit`)** solves this: it converts ISO → **WBFS** format and
automatically **splits** the output into multiple sub-4GB files (`.wbf1`, `.wbf2`, etc.)
that USB Loader GX / WiiFlow read back together as a single game.

## Why you can't just run the downloaded binary directly

The official Mac build from `wit.wiimm.de` is a **universal binary** (arm64 + x86_64).
On modern macOS (especially Apple Silicon), Gatekeeper/code-signing checks can reject
this binary outright with a "zsh: killed" error, and its bundled signature can be in a
state where even `codesign --remove-signature` throws an *"internal error in Code
Signing subsystem"*.

The fix is to **extract just your Mac's own architecture slice** from the universal
binary (`lipo -thin`) and then apply a **fresh, local ad-hoc signature** to that
thinned binary. This sidesteps whatever was wrong with the original universal
binary's signature entirely — hence "**wit-thin**."

## Step-by-step: building `wit-thin`

```bash
# 1. Download the Mac build from wit.wiimm.de, then extract it
cd ~/Downloads
tar -xzf wit-v3.05a-r8638-mac.tar.gz
cd wit-v3.05a-r8638-mac

# 2. Confirm your Mac's architecture
uname -m
#   -> arm64   (Apple Silicon: M1/M2/M3/M4)
#   -> x86_64  (Intel)

# 3. Extract just that architecture from the universal binary
#    (the actual executable lives under bin/wit in the extracted folder)
lipo -thin arm64 bin/wit -output wit-thin
#    (swap "arm64" for "x86_64" if that's what step 2 showed)

# 4. Clear the "downloaded from internet" quarantine flag
xattr -c wit-thin

# 5. Make it executable
chmod +x wit-thin

# 6. Apply a fresh local ad-hoc signature
codesign --force --sign - wit-thin

# 7. Verify it runs
./wit-thin --version
```

If step 7 prints a version string (e.g. `wit: Wiimms ISO Tool v3.05a r8638 mac/arm`),
you're set.

### Troubleshooting notes
- If `codesign --remove-signature` throws *"internal error in Code Signing
  subsystem,"* skip removal and go straight to `codesign --force --sign -` on the
  **thinned** binary — this generally succeeds where it fails on the original
  universal binary.
- If `lipo` errors with "can't open input file," double-check the binary's actual
  path — some archives nest it under `bin/wit` rather than the top-level folder.

## Everyday usage: converting/copying a game to your SD card

```bash
cd ~/Downloads/wit-v3.05a-r8638-mac

./wit-thin copy --wbfs --split ~/Downloads/YourGame.iso "/Volumes/YOURCARD/wbfs/"
```

- `--wbfs` — output WBFS format (what USB Loader GX / WiiFlow expect)
- `--split` — automatically splits output into sub-4GB files for FAT32 compatibility
- `wit` auto-detects the Game ID from the disc header and creates a correctly named
  folder under `/wbfs/` for you

### Verifying a file/folder is named correctly
`wit` reports the Game ID it actually reads from the disc header — worth checking
this against your folder/file naming, since a mismatch (even a single transposed
character) can cause a loader to crash on launch even though the file itself is fine:

```bash
./wit-thin verify "/Volumes/YOURCARD/wbfs/Game Name [GAMEID]/GAMEID.wbfs"
```

### A known limitation: NKit-format files
`wit` **cannot read or convert `.nkit.iso` files** — NKit is a different, separately
maintained compression/trimming format that `wit` can only detect, not process.
Restoring an NKit file to a usable ISO requires different, NKit-specific tooling
(a much deeper rabbit hole involving building older, unmaintained source code under
Mono — a story for another document).

## Quick reference

| Task | Command |
|---|---|
| Check version | `./wit-thin --version` |
| Convert + split ISO to SD card | `./wit-thin copy --wbfs --split <file.iso> "<path>/wbfs/"` |
| Verify a file's Game ID | `./wit-thin verify "<path/to/game.wbfs>"` |
| List disc info | `./wit-thin ls -l <file.iso>` |
