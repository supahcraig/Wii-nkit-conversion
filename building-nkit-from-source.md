# Building NKit from source (Mac)

This is a fallback path, not the primary recommendation — see the main
[NKit-to-WBFS-HOWTO.md](./NKit-to-WBFS-HOWTO.md) for the normal way to get
NKit (a pre-built zip). Build from source if every pre-built source is
unreachable, or you specifically want to inspect/modify NKit's behavior.

**Heads up**: the source repo (`Nanook/NKitv1` on GitHub) is a few years
behind what's actually distributed as pre-built binaries. Building it
produces a working `v1.3` — the swiss-gc/vimm.net zips are `v1.4`. Same core
behavior (including the `OutputLevel=3` fix documented in the main HOWTO),
just an older revision.

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
