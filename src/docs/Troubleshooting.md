---
layout: default
is_doc: true
title: Troubleshooting
permalink: /docs/Troubleshooting.html
base: ../
---

# PathForge — Troubleshooting

Most "PathForge isn't working" reports are about the **native plugin not loading**
or **loading the wrong one**. PathForge is built to tell you exactly what happened:
the moment the native layer fails, `DnskSolver` throws with a diagnostic string, and
`DnskNative.Diagnose()` produces a full report. Start there.

## First thing to check

```csharp
if (!DnskNative.Available)
{
    Debug.LogError(DnskNative.Diagnose()); // full report: what it looked for, what it found
}
```

`Diagnose()` reports, per platform, which DLL name it expected, every search path it
tried, and the OS error code from the load attempt. That output tells you which of
the cases below applies.

---

## `DllNotFoundException` — the plugin won't load

The native library wasn't found at all. In order of likelihood:

1. **Wrong platform binary is enabled.** Each binary is tagged to one platform/CPU
   via its Unity Plugin Importer (see below). If you're on Windows but the Windows
   `.dll` is disabled, you get this. Open the file in the Project window → Inspector →
   **Platforms**, and make sure the row for your build target is checked.
2. **You're building for a platform you don't ship for.** PathForge 1.0.0 ships
   Windows (x86_64), macOS (x86_64 + arm64), Linux (x86_64), and Android (ARM64).
   There is no iOS, Windows 32-bit, Windows ARM64, or WebGL binary in this
   release — building for one will fail. See [Known Limitations](#known-limitations).
3. **Windows: missing or mismatched C runtime.** The PathForge DLL does not require
   the VC++ Redistributable. If you still see this on Windows, the `Diagnose()`
   report will show the Win32 error:
   - `126` — module not found (the .dll isn't where Unity looks).
   - `193` — the .dll is for the wrong CPU (32-bit app loading a 64-bit DLL).
   - `14001` / `127` — missing C runtime (should not happen with PathForge;
     if it does, report it).
4. **The binary is in the wrong folder.** If you copied the native library out of
   the package, it won't be found. Keep it in the package's `Runtime/Plugins/`
   folder.

## `EntryPointNotFoundException` — a function is missing

The DLL loaded, but a specific symbol isn't there. Causes:

1. **Mixed versions.** The C# binding and the native binary are from different
   releases. A binary from an older/newer build lacks the entry point the current
   C# expects. Re-install the package so the C# and all binaries are the same
   version. (PathForge checks the ABI version at construction — a mismatch throws a
   clear "ABI mismatch" error before you hit this.)
2. **Wrong binary for the CPU.** A 32-bit entry point missing from a 64-bit DLL, or
   vice versa. Same fix as above — enable the binary matching your build target.

## Wrong CPU architecture

Symptoms: `193` on Windows, "bad CPU type" on macOS, or a crash on load.

- **Windows:** make sure your build target is 64-bit. PathForge ships x86_64 only
  (no 32-bit, no ARM64 Windows).
- **macOS:** both Intel (`.x86_64.dylib`) and Apple Silicon (`.arm64.dylib`) are
  included and tagged correctly — Unity picks the right one automatically. If you
  see a mismatch, check the Plugin Importer on both files.
- **Linux:** x86_64 (`libdnsk_ffi.so`) and ARM64 (`libdnsk_ffi.aarch64.so`) are both
  included and tagged.

## Plugin Importer misconfiguration

If you edited the binaries' import settings, restore them. Each file must have:

- **Any** platform: **disabled**.
- The **one** matching platform (Windows / macOS / Linux) + the right CPU: **enabled**.
- **Editor**: enabled (so the samples work in the editor).

You can re-import the original files from the package to reset them. If you
**renamed** a binary, its import settings break — don't rename the shipped files.

## Apple Silicon (M1/M2/M3)

No action needed — both `.dylib` variants ship and are tagged. If a project
previously saved on an Intel Mac is opened on Apple Silicon, force a re-import
(Window → Package Manager → select PathForge → right-click → Reimport), then verify
the arm64 `.dylib`'s Platform row is checked.

## Android / iOS

**Android (ARM64): included.** The Android `.so` is tagged to the Android/ARM64
platform. On Android, Unity loads the `.so` from the plugin automatically, so no
extra C# code is needed — `DllImport("dnsk_ffi")` resolves it. Make sure the
Android binary's Plugin Importer row for *Android / ARM64* is enabled.

**iOS: not supported.** There is no iOS static library in the package, and iOS
requires a static build (`DllImport("__Internal")`) that isn't part of this
release. Building for iOS will fail to link. See [Known Limitations](#known-limitations).

## Linux native dependencies

The Linux `.so` links only against the standard C library — no third-party shared
libraries. If you get an "undefined symbol" or "cannot open shared object" error
on Linux, you have a mismatched binary — re-install the package.

## Editor restart / stale plugin

After upgrading PathForge, or if the editor loads a cached copy of the native
library:

1. Close Unity.
2. Delete the project's `Library/` folder (safe to delete; Unity rebuilds it).
3. Reopen the project.

This forces the plugin to re-load from the package. If you changed the binary
files directly, a full editor restart is required — Unity does not hot-reload
native plugins.

## ABI mismatch

If you accidentally mix a C# binding with a native binary from a different
PathForge version, the constructor fails fast with a clear "ABI mismatch" error.
This is intentional — a half-compatible binary would otherwise return silent
garbage. Fix: make sure the C# code and all native binaries come from the same
package version (re-install the package).

---

## Known Limitations

- **Platforms:** Windows x86_64, macOS x86_64 + arm64, Linux x86_64 + arm64, and
  Android ARM64. **iOS is not supported** (needs a static build not in this
  release); nor are Windows 32-bit, Windows ARM64, or WebGL.
- **No incremental graph updates.** When the graph changes, dispose the solver and
  rebuild. (The `SupplyRun` sample shows the dual-solver pattern to make this cheap.)
- **Not local avoidance.** PathForge produces routes; it does not resolve
  agent-to-agent collisions.
- **Main-thread only.** All PathForge calls must run on Unity's main thread.
- **No Web / WebGL target.** There is no WASM build.

## Dependencies

PathForge requires only two **built-in** Unity modules — no third-party packages:

- `com.unity.modules.tilemap`
- `com.unity.modules.ai`

The native kernel has **no third-party runtime dependencies** (see
[Third Party Notices](../Third%20Party%20Notices.html)).

## Threading

- **Call PathForge from the main thread only.** It wraps Unity-facing state and the
  samples are written that way.
- The native kernel handle is **not thread-safe**. Do not share one `DnskSolver`
  across threads. If you need parallelism, use one solver per thread.
- There are **no callbacks and no background threads** in PathForge — it is
  request/response only. Nothing runs after you stop calling it.

## Mono and IL2CPP

PathForge works with **both** script backends. All types crossing the FFI boundary
are blittable (primitive integers, pointers, sequential-layout structs); there is no
`Reflection.Emit` or dynamic proxying, which is what breaks most plugins under
IL2CPP. The samples run identically under Mono and IL2CPP.
