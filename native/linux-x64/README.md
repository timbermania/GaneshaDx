# native/linux-x64/libnfd.so

The `NativeFileDialogExtendedSharp` NuGet package (0.1.0) ships only the
Windows `nfd.dll` — no Linux native library — so the file Open/Save dialogs
(`Nfd.FileOpen` / `Nfd.FileSave`) crash on Linux with a missing-library error.

This is the Linux build of [nativefiledialog-extended](https://github.com/btzy/nativefiledialog-extended),
**portal backend** (uses xdg-desktop-portal, so it shows the desktop's native
file chooser). Built with:

    g++ -shared -fPIC -O2 -o libnfd.so src/nfd_portal.cpp -Isrc -Isrc/include \
        $(pkg-config --cflags --libs dbus-1)

`GaneshaDx.csproj` copies it next to the app output on Unix builds.
