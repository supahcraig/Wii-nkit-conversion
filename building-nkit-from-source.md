# Building NKit from source (Mac)

This is a fallback path, not the primary recommendation — see the main
[NKit-to-WBFS-HOWTO.md](./NKit-to-WBFS-HOWTO.md) for the normal way to get
NKit (a pre-built zip). Build from source if every pre-built source is
unreachable, or you specifically want to inspect/modify NKit's behavior.

## ⚠️ There are TWO different "NKit" repos by the same author — do not confuse them

- **[`Nanook/NKitv1`](https://github.com/Nanook/NKitv1)** — the OLD tool.
  Separate `.exe` per task (`ConvertToISO.exe`, `ConvertToNKit.exe`, etc.),
  banner reads `"ConvertToISO vX.X, NKit.dll vX.X :: Nanook"`. **This is what
  the main HOWTO uses, and what this guide builds.** Builds with Mono, no
  `dotnet` needed. Currently tagged `1.3` in this repo — a few versions
  behind the `1.4` distributed as pre-built binaries (swiss-gc/vimm.net),
  but same core behavior (including the `OutputLevel=3` fix).

- **[`Nanook/NKit`](https://github.com/Nanook/NKit)** — a completely
  different, unrelated rewrite the author calls **"NKit 2"**. One unified
  binary with a different CLI (`nkit -task convert ...`), **requires the
  `dotnet` 6 runtime**, and is **not compatible** with anything in the main
  HOWTO — it doesn't produce `ConvertToISO.exe` or anything like it.

**If you installed `dotnet` 6 trying to build NKit, you were building the
wrong one for this pipeline.** This guide is entirely about `NKitv1`.
There's also no public source exactly matching the `1.4` binaries
themselves — Nanook ships those as compiled binaries only (via swiss-gc,
vimm.net, etc.), without a matching public tag/release on `NKitv1`. Building
`NKitv1` gets you the newest version whose *source* is public, `1.3`, not a
perfect match for `1.4`.

## Dependencies

**Just Mono** — despite this being a .NET project, you do **not** need the
`dotnet` SDK:

```
brew install mono
```

Why not `dotnet`: this is an old-style project (`ToolsVersion="15.0"`,
targeting classic **.NET Framework 4.6.1**), not a modern SDK-style project.
The cross-platform `dotnet` CLI doesn't ship .NET Framework 4.6.1 reference
assemblies, so `dotnet build` can't build it. Mono does ship those reference
assemblies, plus its own MSBuild-workalike (`xbuild`) — installing Mono alone
gets you everything needed to both build and run these tools.

## Get the source

```
git clone https://github.com/Nanook/NKitv1.git
cd NKitv1
```

## Build

```
xbuild NKit.sln /p:Configuration=Release
```

No NuGet restore step needed — the project has no external package
references. This builds all 7 projects (the `NKit` library, the 5 console
tools, and 2 WinForms GUI apps) in one pass. Expect one harmless warning
(`Don't know how to handle GlobalSection ExtensibilityGlobals, Ignoring`) —
that's fine, the build still succeeds.

Each project's compiled output lands in its own `<Project>/bin/Release/`
folder, e.g. `ConvertToISO/bin/Release/ConvertToISO.exe`.

## Run it

The build does **not** copy `NKit.dll.config` into each tool's output
folder automatically. Copy it over from the `NKit` library project first:

```
cp NKit/bin/Release/NKit.dll.config ConvertToISO/bin/Release/
cd ConvertToISO/bin/Release
mono ConvertToISO.exe "/path/to/Game.nkit.iso"
```

(Swap `ConvertToISO` for whichever tool's folder you're using —
`ConvertToNKit`, `RecoverToISO`, `RecoverToNKit`, `RecoveryExtract`,
`NKitProcessingApp`, `NKitExtractionApp` — same pattern.)

From here, the rest of the main HOWTO applies as normal (the `OutputLevel=3`
fix, `wit-thin --psel=data`, etc.) — this just swaps out where `NKit.dll`
itself came from.
