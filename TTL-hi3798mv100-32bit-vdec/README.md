# TTL-hi3798mv100-32bit — VDEC 版

**用途**：需要 Hi3798Mv100 硬件视频解码（播放视频、转码、监控回放）时使用。

## 版本参数

    bootargs_768M=mem=768M mmz=ddr,0,0,220M vmalloc=500M

fastboot 会把 mmz 的 offset 字段改写为 `292M`，内核实际收到：

    mem=768M mmz=ddr,0,292M,220M vmalloc=500M

| 项           | 值         |
| ----------- | --------- |
| 内核 CMA      | 220 MiB   |
| 内核可用内存      | 501.4 MiB |
| VDEC 硬件解码   | 可用        |
| HDMI 原生 TTY | 正常        |

## 实测数据

    cma: Reserved 220 MiB at 0x12400000
    cma: Reserved 4 MiB at 0x2fc00000
    Memory: 513392K/786432K available
    Console: switching to colour frame buffer device 240x67
    histb-6ede58ec login:

VDEC 全程无任何 ERROR 或 FATAL，无 `cma: alloc ... failed`，
全篇无 Oops、无失败单元。

## 刷机

HiTool → 分区表选 `emmc_TTL-hi3798mv100-32.xml`（9 个分区全部 Sel="1"，全量刷写）。

## 参数依据

VDEC 需要 157,696,000 B 连续 CMA（10 帧 4K 缓冲），
加上约 29.3 MiB 既有占用共约 179.7 MiB；且 CMA 要求物理连续。
同时 fastboot 自己的预留区（给 hifb 的 logo buffer 用）下界固定在 249M，
可用量为 `263M - mmz_size`。

220M 时两侧分别为 CMA 220 MiB、预留区 43 MiB，都满足需求。
