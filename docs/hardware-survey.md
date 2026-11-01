# realme GT5 240W (RMX3823) — UniXNU 移植硬件勘察

采集时间：2026-11-01 11:15–11:20 GMT+8
采集方式：`adb`（无 root，`user` 版固件，SELinux enforcing）
目标：把 UniXNU（Darwin/XNU on ARM64）跑到本机

---

## 1. 设备身份

| 项 | 值 |
| --- | --- |
| 厂商 / 型号 | realme RMX3823（GT5 **240W** 版；RMX3820 是普通版） |
| 内部代号 | `RE5C33`（`ro.product.device`） |
| SoC | `SM8550`，平台代号 `kalama`，`ro.soc.manufacturer=QTI` |
| 系统 | Android 16，`RMX3823_16.0.7.200(CN01)`，构建于 2026-05-14 |
| 安全补丁 | 2026-05-01 |
| 内核 | `5.15.180-android13-8-o-01153-g6f24737ed732`，`SMP PREEMPT`，aarch64 |
| 构建类型 | `user` / `release-keys`（非 debug） |

## 2. 与 UniXNU-sm8550-common 的匹配度

这是本次勘察最重要的结论：**共享 SoC 层可以直接复用，不需要改。**

| 项 | 本机实测 | `dts/sm8550.dtsi` | 结论 |
| --- | --- | --- | --- |
| SoC | SM8550 / kalama | `compatible = "qcom,sm8550"` | 一致 |
| CPU 集群 | `0xd46`×3 (A510) + `0xd47`×2 (A710) + `0xd4d`×2 (A715) + `0xd4e`×1 (X3) | 3×A510 + 2×A715 + 2×A710 + 1×X3 | 数量与型号完全一致 |
| 核心数 | 8 | 8 | 一致 |
| GPU | `Adreno740v2` | `qcom,adreno-43050a01` | 同属 A740 家族，需确认 v2 变体的 chip id |
| 页大小 | **4096**（`ro.boot.hardware.cpu.pagesize`） | Kit 强制 native 4K 页 | **关键匹配**，避开 16K 页机型 |
| EGL / Vulkan | `adreno` / `adreno` | `Adreno740` + `AdrenoMetal` | 一致 |

内存与显示：

- RAM：`23645600 kB` ≈ 22.5 GiB → **24GB 版本**
- 屏幕：`1240 × 2772`，`560 dpi`

> 页大小是 4K 这点非常关键。common 仓库的 `ShengGmuLoadAndStartHfi` 明确写着
> "mapped with native4K pages, never 16K pages"，而部分 8 Gen 2 机型是 16K 页，会直接不可用。

## 3. 存储与分区

UFS，A/B 双槽，当前活动槽 `_a`，`ro.boot.dtbo_idx=0`。

设备节点映射：

| 分区 | 设备节点 |
| --- | --- |
| `super`（动态分区） | `/dev/block/sda6` |
| `userdata` | `/dev/block/sda14` |
| `boot_a` | `/dev/block/sde13` |

分区全量（A/B 除单份者外均成对）：`abl` `aop` `aop_config` `apdp` `bluetooth` `boot` `cdt` `connsec` `cpucp` `ddr` `devcfg` `devinfo` `dinfo` `dip` `dsp` `dtbo` `engineering_cdt` `featenabler` `frp` `fsc` `fsg` `hyp` `imagefv` `init_boot` `keymaster` `keystore` `limits` `limits-cdsp` `logdump` `logfs` `mdcompress` `mdm1oemnvbktmp` `mdtp` `mdtpsecapp` `metadata` `misc` `modem` `modemst1` `modemst2` `multiimgoem` `multiimgqti` `ocdt` `oplus_sec` `oplusdycnvbk` `oplusreserve1..5` `oplusstanvbk` `param` `persist` `preisp_dt` `preisp_otp` `qmcs` `qupfw` `qweslicstore` `rawdump` `recovery` `rtice` `rticmpdata` `secdata` `shrm` `splash` `spunvm` `ssd` `storsec` `super` `toolsfv` `tz` `tzsc` `uefi` `uefisecapp` `uefivarstore` `userdata` `vbmeta` `vbmeta_system` `vbmeta_vendor` `vm-bootsys` `vm-data` `vm-persist` `xbl` `xbl_config` `xbl_ramdump` `xbl_sc_logs` `xbl_sc_test_mode`

realme/OPPO 特有、**移植时不可触碰**的分区：`oplus_sec` `oplusstanvbk` `oplusreserve1..5` `ocdt` `preisp_dt` `preisp_otp` `engineering_cdt` `oplusdycnvbk`。误刷会丢指纹/相机校准/保修信息。

## 4. 引导状态（当前唯一硬阻塞）

| 属性 | 值 | 含义 |
| --- | --- | --- |
| `ro.boot.flash.locked` | `1` | **Bootloader 已锁** |
| `ro.boot.verifiedbootstate` | `green` | 验证启动链完整、未被篡改 |
| `ro.secure` | `1` | 强制安全启动 |
| `ro.debuggable` | `0` | 无调试内核 |
| `ro.build.tags` | `release-keys` | 官方签名 |

**在此状态下无法刷入任何自定义镜像，包括 Mu-Silicium UEFI 和改过的 boot。** 这是 Phase 0，不是可以绕过的细节。

可行路径：realme「深度测试」官方解锁流程 —— 安装机型专用 Deep Testing APK、登录 realme 账号、提交申请、等待约 7 天审核、通过后在开发者选项里解锁（**会清空全部数据**）。

现状判断：社区在 2026-08 仍在发布 `RMX3823_16.0.5.1010(CN01)` 的 Magisk 包，说明该机型**目前仍可解锁**。但本机固件是更新的 `16.0.7.200`，Deep Testing APK 的版本支持需要单独确认。

## 5. 本次拿不到、必须另找的数据

`user` 版固件 + SELinux 挡住了以下内容（已实测 `Permission denied`）：

- `/sys/firmware/devicetree/base/{model,compatible,soc}` —— 板级 DTS 主体
- `/proc/cmdline`
- UFS 型号（`/sys/block/sd*/device/model`）
- 面板型号与初始化序列、触摸 IC 型号

`reserved-memory` 的**目录名**可以读（属性被挡），已拿到 60+ 个 region 名称，包括
`aop_cmd_db_region@81c60000`、`gpu_micro_code_region@9b09a000`、`smem_region@81d00000`、
`qtee_region@d8300000`、`xbl_sc_region@d8100000`、`cpucp_fw_region@d8140000`、
`trust_ui_vm_region@f3800000` 等——这些名字本身就是内存布局的重要线索。

**正确获取途径**（不需要 root）：下载官方线刷包，解包 `dtbo.img` / `vendor_boot.img` / `xbl.img`，
用 `dtc` 反编译得到完整板级 DTS。这同时能拿到面板、触摸、UFS 的全部真实参数。

## 6. 仓库现状与结构问题

`github.com/Janit8r/UniXNU-RMX3823` 克隆后确认：**它是 UniXNU 主仓库（引擎）的 fork，不是设备仓库。**

判断依据：含 `components.toml`（17KB）、`devices.toml`（84KB，203 个设备的注册表）、
`profiles/`、`drivers/`（14 个组件）、`kits/`（6 个）、`package/`、`templates/`、`unixnu` CLI、
`.github/workflows/`（docs / macos / porting-record / sync-devices / terminal）。

按 UniXNU 的仓库模型，正确布局应是四个仓库并排：

```
UniXNU/                 引擎（当前 fork 应放在这里，不要按机型命名）
UniXNU-sm8550-common/   SoC 共享层（已有，可直接复用）
UniXNU-rmx3823/         设备仓库 ← 尚不存在，需要从 examples/device 创建
uefi-rmx3823/           Mu-Silicium UEFI 固件 ← 尚不存在，工作量最大
```

`examples/device/` 已经给了完整模板：`board.toml`、`feeds.toml`、`porting.toml`、
`dts/board.dts`、`configs/example_defconfig`、`patches/xnu/series`、`package/`、
`scripts/package.sh`、`proprietary-files.txt`、两个 CI workflow。

## 7. 参照设备

`devices.toml` 里已有多个 SM8550 设备可作模板：

| 设备 | 机型 | 状态 |
| --- | --- | --- |
| `sheng` | 小米平板 6S Pro | 唯一有硬件验证记录的 |
| `fuxi` | 小米 13（手机） | `board = "sheng"`，摘要写 "no hardware run" |
| `nuwa` | 小米 13 Pro | — |
| `nx729j` | 红魔 8 Pro | — |

注意 `fuxi` 的摘要明确写着 **"Kernel links and prelinks; stage B archive chain near complete; no hardware run"** ——
即便是已有的 SM8550 手机移植，也还没有真正的硬件运行记录。这决定了本项目的时间预期。

## 8. 分阶段计划

| 阶段 | 内容 | 阻塞条件 |
| --- | --- | --- |
| P0 | 解锁 Bootloader（深度测试，约 7 天，清数据） | **必须**，否则后续全部无法验证 |
| P1 | 解包官方线刷包 → `dtc` 反编译 dtbo → 得到板级 DTS、面板/触摸/UFS 参数 | 需要线刷包，不需要设备 |
| P2 | 从 `examples/device` 创建 `UniXNU-rmx3823`，填 `board.toml`（`soc_source = "sm8550"`）、defconfig、板级 DTS | 依赖 P1 |
| P3 | 打通内核链：`./unixnu defconfig rmx3823_defconfig` → `tree` → `build kernel`，先求「能编过」 | 依赖 P2 |
| P4 | Mu-Silicium UEFI 平台包（`uefi-rmx3823`）：内存映射、DSC/FDF、面板点亮 | 依赖 P0 才能验证 |
| P5 | 打包 bootimg，首次上机；绿条/无显示排障 | 依赖 P0/P4 |

## 9. 风险清单

1. **解锁可能失败**：`16.0.7.200` 是否在 Deep Testing 支持列表内未确认。失败则项目止步于 P2，只能做不可验证的静态移植。
2. **OPPO/realme 的 ABL 定制**：`uefi_a/b` 分区存在，但 OPPO 对 boot 链有额外校验，Mu-Silicium 是否能在此机型握手未知。
3. **面板驱动**：realme 定制 AMOLED，无开源面板驱动，需从 vendor 内核或 dtbo 反推初始化序列。这是最耗时的部分。
4. **无硬件验证先例**：SM8550 手机侧目前没有任何一台有成功记录（`fuxi` 亦然）。
5. **工具链过旧**：本机 `adb` 为 1.0.26.29（2014 年）、`fastboot` 更老，均需升级到 platform-tools 最新版，否则刷写阶段会出问题。

## 10. 原厂固件解析结果（2026-11-01）

用户提供了 5 个原厂固件文件（`uefi.elf`、`uefi_sec.mbn`、`tz.mbn`、`boot.img`、
`recovery.img`）。这一批把「内存表」和「触摸」两个悬空项都解决了。

### 10.1 `uefi.elf` —— 原厂 UEFI，也是内存表的权威来源

- ELF64，入口 `0xA7000000`，单一 LOAD 段 `0xA7000000` + `0x300000`。
  这就是原厂固件卷的位置和大小。
- 固件卷内嵌了完整的 `uefiplatLA.cfg` 文本（原厂 UEFI 启动时按
  `PlatConfigFileName = "uefiplatLA.cfg"` 解析）。在文件偏移 `0x4703C` 处以
  字符串 blob 形式存在，提取后 204 行、纯 ASCII。
- 该配置声明 `Version = 3`、`MaxMemoryRegions = 74`，实际使用 39 个区域：
  25 个 DDR、2 个 IMEM、12 个寄存器窗口。已逐条转录进
  `uefi-rmx3823` 的 `rmx3823Pkg/Library/MemoryMapLib/MemoryMapLib.c`，
  并用脚本逐行比对，39 行完全一致、分段内零重叠。

关键数值：

| 区域 | 地址 + 大小 | 说明 |
| --- | --- | --- |
| NOMAP | `0x80000000 + 0x01200000` | gunyah_hyp + cpusys_vm |
| XBL DT | `0x81A00000 + 0x00040000` | XBL 发布的设备树 |
| AOP | `0x81C00000 + 0x000A0000` | 含 aop_cmd_db / aop_config |
| SMEM | `0x81D00000 + 0x00200000` | 旧 DTS 写 `0x400000`，偏大一倍 |
| PIL Reserved | `0x8A800000 + 0x18280000` | 覆盖全部 PIL 子区 |
| Display Demura | `0xA2A80000 + 0x02B00000` | |
| UEFI FD | `0xA7000000 + 0x00400000` | 原厂镜像占前 3 MiB |
| UEFI FD Reserved | `0xA7400000 + 0x00200000` | |
| Kernel | `0xA8000000 + 0x10000000` | 原厂内核窗口，ABL 装 boot.img 处 |
| Display Reserved | `0xB8000000 + 0x02B00000` | |
| DXE Heap | `0xBAB00000 + 0x0D1C0000` | **唯一标为 Conventional 的连续区** |
| TZApps Reserved | `0xD8800000 + 0x07400000` | |

其他可用配置项：`NumCpus = 8`、`NumActiveCores = 8`、
`ShmBridgememSize = 0xA00000`、`SecurityFlag = 0xC4`、
`DefaultBDSBootApp = "LinuxLoader"`、`EnableShell = 0x1`。

### 10.2 三方交叉验证

- `tz.mbn` 入口点与首个 LOAD 段地址 `0x14680000`，等于配置里的
  `IMEM Base`；其 `0xd830a000` 段落在 `TZApps Reserved` 内。
- dtbo overlay 里 `aop_cmd_db_region@81c60000`、`smem_region@81d00000`、
  `gpu_micro_code_region@9b09a000` 三者都落在配置对应区域内。

### 10.3 `boot.img` / `recovery.img` 没有基础设备树

两者都是 **header_version 4**，该格式把基础 DTB 放在 `vendor_boot.img`，
不在 `boot.img` 里。全文件扫描无 DTB 魔数（`d00dfeed`）。因此
**面板型号仍然未知**，需要 `vendor_boot.img` 或设备 `/sys/firmware/fdt`。

顺带：`boot.img` 恰好 192 MiB、`recovery.img` 恰好 100 MiB，是分区大小的
原始 dump，不是紧凑的镜像文件。

### 10.4 触摸控制器：Synaptics S3910（SPI）

从 `dtbo.img` 条目 1（`Kalama MTP,zonda-A/B`）的 `fragment@66` 得到，三个
候选里只有它 `status = "ok"`：

| 节点 | 状态 |
| --- | --- |
| `synaptics_tcm@0`（`synaptics,tcm-spi`） | **ok** |
| `goodix-berlin@5d` | disabled |
| `synaptics_tcm_hbp@0` | disabled |

启用节点的参数：

| 项 | 值 |
| --- | --- |
| chip-name | `S3910` |
| 总线 | SPI，CS0，`spi-max-frequency` `0xE4E1C0`（15 MHz） |
| IRQ | GPIO 25，`interrupts = <25 0x2008>`（下降沿 + ONESHOT） |
| reset | GPIO 24 |
| avdd | GPIO 10 |
| 触摸点阵 | TX 17 × RX 38（`0x11 × 0x26`） |
| `panel_type` | 7 |
| `firmware_name` | `AB641` |
| `display-coords` | `0x4D8 × 0xAD4` = **1240 × 2772**，与实机分辨率一致 |

移植含义：`drivers/ShengNovatekTouch` 用不上，需要 Synaptics TCM（SPI）
实现。dtbo 里同时存在 `goodix` 节点（`panel-max-x 0x438`、`max-y 0x960`，
即 1080×2400），说明该机型存在不同面板/触摸的 SKU，本机是 1240×2772 +
S3910。

### 10.5 仍未解决

1. **面板型号**：缺 `vendor_boot.img` 或设备端 FDT。`panel_type = 7` 与
   1240×2772 是唯一线索。
2. **实际内存容量**：原厂映射止于 `0x100000000`，不描述已装 DRAM。24 GiB
   仍是从 `MemTotal 23645600 kB` 推断。
3. **UEFI 安全应用**：`uefi_sec.mbn` 含 OPLUS 密钥，与移植无关，仅确认机型
   为 OPLUS SM8550。
