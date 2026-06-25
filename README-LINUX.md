# Building & running GaneshaDx on Linux

GaneshaDx is a .NET 6 / MonoGame **DesktopGL** (OpenGL) app, so it runs
natively on Linux — no Wine at runtime. The only Linux snag is the
**build-time shader compile**, which this fork works around.

## The problem

MonoGame's content pipeline compiles the HLSL effect
`Content/FFTPolygonShader.fx` with `MGFXC`, which needs Windows'
`d3dcompiler_47` (i.e. a Wine prefix on Linux). Everything else in the
content pipeline (fonts, textures) builds fine on Linux.

## The fix (this fork)

The compiled content is already committed under
`Content/bin/DesktopGL/Content/` (`*.xnb`), including the shader. So on
Linux/macOS we **skip the content pipeline and consume the prebuilt
`.xnb`** instead of recompiling it. Windows builds are unchanged.

`GaneshaDx.csproj` changes:

- `EnableMGCBItems=false` when `'$(OS)' == 'Unix'` — stops the MonoGame
  task from auto-globbing/building `Content.mgcb` (which would invoke the
  Wine-only shader compiler).
- A Unix-only `<Content>` item group copies the prebuilt
  `Content/bin/DesktopGL/Content/**` into the app's output `Content/`.
- The `<MonoGameContentReference>` is now Windows-only.
- `RollForward=Major` — net6.0 is end-of-life; this lets the app run on a
  newer installed runtime (net8/net10) without a net6 runtime present.

Also committed `Content/bin/DesktopGL/Content/OverlayFont.xnb`, which
upstream tracked the other prebuilt `.xnb` but missed (it's `bin/`-ignored;
force-added). Without it the spritefont is absent at runtime.

### File Open/Save dialogs (native file picker)

The `NativeFileDialogExtendedSharp` package (0.1.0) ships **only** the Windows
`nfd.dll` — no Linux native library. So the moment you click **Open** (or any
Save), `Nfd.FileOpen`/`Nfd.FileSave` P/Invokes a missing `nfd` library and the
app crashes.

Fix: a Linux build of
[nativefiledialog-extended](https://github.com/btzy/nativefiledialog-extended)
(portal backend — uses `xdg-desktop-portal`, so you get the desktop's native
file chooser) is vendored at `native/linux-x64/libnfd.so` and copied next to
the app output by `GaneshaDx.csproj` on Unix builds. See
`native/linux-x64/README.md` for the build command. This needs an
`xdg-desktop-portal` implementation running (standard on most desktops,
including Hyprland/Omarchy).

## Prerequisites (Arch)

```bash
# .NET SDK (any of net6/8/10 — RollForward handles the runtime).
# Already present on this box: dotnet 10.
```

Optional, only for **texture import/export** (the editor launches and edits
maps without it): the native **FreeImage** library, used by the
`FreeImage.Standard` package. It is no longer in Arch's official repos
(AUR `freeimage`). Install only if you need texture I/O.

## Build

```bash
cd vendor/GaneshaDx
dotnet build GaneshaDx.csproj -c Release
```

Note: build the **`.csproj`**, not the `.sln` — a bare `dotnet build` picks
the solution, and the EOL-net6 + content conditions resolve most cleanly
per-project.

Output: `bin/Release/net6.0/GaneshaDx` (native apphost) and `GaneshaDx.dll`.

## Run

```bash
./bin/Release/net6.0/GaneshaDx
```

RollForward is baked into `GaneshaDx.runtimeconfig.json`, so it runs on the
installed net10 runtime with no env var. A MonoGame DesktopGL window opens
(OpenGL via SDL2). Verified on .NET SDK 10 + system SDL2/OpenAL, including the
file Open/Save dialogs (see the native file picker note above).

## Rebuilding the shader (only if you edit FFTPolygonShader.fx)

The committed `FFTPolygonShader.xnb` is used as-is on Linux. If you change
the shader source you must recompile it where `MGFXC` works — on Windows,
or on Linux via MonoGame's `mgfxc_wine_setup.sh` (sets up a Wine prefix
with `d3dcompiler_47` and `MGFXC_WINE_PATH`) — then commit the new `.xnb`.
