[![Build Status](https://dev.azure.com/beninifulvio/beninifulvio/_apis/build/status/benini.scid?branchName=github)](https://dev.azure.com/beninifulvio/beninifulvio/_build/latest?definitionId=1&branchName=github)
[![codecov](https://codecov.io/gh/benini/scid/branch/github/graph/badge.svg)](https://codecov.io/gh/benini/scid)
[![coverity](https://scan.coverity.com/projects/14455/badge.svg)](https://scan.coverity.com/projects/benini-scid)
[![Documentation](https://img.shields.io/badge/docs-doxygen-blue.svg)](http://scid.sourceforge.net/doxygen/html/files.html)
[![GitHub license](https://img.shields.io/badge/license-GPL-blue.svg)](https://sourceforge.net/p/scid/code/ci/master/tree/COPYING)

Scid (Shane's Chess Information Database) is a multi-platform (Linux, Mac OS X, Windows) chess database application.

With Scid you can maintain a database of chess games, search games by many criteria, view graphical trends, and produce printable reports on players and openings. You can also analyze games with the Xboard or UCI compatible chess program, play online on FICS, and even use Scid to study endings with endgame tablebases.

Scid is free software and is released under the GPL licence.
In Linux and macOS, the build process is straightforward:

```bash
git clone --depth=1 https://git.code.sf.net/p/scid/code scid-code
cd scid-code
./build_app.sh
make install
```

The `Scid.app` folder contains the app, which can be moved to other directories, such as `/Applications`. It is also possible to create a symbolic link to the executable `Scid.app/Contents/scid/scid`.

## Building on Windows (Visual Studio 2022/2026)

### Prerequisites

- **Visual Studio 2022 or 2026** with the *Desktop development with C++* workload
- **Tcl/Tk 9.0** — install via Chocolatey or download from [tcl.tk](https://www.tcl.tk/software/tcltk/):
  ```
  choco install magicsplat-tcl-tk
  ```
  Default install path: `C:\Program Files\Tcl`

### Build

1. Clone the repository and open the folder in Visual Studio (File → Open → Folder).
2. Visual Studio detects `CMakePresets.json` automatically.
3. Select the **x64 Debug** or **x64 Release** configure preset from the toolbar.
4. Build → Build All (`Ctrl+Shift+B`).

The output executable is placed in `out\build\<preset>\scid.exe`.

> **VS 2026 note:** A regression in VS 2026 causes `cmake` configure to fail when using the
> Ninja generator if `vcvars64` is not activated. `CMakePresets.json` in this repo includes a
> workaround via `$penv{VCToolsInstallDir}` / `$penv{WindowsSdkDir}` macros.
> See [`BUG_REPORT_VS2026.md`](BUG_REPORT_VS2026.md) for details.

### Run / Debug from Visual Studio

The startup configuration is in `.vs/launch.vs.json` (tracked in git).
It sets:

| Setting | Value |
|---------|-------|
| Startup project | `scid.exe` |
| Command arguments | `tcl/start.tcl` |
| Working directory | repository root |
| PATH prepend | `C:\Program Files\Tcl\bin` (Tcl 9.0 runtime DLL) |

To run: set **scid.exe** as the startup item in the toolbar, then press **F5**.

### nmake (legacy)

`Makefile.vc` provides an alternative nmake-based build from a Developer Command Prompt:

```bat
nmake -f Makefile.vc release TCL_DIR="C:\Program Files\Tcl"
```

Key `Makefile.vc` settings (do not need to change for a default Tcl 9.0 install):

| Macro | Default | Description |
|-------|---------|-------------|
| `TCL_DIR` | `C:\Program Files\Tcl` | Tcl/Tk installation root |
| `TCL_VERSION` | `90` | Tcl version (`86` = 8.6, `90` = 9.0) |
| `DEBUG` | _(unset)_ | Set to `1` for a debug build |

Please report issues and bugs here:
https://sourceforge.net/projects/scid/  
For other problems or support, try reaching out to the mailing list:
https://sourceforge.net/p/scid/mailman/
