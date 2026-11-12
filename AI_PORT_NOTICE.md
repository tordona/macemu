# Windows ARM64 Port Notice & Build Guide

## Overview

This branch (`aarch64`) introduces native **Windows on ARM64 (`aarch64-w64-mingw32`)** support to **BasiliskII** and **SheepShaver**, along with their respective GTK+ preference editors (`BasiliskIIGUI.exe` and `SheepShaverGUI.exe`).

The porting, modernization, and verification work in this branch was developed by [@tordona](https://github.com/tordona) using AI pair programming assistance from Google DeepMind's Antigravity coding agent.

---

## Technical Summary of Changes

### 1. Windows ARM64 Architecture Support
* **Instruction Skipping in Fault Handlers (`sigsegv.cpp`)**:
  Implemented `aarch64_skip_instruction()` under `_WIN32 && (_M_ARM64 || __aarch64__)` to safely advance `CONTEXT->Pc += 4` when handling memory faults.
* **Byte-Swap & Atomic Intrinsics (`sysdeps.h`)**:
  Replaced x86-specific inline assembly (`bswap`, `xchgl`) on ARM64 with compiler built-ins (`__builtin_bswap16`, `__builtin_bswap32`, `__builtin_bswap64`) and GCC/Clang standard atomic operations (`__atomic_exchange_n`).
* **Fast Unaligned Access (`vm.hpp` & `sysdeps.h`)**:
  Enabled `VM_CAN_ACCESS_UNALIGNED` and `UNALIGNED_PROFITABLE` for `__aarch64__`, which is essential for performant PowerPC and 68k emulation on ARMv8-A+.
* **Memory Bulk Addressing (`-DMEM_BULK`)**:
  Ensured `-DMEM_BULK` is configured uniformly across all translation units on 64-bit ARM to guarantee consistent struct offsets and memory allocation layouts between CPU engines and emulator glue.

### 2. Modern Toolchain & Standard Conformance
* **C++17 Compatibility**:
  * Removed the deprecated `register` storage class specifier from local variable declarations in `sysdeps.h`.
  * Replaced obsolete `std::unary_function` and `std::binary_function` (removed in C++17) in `nvmemfun.hpp` with backward- and forward-compatible template aliases (`nv_unary_function`, `nv_binary_function`).
  * Guarded x86 32-bit `__thiscall` fast member function pointers in `nvmemfun.hpp` to prevent calling-convention rejections on ARM64.
  * Added missing `#include <new>` in `video_sdl*.cpp` required by C++17 for `std::nothrow`.
* **C23 / GCC 14+ / Clang 16+ SLIRP Compatibility**:
  Added `-std=gnu89` to `SLIRP_CFLAGS` in `configure.ac` so modern compilers defaulting to C23 do not reject SLIRP's K&R-style C function definitions.
* **Preprocessed C++ Tables (`$(CXX) -E`)**:
  Updated the `ppc-execute-impl.cpp` generation rule in `Makefile.in` to invoke `$(CXX) -E` rather than `$(CPP)`, preventing C23 compiler driver errors when preprocessing C++ source code.

### 3. 64-bit LLP64 Windows Pointer Safety & Subsystem Ports
* **64-bit Clipboard / Scrap Subsystem Port (`clip_windows.cpp`)**:
  Ported the modern 64-bit scrap handling logic from `clip_unix.cpp` and `clip_macosx.cpp`. Replaced legacy 32-bit host-stack stub execution with native Mac-space memory allocations (`NewPtrSysClear` trap `0xa71e`, `Host2Mac_memcpy`, and `DisposePtr` `0xa01f`), eliminating crashes caused by 64-bit pointer truncation when copying/pasting between Mac OS and the Windows host.
* **ExtFS / POSIX Emulation 32-bit Virtual Memory (`posix_emu.cpp`)**:
  Fixed an address translation failure where static host BSS memory passed to `Host2MacAddr` lay outside the 32-bit Mac address space on 64-bit Windows. ExtFS volume/file name blocks are now acquired in 32-bit addressable memory via `vm_acquire(..., VM_MAP_DEFAULT | VM_MAP_32BIT)` and freed with `vm_release()`, ensuring the "My Computer" host drive mount functions reliably.
* **Scratch Memory & DR Cache Safety (`main_windows.cpp`)**:
  * Routed `ScratchMem` allocation in Basilisk II through `vm_acquire_mac()` to guarantee allocation within the 32-bit space required by ROM patches.
  * Corrected SheepShaver's `DRCacheAddr` mapping to directly target `DR_CACHE_BASE`.
* **Large (>4 GB) Disk Image Detection (`sys_windows.cpp`)**:
  Replaced 32-bit `GetFileSize()` with `GetFileSizeEx()`, enabling correct capacity calculation and partition mounting for disk images exceeding 4 GB.
* **Video Blitter (`video_blit.h`)**:
  Changed `(unsigned long)source & 1` alignment checks to `(uintptr)source & 1`. On 64-bit Windows (LLP64), `sizeof(unsigned long) == 4` while `sizeof(void *) == 8`; casting to `uintptr` avoids pointer truncation.
* **Handles & Windows Shell (`ntcd.cpp`, `tcp.cpp`, `prefs_editor_gtk.cpp`)**:
  Safely routed Windows `HANDLE`, `HWND`, and `HINSTANCE` pointer conversions through `uintptr_t` / `intptr_t`.

### 4. Windows Single-Instance Protection & Directory Isolation
* **Config-Aware Instance Guard (`main_windows.cpp`, `prefs_windows.cpp`)**:
  Added an automated single-instance check to prevent data corruption caused by launching multiple instances against the same disk images.
  * Computes an FNV-1a 64-bit hash of the normalized, canonical full path of the active preferences file (`GetPrefsPath()`) and locks a session-scoped Windows mutex (`Local\<App>_Instance_<hash>`).
  * If a duplicate instance is launched for the same configuration, it brings the existing emulator window to the foreground (`EnumWindows` -> `ShowWindow(SW_RESTORE)` -> `SetForegroundWindow`) and cleanly exits.
  * Instances run from separate folders or distinct configuration files (`--config`) are isolated and continue to run concurrently without interference.
  * Added `--allow-multiple` command-line switch and an `allowmultiple` boolean preference setting to bypass the check when multiple instances on the same config are deliberately desired.

### 5. Build System & Autotools Improvements
* **JIT Detection**:
  Configured `configure.ac` to automatically disable the dyngen JIT compiler on `aarch64* | arm64*` targets (dyngen JIT is restricted to 32-bit x86).
* **Latent Bug Fix in BasiliskII `configure.ac`**:
  Corrected shell test `[ "x$WANT_JIT" = "xyes" -a "x$CAN_JIT" ]` to `[ "x$WANT_JIT" = "xyes" -a "x$CAN_JIT" = "xyes" ]`, fixing an issue where JIT was improperly enabled even when `CAN_JIT=no`.
* **GTK Preferences Editor Flags**:
  Removed x86-specific `-mms-bitfields` and unsupported `-static-libgcc` flags from the GUI build targets, and added `-fpermissive` to handle legacy GTK callback signature casts under modern C++ compilers.
* **Filesystem Case Sensitivity**:
  Corrected `b2ether/Packet32.cpp` to lowercase `b2ether/packet32.cpp` in `BasiliskII/src/Windows/Makefile.in`.

---

## How to Build on Windows ARM64

### Prerequisites
Install [MSYS2](https://www.msys2.org/) on Windows ARM64 and open the **MSYS2 CLANGARM64** environment:

```bash
# Update base system
pacman -Syu

# Install toolchain and dependencies
pacman -S --needed \
  gcc git make base-devel autoconf automake perl \
  mingw-w64-clang-aarch64-{toolchain,gcc-compat,gtk2,SDL2}
```

### Cloning with Symlink Support
Upstream `macemu` uses Git symlinks for `SheepShaver` files pointing into `BasiliskII`. On Windows, enable Developer Mode or clone with symbolic link support:

```bash
git clone -c core.symlinks=true https://github.com/tordona/macemu.git
cd macemu
git checkout aarch64
```

---

### Building BasiliskII (Windows ARM64)

```bash
cd BasiliskII/src/Windows
../Unix/autogen.sh
make -j$(nproc)
```

Outputs:
* `BasiliskII.exe`
* `BasiliskIIGUI.exe`

---

### Building SheepShaver (Windows ARM64)

```bash
cd SheepShaver
make links
cd src/Windows
../Unix/autogen.sh
make -j$(nproc)
```

Outputs:
* `SheepShaver.exe`
* `SheepShaverGUI.exe`

---

## Upstream Compatibility

All patches have been designed and guarded to maintain 100% backward compatibility with upstream supported platforms:
* Existing x86 and x86_64 inline assembly is preserved unmodified.
* macOS, Linux, and BeOS build scripts and sources remain fully functional.
* Upstream Git symlink structures are fully preserved.
