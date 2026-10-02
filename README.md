# Hi3798Mv100 32bit 固件 — 两个发行版

同一套内核、同一套 rootfs，唯一差异是 `bootargs9-32.bin` 里的 `mmz` 参数。
两个版本都已在真机刷写验证过，见下方"实测状态"。

## 版本对比

|                       | TTL-hi3798mv100-32bit-vdec | TTL-hi3798mv100-32bit-maxmem |
| --------------------- | -------------------------- | ---------------------------- |
| `bootargs_768M` 的 mmz | `mmz=ddr,0,0,220M`         | `mmz=ddr,0,0,48M`            |
| fastboot 算出 offset    | 292M                       | 464M                         |
| 内核 CMA 大小             | 220 MiB                    | 48 MiB                       |
| 内核可用内存                | 501.4 MiB                  | 673.7 MiB                    |
| VDEC 硬件解码             | 可用                         | 不可用                          |
| HDMI 原生 TTY           | 正常                         | 正常                           |

## 怎么选

需要硬件视频解码（播放视频、转码、监控回放）就选 **vdec** 版。
纯当 Linux 服务器用（SSH、容器、下载、文件服务），用不到 VDEC，
想多留 172 MiB 可用内存，就选 **maxmem** 版。

## 刷机

进对应目录，用 HiTool 选 **`emmc_TTL-hi3798mv100-32.xml`** 全量刷写。
该 XML 的 9 行全部 `Sel="1"`，写满 eMMC 的全部 9 个分区。

两版都以 `bootargs9-32.bin` 作为 bootargs 分区的内容（文件名相同，内容不同），
所以分区表 XML 完全通用，不需要区分。

## 实测状态

两个版本都在同一台盒子上刷写并回收 TTL 日志验证：

- **vdec 版**：`cma: Reserved 220 MiB at 0x12400000`，
  `VDEC_DRV_Init` 无任何 FATAL，`cma` 无分配失败，
  `Console: switching to colour frame buffer device 240x67`，
  到达 `histb-6ede58ec login:`，HDMI 同步显示同一内容。
- **maxmem 版**：`cma: Reserved 48 MiB at 0x1d000000`，
  `VDEC_AllocPreBuffer` 报 `VdhBufSize:157696000` 分配失败，
  `VDEC_DRV_ModInit: Init drv fail!`，
  其余（fbcon、login、网络）均正常。

## mmz 参数为什么这么算

盒子 DDR 为 768M，fastboot 的 `lib/reserve_mem.c` 会为该档位重写 mmz 的 offset 字段：

    offset = ((ddrsize/3) * 2) - mmz_size = 512M - mmz_size

内核把 mmz 解析为一块 CMA（`phys_start = offset`，`nbytes = mmz_size`），
而 fastboot 自己的预留区下界固定为 249M。于是：

    CMA 大小   = mmz_size
    预留区可用 = 263M - mmz_size      （hifb 的 logo buffer 需 8,298,496 B）

两件事争同一块空间。VDEC 需要 157,696,000 B（10 帧 4K 缓冲）
加上约 29.3 MiB 的既有 CMA 占用，共约 179.7 MiB；且 CMA 分配要求物理连续。
220M 时 CMA 余 40.3 MiB、预留区余 43 MiB，两侧都够。
