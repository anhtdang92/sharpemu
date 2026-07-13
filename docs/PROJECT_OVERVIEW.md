<!--
Copyright (C) 2026 SharpEmu Emulator Project
SPDX-License-Identifier: GPL-2.0-or-later
-->

# SharpEmu — Project Overview & Testing Guide

This document captures a walkthrough of the SharpEmu codebase: what the project
is, how it is structured, how a game boots through it, its current
compatibility status, and — importantly — the practical and legal realities of
testing it.

It is intended as an orientation document for new contributors. It is not a
substitute for reading the code; file/line references are given so you can dig
in.

---

## 1. What SharpEmu Is

SharpEmu is an **experimental PlayStation 5 emulator written in C# (.NET 10)**.

- **Scope:** PS5 only. It deliberately does *not* target PS4 games — the README
  points at ShadPS4 for that platform.
- **Purpose:** Research and educational. No commercial goals. Focus is on
  accuracy and infrastructure, not per-game compatibility.
- **Primary target:** Windows first (see §3 for why). Linux/macOS are planned
  but not yet the focus.
- **License:** GPL-2.0-or-later, REUSE-compliant (`REUSE.toml`, `LICENSES/`).
- **Size:** ~74k lines of C# across 171 files, in 6 projects.

### Key architectural insight

The PS5 CPU is x86-64 (AMD Zen 2), and so is the typical host. Rather than
interpret or JIT-recompile guest code, SharpEmu's CPU core
(`DirectExecutionBackend`) **runs the game's native x86-64 instructions
directly on the host CPU**. It maps guest code into host memory and uses
structured exception handling (SEH) to trap out when the game invokes a PS5
system function; those traps are routed to C# reimplementations of the system
libraries (High-Level Emulation, "HLE").

This is why the CLI works so hard to relaunch itself with CPU security
mitigations disabled (`TryRunMitigatedChild` in `src/SharpEmu.CLI/Program.cs`):
CET shadow stacks and Control Flow Guard would otherwise reject executing
foreign guest code. It is also why the emulation core is effectively
**Windows-only** today.

---

## 2. Project Layout

The solution (`SharpEmu.slnx`) contains six projects under `src/`:

| Project | Lines | Role |
|---|---:|---|
| **SharpEmu.CLI** | ~830 | Entry point. Builds as `SharpEmu.exe` (GUI subsystem). No args → opens the GUI; with an `eboot.bin` argument → attaches a console and runs headless. Handles the mitigation-disabling relaunch. |
| **SharpEmu.Core** | ~15.6k | The engine: SELF/ELF **loader**, **virtual memory**, and the **CPU** subsystem (`DirectExecutionBackend` native execution, Iced-based disassembly for diagnostics, dispatcher, stub manager). `SharpEmuRuntime` orchestrates the whole run. |
| **SharpEmu.HLE** | ~1.5k | HLE framework: the `[SysAbiExport]` attribute + `ModuleManager` that reflection-scans assemblies to register syscall implementations, keyed by PS5 NID hashes resolved via the embedded **aerolib** symbol database. Also `CpuContext`/registers and the guest-thread model. |
| **SharpEmu.Libs** | ~48.4k | Largest project — the reimplemented PS5 system libraries: Kernel (memory/pthread/runtime), **AGC** (the PS5 GPU, incl. a Gen5 shader→SPIR-V translator), **VideoOut** (Vulkan presenter via Silk.NET), Audio/Ngs2, Pad (DualSense HID + XInput), PlayGo, SaveData, Np, and many more. |
| **SharpEmu.GUI** | ~6.8k | Avalonia desktop frontend — game library, log console, Discord Rich Presence, and a full **ATRAC9** audio decoder. Launches the CLI as a child process. |
| **SharpEmu.Logging** | ~880 | Logging + `BuildInfo` provenance banner (git SHA/branch injected from CI). |

### SharpEmu.Libs subsystems (by folder)

`Agc` (GPU/shaders), `Ampr`, `AppContent`, `Audio`, `AvPlayer`,
`CommonDialog`, `DiscMap`, `Fiber`, `GameUpdate`, `Ime`, `Json`, `Kernel`,
`Mouse`, `Network`, `Ngs2`, `Np`, `NpGameIntent`, `Pad`, `PlayGo`, `Rtc`,
`SaveData`, `Share`, `SystemGesture`, `SystemService`, `Ult`, `UserService`,
`VideoOut`.

### Notable large files

- `Libs/VideoOut/VulkanVideoPresenter.cs` (~6.8k) — Vulkan present path
- `Libs/Kernel/KernelMemoryCompatExports.cs` (~6.6k)
- `Libs/Agc/AgcExports.cs` (~5.9k) — GPU command/export surface
- `Core/Cpu/Native/DirectExecutionBackend.cs` (~4.8k) — native execution core
- `Libs/Agc/Gen5SpirvTranslator*.cs` — PS5 GNM shader → SPIR-V/Vulkan translation

---

## 3. How a Game Boots

The orchestration lives in `SharpEmuRuntime.Run(ebootPath)`
(`src/SharpEmu.Core/Runtime/SharpEmuRuntime.cs`):

1. **Mount** the `eboot.bin`'s directory as `/app0` so the guest can read its
   own assets by that path.
2. **Load the image** — `SelfLoader` parses the SELF/ELF, maps segments into
   virtual memory, resolves import stubs and runtime symbols.
3. **Load adjacent modules** — nearby `.prx` / `sys_module` libraries are
   loaded and linked; imported data symbols are rebound.
4. **Run initializers** for all modules.
5. **Dispatch the entry point** — `DispatchEntry` begins native execution;
   every PS5 syscall traps to an HLE implementation.

### The HLE export mechanism

System functions are plain C# methods tagged with `[SysAbiExport]`
(`src/SharpEmu.HLE/SysAbiExportAttribute.cs`), specifying library name, NID,
export name, and target generation (Gen4/Gen5). At startup `ModuleManager`
reflection-scans the `Libs`/`Kernel` assemblies and registers every tagged
method. Incoming guest calls are matched by NID (hashed export name), resolved
through the embedded **aerolib** database (`SharpEmu.HLE/Aerolib`).

---

## 4. Tooling & Build

- **Dependencies:** Iced (x86 decode), Silk.NET.Vulkan (GPU/present), Avalonia
  (GUI). Central package management (`Directory.Packages.props`) with lock
  files; .NET SDK pinned to `10.0.103` (`global.json`).
- **CI:** `.github/workflows/workflow.yml` — REUSE license lint → build on
  `windows-latest` → publish a self-contained single-file `win-x64` release.
- **Build locally:**
  ```
  dotnet build SharpEmu.slnx
  ```
  or publish the runnable CLI:
  ```
  dotnet publish src/SharpEmu.CLI/SharpEmu.CLI.csproj -c Release -r win-x64 --self-contained true
  ```
  Artifacts land in `artifacts/`.
- **Run:**
  ```
  .\SharpEmu "eboot.bin" 2>&1 | Tee-Object -FilePath "log.txt"
  ```
  or launch `SharpEmu.exe` with no arguments for the GUI.
- **CLI flags:** `--strict`, `--trace-imports[=N]`, `--cpu-engine=native`,
  `--log-level=<level>`.

---

## 5. Current Compatibility Status

Early-stage. "Working" means *reaches a rendering/output milestone*, **not
playable**. Per the README's status list, the emulator can currently: load
`eboot.bin`/`.elf`, execute native CPU instructions, read game metadata, load
system modules, partially handle kernel functions, and reach the `sceVideoOut`
and AGC (GPU) stages on some titles.

Games tracked as tested (tracking issues live on the upstream repo
`par274/sharpemu`):

| Game | Title ID | State |
|---|---|---|
| **Demon's Souls Remake** | PPSA01341 | Furthest along — reaches a "video loop" / first `sceVideoOut` frame; shaders being converted to SPIR-V/Vulkan. Committed screenshot: `.github/images/des-videoout-shaders.jpg`. |
| **Dreaming Sarah** | PPSA02929 | Real texture rendering (splash texture). Committed screenshot: `.github/images/dreaming-sarah.jpg`. |
| **Poppy Playtime Chapter 1** | PPSA20591 | Boots / partial. |
| **SILENT HILL: The Short Message** | PPSA10112 | Boots / partial. |

The bottleneck between "first frame" and "playable" is the **AGC shader →
SPIR-V** pipeline (`Libs/Agc/Gen5SpirvTranslator*.cs`): games submit GPU work,
but turning submitted PS5 shaders into drawable Vulkan output is still in
progress.

---

## 6. The Input Contract — What File the Emulator Actually Loads

**SharpEmu loads a decrypted `eboot.bin` (a SELF or plain ELF).** That is the
only thing the loader accepts:

- `SharpEmuRuntime.Run(ebootPath)` takes a path to that one file.
- `SelfLoader` (`src/SharpEmu.Core/Loader/SelfLoader.cs`) validates it by
  checking for the **SELF magic** (`0x4F153D1D`) or a raw **ELF header** —
  nothing else.

But it expects that file inside the game's **unpacked folder**, because on
startup it mounts the directory as `/app0` and loads the adjacent `.prx`
modules and `sce_sys/` metadata:

```
<game folder>/
  eboot.bin        ← decrypted SELF/ELF; this exact path is passed to Run()
  sce_sys/          ← param.sfo, etc.
  *.prx             ← modules loaded alongside
  <assets…>         ← reached via /app0
```

### What SharpEmu does NOT read

There is **no `.pkg` handling anywhere in the codebase** (verified by search).
A retail PS5 title ships as encrypted `.pkg` package files (base + patch/DLC),
optionally with a `.crc` integrity manifest. The `.pkg` bodies are encrypted;
the `eboot.bin` lives *inside* the base package. The emulator has **no
unpacker, no PKG reader, and no decryptor** — it begins one layer *after* the
decrypt/extract step:

```
retail .pkg (encrypted container)   ← SharpEmu does NOT read this
        │  unpack + decrypt          ← NOT part of SharpEmu; see §7
        ▼
<game folder>/eboot.bin + assets    ← SharpEmu.Run() starts here
```

---

## 7. Testing — Legal & Practical Realities

There are two distinct things to test.

### 7a. Test that the emulator builds and runs (no game needed)

Build as in §4, then launch `SharpEmu.exe` with no arguments. The Avalonia GUI
opens, exercising the frontend, game-library UI, logging, and CLI/GUI plumbing.
If that works, your build is good.

### 7b. Test the emulation core (loader → CPU → HLE → render)

This needs a decrypted `eboot.bin`. The legal source of one for testing is
**PS5 homebrew**, not a commercial game:

- Homebrew `.elf`/eboot binaries are **unencrypted** and freely, legally
  available.
- SharpEmu's loader accepts a raw ELF directly (it checks for the SELF magic
  *or* a plain ELF header — see §6).
- Running one drives the same early code paths a retail game hits: SELF/ELF
  parsing, virtual-memory mapping, module linking, native CPU execution, and
  HLE syscall dispatch.

```
.\SharpEmu "path\to\homebrew.elf" 2>&1 | Tee-Object -FilePath "log.txt"
```

When it stops or traps, `log.txt` shows where — that log is the starting point
for diagnosing and fixing loader/HLE/shader issues.

### Retail games and the decryption boundary

A retail game you legally own is still distributed as encrypted `.pkg` files.
Converting those into the unpacked `eboot.bin` folder that SharpEmu expects
requires **decrypting the console's protected content**. That decryption/
circumvention step is:

- **not implemented in SharpEmu** (it has no unpacker), and
- **outside the scope of this project and this documentation** — it is a
  copy-protection circumvention step, unlawful under anti-circumvention law
  (e.g. DMCA §1201) even for a game you legally own.

This document therefore does not cover PKG decryption, extraction tools, or
keys. Legal ownership of a game grants the right to play it on your own
console; it does not extend to breaking its encryption. Everything *downstream*
of a legitimately-produced `eboot.bin` is fair game for development and testing.

> The project itself is explicit on this point: the README states SharpEmu
> "does **not** support or condone piracy," that all test games were "dumped
> from consoles that we personally own," and that "users are expected to use
> legally obtained copies of their games." No game data ships with the repo.

---

## 8. Where Contribution Is Most Useful

Given the current state, the highest-value work (all doable from a boot log,
no retail game required — homebrew or a synthetic eboot suffices to reproduce
most early failures):

1. **Read a boot/trace log** and pinpoint where execution stalls — an
   unimplemented HLE import, a wrong kernel return, or a CPU trap.
2. **Implement missing HLE functions** in `SharpEmu.Libs` — often well-scoped
   additions keyed by a specific NID the log names as unimplemented.
3. **Advance the AGC shader→SPIR-V pipeline** (`Libs/Agc/Gen5SpirvTranslator*`)
   — the actual bottleneck between "first frame" and "rendering." Hardest area,
   biggest payoff.
4. **Fix loader/kernel/memory bugs** surfaced by traces.

### Environment note

The direct-execution CPU core is Windows-only, so the run/test loop happens on
a Windows machine. Code work can happen anywhere; the actual execution and log
capture must be on Windows.

---

## Appendix: Quick Reference

- **Solution:** `SharpEmu.slnx`
- **Entry point:** `src/SharpEmu.CLI/Program.cs`
- **Run orchestration:** `src/SharpEmu.Core/Runtime/SharpEmuRuntime.cs` → `Run()`
- **Loader:** `src/SharpEmu.Core/Loader/SelfLoader.cs` (SELF magic `0x4F153D1D`)
- **CPU core:** `src/SharpEmu.Core/Cpu/Native/DirectExecutionBackend.cs`
- **HLE export attribute:** `src/SharpEmu.HLE/SysAbiExportAttribute.cs`
- **Symbol DB:** `src/SharpEmu.HLE/Aerolib/`
- **GPU/shaders:** `src/SharpEmu.Libs/Agc/`
- **Vulkan present:** `src/SharpEmu.Libs/VideoOut/VulkanVideoPresenter.cs`
- **CI:** `.github/workflows/workflow.yml`
- **Input the emulator loads:** a decrypted `eboot.bin` (SELF/ELF) inside an
  unpacked game folder. `.pkg` files are **not** supported.
