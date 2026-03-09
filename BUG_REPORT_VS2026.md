# Bug Report: VS 2026 CMakePresets.json – `strategy: external` Does Not Activate vcvars64

**Product:** Visual Studio 2026 Enterprise v18.3.2  
**Component:** CMake / CMake Integration  
**Severity:** High — prevents all Ninja-generator C++ projects from configuring correctly in the IDE  
**Report URL:** https://developercommunity.visualstudio.com/

---

## Summary

In Visual Studio 2026, setting `"architecture": {"value": "x64", "strategy": "external"}` in
`CMakePresets.json` no longer causes the IDE to activate `vcvars64.bat` before running
`cmake --fresh` or `cmake --build`.  
As a result cmake's MSVC compiler auto-detection picks x86 SDK/CRT library paths from the VS
IDE process PATH, and the compiler test (`CMakeTestCCompiler.cmake:67`) fails with:

```
LINK : fatal error LNK1120: ... unresolved externals
LINK : warning LNK4272: library machine type 'x86' conflicts with target machine type 'x64'
```

This regression does **not** occur in Visual Studio 2022 (v17.x), where `strategy: external`
correctly activates `vcvars64.bat` and cmake finds x64 tools automatically.

---

## Environment

| Item | Value |
|------|-------|
| VS version | Visual Studio 2026 Enterprise 18.3.2 |
| MSVC toolset | 14.50.35717 (cl.exe 19.50.35725.0) |
| Windows SDK | 10.0.26100.0 |
| cmake (VS bundled) | 4.1.2-msvc8 |
| cmake (standalone PATH) | 4.2.3 |
| Generator | Ninja |
| OS | Windows 11 x64 |

---

## Steps to Reproduce

1. Create a C++ project with `CMakeLists.txt` and `CMakePresets.json`:

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "x64-debug",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/out/build/x64-debug",
      "architecture": { "value": "x64", "strategy": "external" },
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    }
  ]
}
```

2. Open the folder in Visual Studio 2026.
3. Select the `x64-debug` configure preset from the toolbar.
4. Observe the CMake configure output — cmake runs with x86 MSVC tools in PATH and fails.

**CMakeError.log will show:**

```
lib\x86\LIBCMT.lib : fatal error LNK1120: ...
LNK4272: library machine type 'x86' conflicts with target machine type 'x64'
```

---

## Expected Behavior

Visual Studio activates `vcvars64.bat` before running cmake when
`"strategy": "external"` is specified (as it did in VS 2022), so cmake finds the
correct x64 host/target tools, INCLUDE, and LIB paths automatically.

---

## Actual Behavior

VS 2026 runs cmake with only the VS IDE process environment (x86 MSVC tools in PATH).
cmake auto-detection selects the first `cl.exe` on PATH, which is the x86 host,
and resolves Windows SDK/CRT libs using x86 paths.

**Root cause observation:** VS 2026 **does** set `VCToolsInstallDir`, `WindowsSdkDir`, and
`WindowsSDKVersion` in its process environment (without vcvars), but does **not** set the
x64 `INCLUDE` / `LIB` / `PATH` variables that vcvars would normally provide.

---

## Workaround (CMakePresets.json)

Use `$penv{}` macros to manually construct the correct x64 INCLUDE/LIB paths and
point `CMAKE_C_COMPILER`/`CMAKE_CXX_COMPILER` at the x64 host cl.exe.
VS 2026 always sets `VCToolsInstallDir`, `WindowsSdkDir`, and `WindowsSDKVersion`
in its process env, so these macros resolve correctly regardless of toolset version.

```json
{
  "version": 3,
  "configurePresets": [
    {
      "name": "windows-base",
      "hidden": true,
      "generator": "Ninja",
      "cacheVariables": {
        "CMAKE_C_COMPILER":   "$penv{VCToolsInstallDir}bin\\Hostx64\\x64\\cl.exe",
        "CMAKE_CXX_COMPILER": "$penv{VCToolsInstallDir}bin\\Hostx64\\x64\\cl.exe"
      },
      "environment": {
        "INCLUDE": "$penv{VCToolsInstallDir}include;$penv{VCToolsInstallDir}ATLMFC\\include;$penv{WindowsSdkDir}include\\$penv{WindowsSDKVersion}ucrt;$penv{WindowsSdkDir}include\\$penv{WindowsSDKVersion}um;$penv{WindowsSdkDir}include\\$penv{WindowsSDKVersion}shared;$penv{WindowsSdkDir}include\\$penv{WindowsSDKVersion}winrt;$penv{WindowsSdkDir}include\\$penv{WindowsSDKVersion}cppwinrt",
        "LIB": "$penv{VCToolsInstallDir}ATLMFC\\lib\\x64;$penv{VCToolsInstallDir}lib\\x64;$penv{WindowsSdkDir}lib\\$penv{WindowsSDKVersion}ucrt\\x64;$penv{WindowsSdkDir}lib\\$penv{WindowsSDKVersion}um\\x64"
      }
    },
    {
      "name": "x64-debug",
      "inherits": "windows-base",
      "architecture": { "value": "x64", "strategy": "external" },
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug" }
    }
  ]
}
```

A full working example of this workaround can be found at:  
https://github.com/zbrad/scid-original/blob/vs2026-build/CMakePresets.json

---

## Additional Notes

- The bug is **specific to the VS IDE cmake integration**. Running `cmake --preset x64-debug`
  from a Developer Command Prompt (where vcvars is already active) works correctly.
- The cmake version mismatch between the VS-bundled cmake (4.1.2-msvc8) and a standalone
  PATH cmake (4.2.3) causes an unnecessary `--fresh` re-configure; this is a separate
  minor issue but can compound the diagnosis.
- The `$penv{}` workaround requires CMakePresets.json `version: 3` (cmake 3.21+).
