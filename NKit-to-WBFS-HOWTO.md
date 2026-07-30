# NKit .nkit.iso → playable ISO / WBFS — working process

## The core issue, in short

A `.nkit.iso` is NKit's shrunk, preservation-oriented format for Wii discs —
it's not directly playable/bootable; it has to be reconstructed back into a
real ISO first. That reconstruction runs into one recurring snag:

- Most `.nkit.iso` files have their Wii **system update partition**
  (Nintendo's disc-embedded OS updater, not game content) stripped out
  entirely to save space.
- Rebuilding that partition byte-for-byte needs recovery data NKit doesn't
  distribute (it's Nintendo's copyrighted software), so a fresh install won't
  have it.
- NKit's own verification step correctly notices it can't rebuild an exact
  original — and by default just **deletes the output** rather than hand you
  an "unverified" file, even though the actual game data converted fine.

How we got around it:

1. Use an NKit distribution with a populated recovery/database set, for
   correct game identification.
2. Flip an undocumented debug setting so NKit keeps the unverified output
   instead of deleting it.
3. Use `wit` to explicitly drop that now-unrecoverable (filler-filled,
   invalid) update partition when packaging to WBFS — which is also
   *required* for real Wii hardware to accept the disc at all, since real
   hardware/`wit` reject a present-but-invalid update partition even though
   Dolphin doesn't care.

End result: a fully game-complete, real-hardware-bootable backup. The only
thing missing is Nintendo's update installer, which the game doesn't need.

This documents the exact, verified-working process for converting a `.nkit.iso`
(Wii, scrubbed/compressed NKit format) into a real ISO and then into a
split `.wbfs` that USB Loader GX will actually list and boot on real hardware.

Confirmed working end-to-end on: Guitar Hero III: Legends of Rock, The Beatles: Rock Band.

## Tools required

- **Mono** — required to run NKit's `.exe` tools on Mac/Linux (they're .NET
  binaries). Not installed on macOS by default:
  ```
  brew install mono
  ```

- **NKit + Wii partition recovery data** — get the **"NKit 1.4 + Wii Partitions"**
  bundle from **https://vimm.net/vault/?p=nkit** (not bare NKit — this bundle
  includes the Redump database and recovery files NKit needs to identify and
  rebuild discs). It downloads as `NKit 1.4 + Wii Partitions.7z`; unzip it —
  it extracts into a folder named `NKit`. `cd` into that folder to run the tools:
  ```
  cd NKit
  mono ConvertToISO.exe ...
  ```

- **wit / wit-thin** (Wiimms ISO Tools) — used for the final WBFS packaging,
  update-partition stripping, and FAT32 splitting.

  Get the official tools from **https://wit.wiimm.de/** (download page links
  to Windows/Mac/Linux builds). The build used here was **`wit-v3.05a-r8638-mac`**.

  `wit-thin` is **not** an official pre-built variant — on Apple Silicon Macs,
  the official universal binary (`bin/wit`) can get killed outright by
  Gatekeeper/code-signing checks before it even runs. The fix is to extract
  just your Mac's own architecture slice with `lipo -thin` and apply a fresh
  local ad-hoc signature — producing a binary literally named `wit-thin` for
  that reason. Full step-by-step build instructions, troubleshooting, and
  day-to-day usage are in
  **[wit-mac-setup-guide.md](./wit-mac-setup-guide.md)** — read that first if
  `wit` won't run for you at all (e.g. silently killed, "zsh: killed", or a
  code-signing error). Once built, it accepts the same `copy`/`--wbfs`/
  `--split`/`--psel` options used below. It won't be on your PATH — invoke it
  with the full path to wherever you extracted it, or `cd` there first and
  run `./wit-thin`.

## Step 1 — Convert `.nkit.iso` to `.iso` with NKit

```
cd NKit
mono ConvertToISO.exe "/path/to/Game.nkit.iso"
```

Requires ~4.5GB of free disk space for the output (Wii ISOs are a fixed ~4.37GiB
regardless of how compressed the source `.nkit.iso` is). If you get a
"Disk full" error, free up space and re-run.

### Known issue: missing Wii update-partition recovery data

Most `.nkit.iso` files have their Wii **system update partition** stripped out
(that's what makes NKit format smaller). To rebuild it byte-perfectly, NKit
needs the original partition's data in `Recovery/NkitExtracted/Wii/` — which is
**not distributed** (it's Nintendo's copyrighted update software, only ever
present if you've personally extracted it before). Without it you'll see:

```
!! Update partition *_XXXXXXXX missing - Adding filler. It may be Recoverable
...
Verification Failed Crc:XXXXXXXX - Failed Test Crc:XXXXXXXX
Deleting Output
```

NKit fills the missing partition with zero/filler data, then its own
verification step (rightly) flags that as not matching the Redump-verified
original, and **by default deletes the output entirely** rather than hand you
an unverified file. This is hardcoded behavior (confirmed by reading the actual
NKit source, `Converter.cs`), **not** controlled by the documented `FullVerify`
or `RecoveryMatchFailDelete` config keys — those do nothing here.

The undocumented fix: set `OutputLevel` to `3` (an undocumented "Debug" level;
the config comment only documents 0–2) in `NKit.dll.config`:

```xml
<add key="OutputLevel" value="3"/>
```

This makes the delete-on-verify-fail step get skipped (logged as
`"Deleting Output (Skipped as OutputLevel is 3:Debug)"`). The output survives,
but as a raw temp file — it does NOT get renamed/moved to its normal output
location, because that also only happens on the success path. Find it here:

```
NKit/Processed/Temp/tmp########.tmp
```

...and move/rename it yourself to a proper `.iso` filename.

**Practical effect of the filler partition**: the resulting ISO plays fine in
Dolphin, and is fine for real hardware *once the update partition is stripped
entirely* (see Step 2) — real Wii firmware and `wit`'s own strict partition
loader will reject a *present-but-invalid* update partition (zero-filled TMD),
throwing an error like:

```
!! ERROR in wd_load_part() @ src/libwbfs/wiidisc.c#2105
!! Invalid TMD size (0x0) in partition '1 [UPDATE]'
```

Hence Step 2 always uses `--psel=data` to drop that partition rather than
trying to carry a broken one.

## Step 2 — Convert `.iso` to split `.wbfs` with wit-thin, stripping the update partition

```
wit-thin copy --wbfs --split --psel=data \
  "/path/to/Game.iso" \
  "~/Wii Game Roms/wbfs"
```

- `--wbfs` — output format
- `--split` — split output over 4GiB into `.wbfs` + `.wbf1` (+`.wbf2`...) parts,
  required for FAT32 SD cards (4 GiB / 4,294,967,295-byte file size limit)
- `--psel=data` — **only copy the DATA partition**, dropping the (filler-only,
  invalid) UPDATE partition entirely. This is what actually makes the disc
  bootable on real hardware / loadable by `wit` itself without error. The
  update partition is Nintendo's system-update installer, not game content —
  dropping it doesn't affect gameplay, it's standard practice for USB-loader
  Wii backups.

wit-thin auto-creates a subfolder named `<Title> [<ID6>]/` and writes
`<ID6>.wbfs` (+ `.wbf1` etc.) inside it, matching the standard WBFS Manager /
USB Loader GX / WiiFlow convention.

## Step 3 — Verify the game ID (don't just trust the folder name)

Cross-check against local databases before trusting a folder/file name blindly:

```
grep -i "<game title>" "NKit/Dats/GameTdb/wiitdb.txt"
grep -i -B2 -A2 "<game title>" "NKit/Dats/Redump/Wii/Redump.dat"
```

And/or check the ID is actually embedded in the produced file itself:

```
strings -a "Game.wbfs" | grep -i "<part of title>"
grep -a -o "XXXXXX" "Game.wbfs"   # the expected ID6
```

## Step 4 — Copy to the SD card

Copy the whole per-game folder (all split parts together, same base filename)
straight into the SD card's `wbfs/` directory:

```
cp -R "~/Wii Game Roms/wbfs/<Title> [<ID6>]" "/Volumes/<SDCARD>/wbfs/"
```

Split parts (`.wbfs`, `.wbf1`, `.wbf2`...) must:
- share the exact same base filename (differing only by extension), and
- sit together in the same folder.

USB Loader GX / WiiFlow join them back into one virtual disc image
automatically — no merging needed.

## Known gotcha #1: filenames must be `<ID6>.wbfs`, not the title

If a `.wbfs` you already have (not run through this pipeline — e.g. downloaded
pre-made) is named with the game title instead of its ID6
(`Guitar Hero - Metallica [SXBE52].wbfs` instead of `SXBE52.wbfs`), USB Loader
GX may fail to even recognize it as a valid game, since it matches split parts
by shared base filename and expects the ID6 convention. Rename all parts to
`<ID6>.wbfs` / `<ID6>.wbf1` / etc. to match every other working title.

## Known gotcha #2: stale game-list cache

USB Loader GX caches its parsed game list in
`apps/usbloader_gx/TitlesCache.bin` on the SD card, and doesn't always rescan
automatically when a new title is dropped in — a perfectly valid game can
simply fail to appear in the carousel. Fixes, easiest first:
- Rename the game's folder (forces the loader to treat it as new) — confirmed
  to work.
- Delete/rename `TitlesCache.bin` to force a full rebuild on next launch.
- Use USB Loader GX's own "Update Cache" option on the Wii itself.

## Summary pipeline

```
Game.nkit.iso
   │  mono ConvertToISO.exe   ("NKit 1.4 + Wii Partitions" bundle, OutputLevel=3)
   ▼
Game.iso                       (update partition = filler, plays in Dolphin)
   │  wit-thin copy --wbfs --split --psel=data
   ▼
<Title> [ID6]/ID6.wbfs (+ .wbf1...)   (update partition dropped, real-hardware bootable)
   │  cp -R  →  SD card /wbfs/
   ▼
Playable in USB Loader GX
```

If a `.wbfs` is already in hand (no `.nkit.iso` involved), skip straight to
`wit-thin copy --wbfs --split` (no `--psel` needed unless it still carries an
update partition) — no NKit step required.
