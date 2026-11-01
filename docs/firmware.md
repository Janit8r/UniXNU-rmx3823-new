# UEFI firmware for rmx3823

The firmware is a fork of `UniXNU/uefi-sheng`, kept at
[`Janit8r/uefi-rmx3823`](https://github.com/Janit8r/uefi-rmx3823) on branch
`sheng`. It is not vendored into this repository: a firmware tree is an EDK II
workspace with six submodules, and it has a different lifetime from the board
description here.

`board.toml [firmware]` names it, and `./unixnu build firmware --uefi <dir>`
builds it.

## The platform package

`Platforms/Qualcomm/rmx3823Pkg` in the firmware checkout.

It descends from `Platforms/Qualcomm/qrd8550Pkg`, the SM8550 reference design,
not from `nuwaPkg` or `shengPkg`. Both of those are Xiaomi boards with
Xiaomi-specific memory maps and Xiaomi display data; RMX3823 is a realme board
whose ABL comes from the same Qualcomm `boot_images` baseline as the reference
design. Nothing in the Xiaomi packages applies here, and the reference design
carries no vendor assumptions to unpick.

Two things are reused from QRD8550 on purpose:

- `Binaries/qrd8550/*` — the prebuilt Qualcomm DXE blobs. `Device-Binaries` has
  no `rmx3823` set, and these are SoC-level blobs, not board-level ones.
- `Silicium-ACPI/Platforms/Qualcomm/qrd8550/AcpiTables.inf` — Realme has no
  SM8550 ACPI package upstream and the tables are SoC-generic. XNU does not
  consume ACPI, so these only matter for a Linux payload.

Board deltas are limited to the platform GUID, the SMBIOS vendor and model
(`realme` / `RMX3823`), and the framebuffer geometry (1240x2772, the native
panel size).

## UniXNU integration

UniXNU cannot boot from a bare Mu-Silicium package. Four modules must be in the
firmware volume, and they are what the port adds on top of the reference design:

| Module | Purpose |
| --- | --- |
| `UniXNUPkg/Application/UniXNU/UniXNU.inf` | Boot menu and configuration reader |
| `UniXNUPkg/Driver/UniXNUDxe/UniXNUDxe.inf` | Boot option registration |
| `UniXNUPkg/Driver/UniXNURamDiskDxe/UniXNURamDiskDxe.inf` | Publishes the initrd range as Block I/O |
| `DarwinBootPkg/Driver/DarwinBoot.inf` | The XNU loader |

`UniXNURamDiskDxe` reads `/chosen/linux,initrd-start|end`, which the Android
bootloader writes into the firmware device tree. `DiskIoDxe`, `PartitionDxe`
and `EnhancedFatDxe` then expose the payload's FAT volume, which is how the menu
reads `\EFI\UniXNU\unixnu.toml`. `EnglishDxe` is not optional — FAT needs its
Unicode collation service.

DarwinBoot lives in the firmware volume rather than being loaded from one,
because the image verification of this build refuses the latter.

### Vendor hardware is off

`rmx3823.dsc` defines `RMX3823_VENDOR_HW = 0`, which keeps every vendor DXE blob
out of the firmware volume. At that setting `SimpleFbDxe` publishes a GOP over
the framebuffer ABL is already scanning out, so the menu is visible without a
single vendor hardware driver.

This is not a shortcut, it is the only usable setting today: rmx3823 has no
panel XML and no vendor `DisplayDxe` data in `RAW.inc`. On sheng, the equivalent
vendor chain blanked the panel *even with* that board's own panel data present.
Setting `RMX3823_VENDOR_HW = 1` loads the QRD8550 driver set — a set written for
a different board — and should be treated as an experiment, not a fix.

## Device configuration

`Resources/Configs/rmx3823.toml` in the firmware checkout:

| Key | Value | Why |
| --- | --- | --- |
| `uefi_fd.base` | `0xA7000000` | Must equal the `UEFI FD` entry in `MemoryMapLib.c`; also where the stock `uefi.elf` loads |
| `uefi_fd.size` | `0x400000` | The full carveout the vendor reserves. The stock image is 3 MiB of it, so this leaves 1 MiB of headroom |
| `boot_image.header_version` | `1` | v1/v2 keep the kernel slot at a fixed offset, which is what the BootShim payload layout expects |
| `boot_image.base` | `0x0` | BootShim is position independent, so this only has to be reachable RAM |
| `boot_image.kernel_offset` | `0x8000` | Same as sheng, the only SM8550 board known to boot through BootShim |
| `boot_image.tags_offset` | `0x1E00000` | Same as sheng |
| `boot_image.pagesize` | `4096` | Same as sheng |
| `boot_image_kernel.append_dtb` | `false` | No `Resources/DTBs/rmx3823.dtb` exists, and XNU takes its ADT from this repository's `dts/board.dts` anyway |

All five `boot_image` fields are stated even though they are the engine's
defaults, because `build_uefi.py` gets its defaults from `mkbootimg.py`
instead — `base 0x10000000`, `pagesize 2048`. With only `header_version` set,
the two build paths produced different images, and `mkbootimg.py`'s kernel
address `0x10008000` is not in SM8550's DDR at all. The firmware repository's
lint job now rejects the config if any of the five goes missing.

The lint workflow also checks that `uefi_fd` agrees with `MemoryMapLib.c`,
because a mismatch there produces a firmware that loads over its own memory map.

## Memory map: taken from the stock firmware

`rmx3823Pkg/Library/MemoryMapLib/MemoryMapLib.c` is no longer the QRD8550
reference layout. It is this board's own map — 39 regions — transcribed from the
`uefiplatLA.cfg` that the stock `uefi.elf` carries as an embedded string blob
(the stock UEFI parses that text at boot: `PlatConfigFileName =
"uefiplatLA.cfg"`). The extracted file is committed next to the table as
`Platforms/Qualcomm/rmx3823Pkg/uefiplat-rmx3823.cfg`.

The extraction is checked mechanically rather than by eye: a script re-extracts
the cfg from the ELF, parses both files and compares address, size and name row
by row. 39 rows, exact match, no overlaps within any section.

Other files corroborate it independently:

| Evidence | Agrees with |
| --- | --- |
| `tz.mbn` entry point and first LOAD vaddr, `0x14680000` | `IMEM Base` `0x14680000` + `0x2A000` |
| `tz.mbn` segment at `0xd830a000` | inside `TZApps Reserved` `0xD8800000` + `0x07400000` |
| `aop_cmd_db_region` `0x81C60000` (dtbo overlay) | `AOP` `0x81C00000` + `0xA0000` |
| `smem_region` `0x81D00000` (dtbo overlay) | `SMEM` base |
| `gpu_micro_code_region` `0x9B09A000` (dtbo overlay) | inside `PIL Reserved` `0x8A800000` + `0x18280000` |

Two things this table does **not** settle:

- It is not a description of installed RAM. It stops at `0x100000000`, so the
  24 GiB this device reports is covered only up to the low 2 GiB window plus
  carveouts. That is normal for Qualcomm UEFI — the kernel takes its range from
  the ADT — but it means the ADT memory node in `dts/board.dts`, not this table,
  is what has to be right about the top of RAM. That node is still provisional.
- `Display Demura` and `Display Reserved` are panel-specific, and no panel model
  has been identified yet. The sizes in the table are the vendor's own, which is
  the best available answer, but they are the first thing to re-check if the
  panel comes up wrong.

Two deliberate departures from a literal transcription, both documented in the
file itself. The vendor cfg marks both `NOMAP` entries `BuildHob = NoMap`, and
Mu-Silicium has no such HobOption; they are emitted as `AddMem` + `MEM_RES` +
`Reserv`, following `qrd8550Pkg`. And the regions the QRD8550 table carries that
this board's cfg does not — GPU PRR, TPMControl, USB UCSI Shared, PLD_PEP,
TZ STAT, ADSP RPC, MPSS_EFS and four PCIe apertures — were dropped rather than
inherited: the gap `0x81F00000..0x8A800000` they would fill is not described by
the vendor map at all, and a region absent from the table gets no HOB, so DXE
neither allocates nor maps it.

## Building

```sh
./unixnu build firmware --device . --uefi ../uefi-rmx3823
```

`rmx3823Pkg` is deliberately **not** buildable with Mu-Silicium's own
`build_uefi.py`. That script assembles `PACKAGES_PATH` from the Mu-Silicium tree
alone, so it cannot resolve `UniXNUPkg` or `DarwinBootPkg`. The engine supplies
them through `board.toml [firmware] packages` as `{tree}/package` and
`{tree}/drivers/DarwinBoot`.

The output is `Build/rmx3823Pkg/{target}_CLANGPDB/FV/SILICIUM_UEFI.fd`, and the
driver checks that its size matches `uefi_fd.size` before accepting it.

## Status

The firmware builds. `uefi-rmx3823` workflow run 41276749199 compiled
`rmx3823Pkg` on ubuntu-24.04 with `UniXNU.efi`, `UniXNUDxe.efi`,
`UniXNURamDiskDxe.efi`, `UniXNULib` and `DarwinBoot.efi` in the volume, and
produced a `SILICIUM_UEFI.fd` of exactly `0x300000` bytes plus a boot image.

Nothing has run on hardware. The two things that must be settled before a flash
is worth attempting are the memory map (above) and whether the ABL accepts the
boot image at all — a rejection there shows as the device dropping straight back
to fastboot with nothing on screen, which is distinct from a memory-map fault
later in DXE.
