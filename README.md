# Hi3798Mv100 32bit 固件 — 两个发行版

同一套内核、同一套 rootfs，唯一差异是 `bootargs9-32.bin` 里的 `mmz` 参数。
两个版本都已在真机刷写验证过。

---

## 版本信息

| 项    | 值                                                                        |
| ---- | ------------------------------------------------------------------------ |
| 发行版  | v1.0                                                                     |
| 构建日期 | 2026-10-02                                                               |
| 内核   | `4.4.35_ecoo_81082668`（已启用 fbcon），md5 `f9790e40b817f5fd587b54725b37acb0` |
| 变体   | `vdec`（mmz 220M）/ `maxmem`（mmz 48M）                                      |

本版包含五项修复：内核启用 fbcon（HDMI 出原生 TTY）、hifb 填充真实
`fb_var_screeninfo`、修复 logo workqueue 竞态（曾导致开机 Oops）、
调优 mmz 使 VDEC 与 fastboot 预留区共存、以及关闭控制台自动熄灭
（`consoleblank=0`）。

## 一、系统说明

### 发行版与架构

| 项      | 值                                    |
| ------ | ------------------------------------ |
| 发行版    | Debian 13.7 (trixie)                 |
| 架构     | armhf（ARM 32 位，hard-float ABI）       |
| 处理器    | Hi3798Mv100（Cortex-A7）               |
| DDR    | 768M（fastboot 选用 `bootargs_768M` 档位） |
| 内核     | `4.4.35_ecoo_81082668`，厂商内核重编        |
| 已装软件包  | 192 个（166 个 armhf + 26 个 all）        |
| rootfs | ext4，镜像内 512 MiB，首次开机自动扩展到占满 eMMC    |

armhf 是 Debian 的官方支持架构，`deb.debian.org` 的 trixie 与 trixie-security
两个 Release 文件的 `Architectures` 字段都含 `armhf`，可直接使用官方源。

### 登录

| 方式   | 说明                                      |
| ---- | --------------------------------------- |
| HDMI | 开机后直接显示 tty1 的登录提示符（内核 fbcon 实现，非第三方程序） |
| 串口   | `ttyAMA0`，115200 8N1，同样有登录提示符           |
| SSH  | `sshd` 默认启用，走 22 端口                     |

两个本地终端对应 `getty@tty1.service` 与 `serial-getty@ttyAMA0.service`，
均已启用。HDMI 与串口显示的是同一个 tty1 和同一个 hostname。

默认账号 `root`，默认密码 `ecoo1234`，**建议首次登录后立即修改**。

HDMI 上的实际效果（VDEC 版）：

![HDMI 登录提示符](image/WIN_20261002_13_17_30_Pro.jpg)

### hostname

镜像是 `histb`，首次开机会按 eMMC serial 推导成 `histb-<hex>` 的形式
（例如 `histb-6ede58ec`）。推导逻辑在 `/usr/local/sbin/ecoo-firstboot.sh`，
读取 `/sys/block/mmcblk0/device/serial`，读不到时依次退化为
`/proc/sys/kernel/random/uuid` 和时间戳。每台盒子各不相同。

### 内核命令参数

    model=mv100 console=ttyAMA0,115200 console=tty0 root=/dev/mmcblk0p9 \
    rootfstype=ext4 rootwait \
    blkdevparts=mmcblk0:1M(boot),1M(bootargs),4M(baseparam),4M(pqparam),\
    4M(logo),20M(kernel),64M(busybox),512M(backup),-(ubuntu) \
    mem=768M mmz=ddr,0,<offset>,<size> vmalloc=500M consoleblank=0

`console=tty0` 是 HDMI 出现内核日志与登录提示符的关键。
`consoleblank=0` 关闭内核的控制台自动熄灭（见"已知问题"）。
`mmz` 的 offset 由 fastboot 按 mmz size 算出，两个版本取值不同，详见下文。

### 启用的服务

    multi-user.target : ecoo-fbconsole, ecoo-swap, ecoo-system-init, ssh,
                        systemd-networkd, systemd-resolved, wpa_supplicant,
                        e2scrub_reap, remote-fs.target
    sysinit.target    : ecoo-loopback, haveged, systemd-network-generator,
                        systemd-pstore, systemd-timesyncd
    getty.target      : getty@tty1, serial-getty@ttyAMA0
    timers.target     : apt-daily, dpkg-db-backup, e2scrub_all, fstrim, logrotate

`ecoo-fbconsole` 是在内核尚未支持 fbcon 时用的用户态兜底程序。
本固件的内核已开启 fbcon，该服务启动后会自动检测到并退出，不占资源。

### 软件与网络

预装 `openssh-server`、`curl`、`wget`、`nano`、`vim-tiny`、`cron`、
`ntfs-3g`、`dosfstools`、`fuse3`、`wpasupplicant`、`firmware-realtek`
（常见 USB 无线网卡固件）、`haveged`（熵源）。

网络由 `systemd-networkd` 管理，`/etc/network/interfaces` 未使用。
有线口默认 DHCP。无线可用 `wpa_supplicant`，但需自行配置。

apt 源为官方源，classic 格式写在 `/etc/apt/sources.list`：

    deb http://deb.debian.org/debian trixie main contrib non-free-firmware
    deb http://deb.debian.org/debian trixie-updates main contrib non-free-firmware
    deb http://deb.debian.org/debian-security trixie-security main contrib non-free-firmware

`ca-certificates` 与 `debian-archive-keyring` 已安装，
`/etc/apt/trusted.gpg.d/` 含 trixie 的 stable 与 security keyring，可直接换 https 源。

另有一项网络调优 `/etc/apt/apt.conf.d/99-ecoo-network-tuning`：
`Acquire::ForceIPv4 "true"`，避免这类盒子在只通告 IPv6 却没有实际路由的
路由器后面让 apt 卡在 IPv6 连接上。

### 磁盘与日志

镜像内 rootfs 占 512 MiB，已用约 292 MiB，空闲约 220 MiB。
首次开机会执行一次 `resize2fs` 把文件系统撑满 eMMC 分区，之后可用空间
取决于 eMMC 容量（实测机型可扩到约 6.8 GiB）。

journald 配置为 volatile（`/etc/systemd/journald.conf.d/00-ecoo-volatile.conf`），
日志写 `/run`，不落盘，避免小容量 eMMC 被日志撑满。

### 首次开机做了什么

1. `ecoo-resize-rootfs` 扩展根文件系统（带 360 秒超时保护，单次实测不足 1 秒）
2. `ecoo-firstboot` 推导 hostname、生成 machine-id 与 `/etc/first_init` 标记
3. `ecoo-swap` 创建 512 MiB `/swapfile` 并启用（`swapfile.swap` 在首次开机会
   先报一次 FAILED，属预期——fstab 条目在 `local-fs.target` 之前激活，
   而创建文件的 `ecoo-swap` 在其之后运行，随后会正常 swapon）

### 开机画面

开机 logo 分区显示 Debian 螺旋标志（由 fastboot 阶段显示，随后交给内核接管）：

![开机 logo](image/WIN_20261002_14_15_34_Pro.jpg)

内核接管 HDMI 后，从内核引导日志开始就是原生 TTY 输出，
下面两张是同一块屏上连续的两个画面（注意 swap 已正常启用）：

![HDMI 启动日志 · 前半](image/WIN_20261002_14_15_44_Pro.jpg)

![HDMI 启动日志 · 后半](image/WIN_20261002_14_15_48_Pro.jpg)

---

## 二、版本对比

|                       | TTL-hi3798mv100-32bit-vdec | TTL-hi3798mv100-32bit-maxmem |
| --------------------- | -------------------------- | ---------------------------- |
| `bootargs_768M` 的 mmz | `mmz=ddr,0,0,220M`         | `mmz=ddr,0,0,48M`            |
| fastboot 算出 offset    | 292M                       | 464M                         |
| 内核 CMA 大小             | 220 MiB                    | 48 MiB                       |
| 内核可用内存                | 501.4 MiB                  | 673.7 MiB                    |
| VDEC 硬件解码             | 可用                         | 不可用                          |
| HDMI 原生 TTY           | 正常                         | 正常                           |
| 实测日志                  | PDD119                     | PDD104                       |

除 `bootargs9-32.bin` 外，两个目录里其余 9 个刷机文件 md5 完全相同。

## 三、怎么选

需要硬件视频解码（播放视频、转码、监控回放）选 **vdec** 版。
纯当 Linux 服务器用（SSH、容器、下载、文件服务），用不到 VDEC，
想多留 172 MiB 可用内存，选 **maxmem** 版。

## 四、刷机

进对应目录，用 HiTool 选 `emmc_TTL-hi3798mv100-32.xml` 全量刷写。
该 XML 的 9 行全部 `Sel="1"`，写满 eMMC 的全部 9 个分区。

两版都以 `bootargs9-32.bin` 作为 bootargs 分区的内容（文件名相同，内容不同），
所以分区表 XML 完全通用，不需要区分。

## 五、实测状态

- **vdec 版**：`cma: Reserved 220 MiB at 0x12400000`，
  `VDEC_DRV_Init` 无任何 FATAL，`cma` 无分配失败，
  `Console: switching to colour frame buffer device 240x67`，
  到达 `histb-6ede58ec login:`，HDMI 同步显示同一内容。
- **maxmem 版**：`cma: Reserved 48 MiB at 0x1d000000`，
  `VDEC_AllocPreBuffer` 报 `VdhBufSize:157696000` 分配失败，
  `VDEC_DRV_ModInit: Init drv fail!`，
  其余（fbcon、login、网络）均正常。

## 六、mmz 参数为什么这么算

盒子 DDR 为 768M，fastboot 的 `lib/reserve_mem.c` 会为该档位重写 mmz 的 offset 字段：

    offset = ((ddrsize/3) * 2) - mmz_size = 512M - mmz_size

内核把 mmz 解析为一块 CMA（`phys_start = offset`，`nbytes = mmz_size`），
而 fastboot 自己的预留区下界固定为 249M。于是：

    CMA 大小   = mmz_size
    预留区可用 = 263M - mmz_size      （hifb 的 logo buffer 需 8,298,496 B）

两件事争同一块空间。VDEC 需要 157,696,000 B（10 帧 4K 缓冲）
加上约 29.3 MiB 的既有 CMA 占用，共约 179.7 MiB；且 CMA 分配要求物理连续，
48M 池的最大连续空闲块只有 11.7 MiB。220M 时 CMA 余 40.3 MiB、预留区余 43 MiB，
两侧都够。

## 七、已知问题

### 已修复：开机约 10 分钟后 HDMI 黑屏

**现象**：盒子正常跑着，不接键盘也没有输入，约 10 分钟后 HDMI 输出变成纯黑，
但系统本身仍在运行（SSH、网络、串口都正常）。

**原因**：内核 `drivers/tty/vt/vt.c` 里

    static int blankinterval = 10*60;
    core_param(consoleblank, blankinterval, int, 0444);

默认 10 分钟无键盘活动就熄灭控制台。`vt.c` 里 `if (blankinterval)` 为真时
会启动 `console_timer`，到点置 `console_blanked = 1`，而
`DO_UPDATE(vc)` 宏是 `(CON_IS_VISIBLE(vc) && !console_blanked)`，
一旦 blanked，fbcon 就不再刷新屏幕并调用 `fbcon_blank()` 清屏。

这是所有 Linux 控制台的默认行为，普通服务器上只是"屏幕暗了"，
但在这台盒子上表现为 HDMI 彻底黑屏，容易被误判为驱动故障。

**修复**：在内核命令行加 `consoleblank=0`，使 `blankinterval` 为 0，
定时器根本不启动。本固件的两个版本都已加入该参数。

如果你手上有早于该修复的版本，只需重刷 bootargs 分区即可，不必全量：

    HiTool → 分区表选 emmc_TTL-hi3798mv100-32-bootargs-noblank-vdec.xml
                                         或 ...-bootargs-noblank-maxmem.xml

- **不要热插拔 HDMI。** 厂商的 HDMI hotplug 回调为空，拔掉再插会走到
  `Sink: Deactive` / `PHY Output: Disable`，屏幕不再亮，需重启。开机前插好。
- **全量刷写会重写 `baseparam` 分区**，原厂 MAC 地址可能被覆盖。
  介意的话可只刷 kernel 与 bootargs 两个分区。
- **`swapfile.service` 首次开机 FAILED** 属预期，见上文说明。
- **VDEC 版的可用内存少 172 MiB**，这是换取硬件解码能力的代价。
