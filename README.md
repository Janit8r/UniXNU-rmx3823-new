# UniXNU on realme GT5 240W (RMX3823)

Device repository for running UniXNU on the realme GT5 240W. The board is
Qualcomm `kalama` / SM8550, so shared platform support comes from
`UniXNU-sm8550-common` and this repository carries only board inputs.

Follow [Port a device](https://unixnu.com/porting/) from input collection through
publication, and read `AGENTS.md` first.

## Board summary

| Item | Value |
| --- | --- |
| Model | realme RMX3823 (GT5 240W) |
| SoC | SM8550, `kalama`, `soc_id` 519, revision 2.0 |
| Firmware | `RMX3823_16.0.7.200(CN01)`, Android 16, security patch 2026-05-01 |
| CPU | 3x Cortex-A510, 2x Cortex-A710, 2x Cortex-A715, 1x Cortex-X3 |
| GPU | Adreno740v2 |
| Memory | 24 GiB SKU, page size 4096 |
| Display | 1240 x 2772, 560 dpi |
| Storage | UFS, A/B slots, active slot `_a`, `dtbo_idx` 0 |
| Touch | Synaptics S3910 or Goodix Berlin (dual source) |

Board overlays come from the stock `dtbo.img`: entries 1 and 2
(`Kalama MTP,zonda-A/B`, `qcom,msm-id` 0x207) match this device, entries 0 and 3
carry 0x258/0x259 and do not.

## Layout

- `board.toml` board identity, component selection, kernel, firmware, payload.
- `feeds.toml` pins the SoC source at a reviewed commit.
- `dts/board.dts` board device tree; includes `sm8550.dtsi` from the SoC source.
- `configs/rmx3823_defconfig` board kernel deltas.
- `patches/xnu/series` board kernel patches, in application order.
- `package/` board packaging and `Kit.toml`.
- `porting.toml` maintainer and agent record, one area per porting stage.

## Build

Building requires macOS or Linux. On Windows, only `defconfig` and `status` run.

```sh
./unixnu defconfig rmx3823_defconfig --device ../UniXNU-rmx3823
./unixnu fetch     --device ../UniXNU-rmx3823
./unixnu status --json --device ../UniXNU-rmx3823
./unixnu tree      --device ../UniXNU-rmx3823
./unixnu build kernel --device ../UniXNU-rmx3823
```

On a Windows host, build in CI instead: run the `macOS build` workflow in the
UniXNU checkout with `device_repository = <owner>/UniXNU-rmx3823`,
`device_ref = main`, `defconfig = rmx3823_defconfig` and `steps = kernel`.

`local.toml` is ignored by Git and points the SoC source at a local checkout.

## Status

Not bootable yet. The SoC layer is reused unchanged; the open work is the board
memory map, panel and touch identification from the firmware package, and a
`uefi-rmx3823` Mu-Silicium checkout.

- [Device repository](https://unixnu.com/porting/device-repository)
- [Device tree](https://unixnu.com/porting/device-tree)
- [Component catalog](https://unixnu.com/reference/components)
