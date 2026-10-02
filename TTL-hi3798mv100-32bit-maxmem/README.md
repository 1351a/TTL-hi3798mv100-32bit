# TTL-hi3798mv100-32bit — 大内存版

**用途**：纯 Linux 服务器场景（SSH、容器、下载机、文件服务），
不需要硬件视频解码，希望尽量多留可用内存时使用。

## 版本参数

    bootargs_768M=mem=768M mmz=ddr,0,0,48M vmalloc=500M

fastboot 会把 mmz 的 offset 字段改写为 `464M`，内核实际收到：

    mem=768M mmz=ddr,0,464M,48M vmalloc=500M

| 项           | 值         |
| ----------- | --------- |
| 内核 CMA      | 48 MiB    |
| 内核可用内存      | 673.7 MiB |
| VDEC 硬件解码   | **不可用**   |
| HDMI 原生 TTY | 正常        |

## 实测数据

    cma: Reserved 48 MiB at 0x1d000000
    Memory: 689864K/786432K available
    Console: switching to colour frame buffer device 240x67
    histb-6ede58ec login:

系统可以正常启动并到达 login，HDMI 正常显示 TTY，网络正常。
但 VDEC 初始化会失败（日志中可见）：

    ERROR-HI_VDEC: VDEC_AllocPreBuffer: VDH Alloc 0x9664000 failed
    FATAL-HI_VDEC: VDEC_DRV_Init: alloc pre buffer fail
    FATAL-HI_VDEC: VDEC_DRV_ModInit: Init drv fail!
    cma: alloc 38500 failed: -12

这不影响系统启动与日常使用，只是硬件视频解码不可用。

## 刷机

HiTool → 分区表选 `emmc_TTL-hi3798mv100-32.xml`（9 个分区全部 Sel="1"，全量刷写）。

## 为什么 VDEC 会失败

VDEC 预分配要 157,696,000 B（约 150.4 MiB）连续 CMA，
而 48M 档位只给 CMA 48 MiB，且其中已有约 29.3 MiB 固定占用，
最大连续空闲块仅 11.7 MiB，因此分配必然失败。

要 VDEC 可用请改用 `TTL-hi3798mv100-32bit-vdec` 版；代价是可用内存少约 172 MiB。
