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
| `uefi_fd.base` | `0xA7100000` | Must equal the `UEFI FD` entry in `MemoryMapLib.c` |
| `uefi_fd.size` | `0x300000` | 3 MiB inside the stock 4 MiB FD window at `0xA7000000` |
| `boot_image.header_version` | `1` | v1/v2 keep the kernel slot at a fixed offset, which is what the BootShim payload layout expects |
| `boot_image_kernel.append_dtb` | `false` | No `Resources/DTBs/rmx3823.dtb` exists, and XNU takes its ADT from this repository's `dts/board.dts` anyway |

The lint workflow in the firmware repository checks that the first two agree
with `MemoryMapLib.c`, because a mismatch there produces a firmware that loads
over its own memory map.

## Memory map: not yet verified

`rmx3823Pkg/Library/MemoryMapLib/MemoryMapLib.c` still carries the QRD8550
reference layout. That is a defensible starting point but it is **not**
confirmed against this board.

What corroborates it: three carveouts that are independently observable from the
board's own dtbo overlay (entry 1, machine `Kalama MTP,zonda-A/B`) land exactly
where the reference map puts them.

| Carveout in the overlay | Address | Reference map entry |
| --- | --- | --- |
| `aop_cmd_db_region` | `0x81C60000` | `AOP` `0x81C00000` + `0xA0000` |
| `smem_region` | `0x81D00000` | `SMEM` base |
| `gpu_micro_code_region` | `0x9B09A000` | inside `PIL Reserved` `0x8A800000` + `0x18280000` |

What is still unverified, and what it costs if wrong:

- `Display Demura` and `Display Reserved` sizes are panel-specific.
- `TZApps Reserved` size.
- That `0xA8000000..0xB8000000` (`Kernel`) is genuinely unreserved — UEFI hands
  that window to the next-stage kernel.
- That the `0xA7100000` FD window is unreserved.

Getting these wrong means UEFI either hands the kernel memory XBL still owns, or
scribbles over a carveout. A boot failure at this stage should be treated as a
memory-map problem before suspecting drivers.

The authoritative sources are, in order:

1. The stock firmware's `uefiplat.cfg`, which is how shengPkg's 72 descriptors
   were reconciled against 34 Linux DTS reservations.
2. A dump of the running device's `/proc/device-tree/reserved-memory`, or
   `/sys/firmware/fdt`.
3. The `XBL DT` carveout at `0x81A00000`, which is the device tree XBL publishes
   to the OS.

None of these is reachable from `dtbo.img`: the overlays carry display,
regulator and touch fragments, not the base memory map.

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

The package has been created and integrated but **has never been compiled**.
The first firmware build is the next step, followed by reconciling the memory
map against the stock firmware.
