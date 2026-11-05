# uefi-framework

A freestanding **x64 UEFI application framework** — MSVC-built, zero dependencies, no EDK II.
We use it to run our own code **before the operating system exists**: poking the boot chain and
Secure Boot variables (PK/db/dbx) at the UEFI stage, where Windows-side tools can't reach.
This is not a study project — the goal is to ship a working hack, so the framework is kept
minimal and boring on purpose.

If you can build a C++ exe in Visual Studio, you can build and flash this in five minutes
(see [Quick start](#quick-start)).

## Why not EDK II

EDK II is an ecosystem: build tools, packages, PYTHON, a DSC/FDF ceremony. We want one `.efi`,
F5 in Visual Studio, copy to a USB stick, reboot. This repo is ~40 headers and a handful of
`.cxx` files. Everything you need at the UEFI stage is here; add what you need as you hack.

## What's inside

```
uefi-framework/
├── uefi-framework.sln            # open this in Visual Studio 2022
├── source/
│   └── entry.cxx                 # EfiMain() — your code starts here
└── ext/fw/
    ├── fw.h                      # umbrella include: all EFI ABI
    ├── context.h                 # g_ctx — image handle, system/boot/runtime tables, con_in/con_out
    ├── efi/                      # hand-written EFI ABI (MS x64 calling convention)
    │   ├── abi.h guid.h status.h
    │   ├── service/              # boot_services, runtime_services, system_table
    │   └── protocol/             # loaded_image, file_system, device_path,
    │                             # graphics_output, simple_text_input/output
    ├── core/
    │   ├── arch.h                # CR0/CR3, PML4E..PTE bitfields, virtual_address, WPdisable
    │   ├── crt/                  # our CRT: memcpy/memset (MASM), operator new, string fns
    │   └── stl/                  # mini-STL: string_view, span, optional, type_traits, utility
    └── ztils/                    # research utilities
        ├── serial/               # c_serial — raw COM1 (0x3F8) 115200 8N1, {}/{:x} formatting
        ├── pattern_scan/         # IDA-style signature scanner (?? wildcards), scan_as<T>()
        ├── pe_util/              # PE64 parse: sections, exports, RVAs, map_to(), apply_relocs()
        └── detour_hook/          # c_detour_hook — 14-byte JMP detour with save/enable/disable
```

## Quick start

**Build.** Requires Visual Studio 2022 (v143 toolset, any edition incl. BuildTools) with the
*C++ x64/x86 build tools* workload (MASM is included). Open the `.sln`, pick **Release | x64**,
build. Output: `build\x64\Release\uefi-framework.efi`. Command line:

```
msbuild uefi-framework.sln /p:Configuration=Release /p:Platform=x64
```

**Run.** Copy the `.efi` to a FAT32 USB stick and boot it:

- From the **UEFI Shell**: `fs0:` then `path\to\uefi-framework.efi` (best iteration loop —
  no renaming, just re-copy and re-run).
- From the **boot menu fallback**: name it `\EFI\BOOT\BOOTX64.EFI` on the stick.

⚠ **Secure Boot**: an unsigned `.efi` is refused while SB is enforced. Either disable Secure
Boot in firmware for development, or run on a machine where we already own the PK (that's
part of the plan — see [What we're hacking](#what-were-hacking)).

## Writing your first hack

`source/entry.cxx` starts clean:

```cpp
#include "fw/context.h"          // pulls in the whole ABI + g_ctx

efi_status_t EfiMain(efi_handle image_handle, efi::system_table* system_table) {
    g_ctx.init(image_handle, system_table);

    // text console:
    g_ctx.con_out()->output_string(u"hello from uefi-framework\n");

    // runtime services — e.g. read a Secure Boot variable:
    constexpr auto efi_global_variable =
        efi::make_guid(0x8BE4DF61, 0x93CA, 0x11D2, 0xAA, 0x0D, 0x00, 0xE0, 0x98, 0x03, 0x2B, 0x8C);

    std::uint32_t attrs = 0;
    std::uintptr_t sz = 0;
    g_ctx.runtime_services()->get_variable(u"PK", const_cast<efi_guid*>(&efi_global_variable),
                                           &attrs, &sz, nullptr);   // -> buffer_too_small, sz filled
    // ... allocate sz, call again, done.

    return efi::status_codes::success;
}
```

## Debugging: serial, because there is no debugger

`g_serial` is a plain 16550 COM1 driver (port `0x3F8`, 115200 8N1, no interrupts). On bare
metal use a USB↔TTL cable on the MB's COM header; in QEMU/OVMF use
`-serial mon:stdio`. Format strings use `{}` positional placeholders, `{:x}` for hex:

```cpp
#include "fw/ztils/serial/serial.h"

g_serial.println("image at {:x}, size {}", image_base, size);
g_serial.dump(ptr, 0x40);          // hexdump
g_serial.assert(ok, "GOP not found");
```

Tip: print **early and often** in `EfiMain` — an EFI app that "hangs" is usually a fault with
nobody to tell you. Wire `println` before each stage.

## Utilities in anger

```cpp
// IDA-style pattern scan over any buffer/image range:
auto* tgt = ztils::scan_as<void*>(base, size, "48 8B ?? 48 89 5C ?? ?? 57 48 83 EC ?? 48 8B DA");

// PE walking (e.g. bootmgfw.efi / winsloader mapped into memory):
if (ztils::pe_valid(base)) {
    auto text = ztils::get_section(base, ".text");
    auto* exp = ztils::get_export(base, "EfiMain");
    ztils::apply_relocs(base, delta);            // after map_to() at a new base
}

// inline detour (remember WP: pages are RX under OS loader maps; under bare EFI RWX is fine):
c_detour_hook hk;
hk.make(target_fn, my_replacement);   // 14-byte abs jmp, original bytes saved
hk.disable();                          // restore + re-enable later
```

## Rules of the house

- **No CRT, no exceptions, no RTTI.** `IgnoreAllDefaultLibraries`; our own
  `memcpy/memset/memmove` live in `core/crt/memory/memory.asm`. Don't include `<string>`,
  `<vector>` etc. — use `fw/core/stl.h` and fixed buffers.
- **C++23, `/GS-`** (no security cookies), MSVC only. x64 EFI ABI (Microsoft x64 calling
  convention) throughout.
- **Keep the entry lean.** `EfiMain` is the playground; put reusable bits under `ext/fw/ztils/`.
- Status codes: check `efi::status_codes`, never assume success from boot services.
- `build/`, `.vs/`, `*.vcxproj.user` are gitignored — never commit them.

## What we're hacking

Motivation for this framework (context for the research, not an EDK tutorial):

- **Own the keys before the OS does.** The endgame is manipulating Secure Boot key material
  (PK/db/dbx) from the UEFI stage itself — Runtime Services are right there, no
  `SetFirmwareEnvironmentVariable` privileges dance, no vendor tool dependency.
- **Boot-chain surgery.** `pe_util` + `pattern_scan` + `detour_hook` are exactly the kit for
  mapping `bootmgfw.efi`/`winload.efi` yourself, finding the interesting exports/code regions,
  and detouring them — see the `bootmgfw` hint in `detour_hook.h`.
- **Anti-cheat / early-boot questions** we can only answer with code that runs at tick 0:
  what a driver in our position can see and change before any OS-resident agent exists.

Anything in this repo may break while we iterate. If it builds and boots, it's fair game.
