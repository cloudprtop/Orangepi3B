# Orange Pi 3B (RK3566) 定制 Armbian 镜像 — 编译 · 测试 · 踩坑记录

给 **Xunlong Orange Pi 3B V2.1**（RK3566）从源码编译 Armbian，并把一批板级问题（风扇不转、网口黄灯不亮、登录状态页消失、IPv6 刷屏、WiFi 固件告警）固化进镜像，最后做一轮全硬件体检与跑分。

- 目标板：Orange Pi 3B **V2.1**（RK3566 / RK809-5 / YT8531 千兆 PHY / AP6256 WiFi5+BT5）
- Armbian：`main` 分支，`26.11.0-trunk`
- 内核：`current` = **6.18.57 / 6.18.58**（mainline rockchip64）
- 发行版：Ubuntu 24.04 `resolute` + Debian 13 `trixie`，均 minimal 无桌面
- 产物：单分区 GPT/ext4 镜像 1.7 GiB → xz 后 366~370 MB

---

## 目录

- [一、编译流程](#一编译流程)
- [二、测试内容](#二测试内容)
- [三、遇到的问题](#三遇到的问题)

---

## 一、编译流程

### 1.1 构建环境

| 项目 | 要求 |
|---|---|
| 宿主系统 | Ubuntu 22.04+ / Debian 12+ x86_64 |
| CPU / 内存 | 建议 ≥ 4 核 / ≥ 8 GB；内存不足时见 1.4 的 `KERNEL_BTF=no` |
| 磁盘 | 建议 ≥ 50 GB 可用（源码 + 内核 + rootfs + 产物） |
| 交叉编译 | amd64 上 cross build arm64，不需要 Docker |

> 官方文档说需要 Debian 13 或 Docker，但 `lib/functions/host/host-release.sh` 的宿主白名单里包含 `jammy`，所以 Ubuntu 22.04 可以**原生**构建，不必套容器。
> 磁盘占用随内核分支和是否保留中间产物变化，只编一个分支时 50 GB 足够。

### 1.2 拉取源码

GitHub 直连在国内常见只有几十 KB/s（191k objects 容易卡在中途），走加速前缀 + 浅克隆：

```bash
git clone --depth 1 https://ghfast.top/https://github.com/armbian/build
cd build
./compile.sh config-dump BOARD=orangepi3b BRANCH=current RELEASE=trixie
```

`config-dump` 用来在动手前确认板级参数，省掉一次几十分钟的空跑：

```
BOARD_TYPE=csc          BOARDFAMILY=rk35xx        LINUXFAMILY=rockchip64
KERNELBRANCH=branch:linux-6.18.y                  KERNELPATCHDIR=archive/rockchip64-6.18
BOOTSOURCE=u-boot v2026.01                        BOOTCONFIG=orangepi-3b-rk3566_defconfig
BOOT_SCENARIO=binman    SERIALCON=ttyS2           OVERLAY_PREFIX=rockchip-rk3566
```

### 1.3 一个开关搞定国内镜像

```bash
REGIONAL_MIRROR=china
```

自动展开为：`MAINLINE_MIRROR=tuna`（内核/固件走 TUNA）、`UBOOT_MIRROR=gitee`、`GITHUB_MIRROR=ghproxy`、`DOWNLOAD_MIRROR=china`、`GHCR_MIRROR=nju`。
实测 `git.kernel.org`（502/超时）、`registry-1.docker.io`（不通）全部绕开，Armbian 自己的包也能命中 `ghcr.nju.edu.cn` 的远程缓存（U-Boot 直接复用，不用自己编）。

### 1.4 完整编译命令

```bash
cd /root/build/armbian-build

nohup ./compile.sh build \
  BOARD=orangepi3b \
  BRANCH=current \
  RELEASE=trixie \
  BUILD_DESKTOP=no \
  BUILD_MINIMAL=yes \
  KERNEL_BTF=no \
  KERNEL_CONFIGURE=no \
  REGIONAL_MIRROR=china \
  COMPRESS_OUTPUTIMAGE=sha,img \
  > /root/build-trixie.log 2>&1 < /dev/null &
```

几个参数的原因：

| 参数 | 作用 |
|---|---|
| `KERNEL_BTF=no` | 关闭 BTF 生成，显著降低构建内存占用。内存 < 8 GB 的构建机建议加，否则可能卡在准备阶段的内存检查（`Not enough RAM ... for BTF build`，exit 43）。只影响 eBPF CO-RE 调试信息，功能无损 |
| `BUILD_MINIMAL=yes` + `BUILD_DESKTOP=no` | 不要桌面，镜像小、启动快 |
| `KERNEL_CONFIGURE=no` | 复用板级默认 `.config`，不弹 menuconfig |
| `COMPRESS_OUTPUTIMAGE=sha,img` | **只出 `.img` + `.sha`，不出 `.xz`** —— xz 要自己压（见 1.7） |

> swap **不能**绕过这个检查（它看的是物理可用内存），只能保证编译过程不 OOM。内存 16 GB 以上可以去掉 `KERNEL_BTF=no`，编出来的镜像会带上 BTF 调试信息。

### 1.5 定制内容怎么进镜像：`userpatches` 两层结构

板级修复分两层，**只改 dtb 或只改 rootfs 都不够**，否则重刷镜像就丢。

```
userpatches/
├── kernel/archive/rockchip64-6.18/          ← 内核补丁（注意：没有 patch/ 这一层）
│   ├── orangepi3b-0001-arm64-dts-...-enable-fan-pwm-and-phy-leds.patch
│   └── orangepi3b-0002-net-phy-motorcomm-add-YT8531-PHY-LED-hw-control.patch
├── overlay/                                 ← 打包进 rootfs 的文件（chroot 内为 /tmp/overlay/）
│   ├── fan-ctl.py
│   ├── fan-ctl.service
│   ├── eth-yellow-led
│   ├── eth-yellow-led.service
│   └── 21-local-ip
└── customize-image.sh                       ← 在 rootfs 内执行的安装脚本
```

**两个内核补丁**

| 补丁 | 内容 |
|---|---|
| `0001-...dts...enable-fan-pwm-and-phy-leds` | `rgmii_phy1` 下加 `leds{}`（`led@0`=activity、`led@1`=lan，`default-state="keep"`）；文件末尾加 `&pwm7 { status = "okay"; };` |
| `0002-...motorcomm-add-YT8531-PHY-LED-hw-control` | 给 YT8531 的 `phy_driver` 补 `.led_hw_is_supported` / `.led_hw_control_set` / `.led_hw_control_get` 三个回调（mainline 里只有 YT8521 / YT8531S 有，唯独 YT8531 漏了） |

**`customize-image.sh`**（只对 `BOARD=orangepi3b` 生效，签名 `$RELEASE $LINUXFAMILY $BOARD $BUILD_DESKTOP $ARCH`）：

1. `chmod 0755 /etc/update-motd.d/*` —— 这是「登录状态页消失」的**根因修复**（打包出来的脚本没有 x 位，`run-parts` 只跑可执行文件）
2. `MOTD_DISABLE="clear ip-info"` —— 干掉每次登录 `curl` 两个公网 IP 造成的 2.3 秒延迟
3. 安装 `21-local-ip` —— 只显示本机 IPv4 + 默认路由接口的一个非 ULA IPv6，零网络请求
4. 安装 `fan-ctl.py` + `fan-ctl.service` 并 `enable`
5. 安装 `eth-yellow-led` + `eth-yellow-led.service` 并 `enable`（写 MDIO 页 `0xA00D = 0x1070`）
6. 补 `brcmfmac43456-sdio.xunlong,orangepi-3b-v2.1.{bin,txt,clm_blob}` 三个软链，消除 dmesg 的 `error -2`

脚本内**零 apt / 零 curl / 零 wget**，全部是文件操作，保证构建可重现、不依赖外网。

> WiFi 的 SSID/密码没有固化（每台设备不同），仍需首启 `armbian-firstlogin` 或手工配。

### 1.6 构建耗时

下表是同一个项目连续三次构建的记录，**主要看相对关系**，绝对值取决于构建机的核数与缓存状态：

| 版本 | 发行版 | 内核 | 耗时 | 说明 |
|---|---|---|---|---|
| 第一版 | resolute 24.04 | 6.18.57 | **158 分 37 秒** | 内核无缓存，全量编译（内核目标文件 13477 个约 100 分钟） |
| 第二版 | resolute 24.04 | 6.18.58 | **74 分 14 秒** | 补丁改动导致内核 artifact hash 变化，内核重编 |
| 第三版 | trixie 13 | 6.18.58 | **8 分 09 秒** | 内核 + rootfs 全部命中远程缓存 |

**结论**：改内核补丁 → 内核全量重编，几十分钟起步；只换发行版且内核不动 → 十分钟内完成。
想省时间，**尽量把内核侧的改动一次改完再编**，别用「编一次验证一处」的方式迭代。

### 1.7 产物交付

`COMPRESS_OUTPUTIMAGE=sha,img` 只给 `.img` 和 `.sha`，xz 需要手工：

```bash
IMG=Armbian-unofficial_26.11.0-trunk_Orangepi3b_trixie_current_6.18.58_minimal.img

# 1. 压缩（1.7G 约 5~12 分钟，-T 按 CPU 核数给）
xz -6 -T0 -k -f output/images/$IMG

# 2. 复制到共享目录并改属主
cp -f output/images/$IMG.xz /home/ctyun/app/
chown ctyun:ctyun /home/ctyun/app/$IMG.xz

# 3. 对"目标文件"重新算 sha（跨文件系统拷贝后必须重算）
cd /home/ctyun/app && sha256sum $IMG.xz > SHA256SUMS.img.xz
```

**校验的正确姿势**：

```bash
xz -t xxx.img.xz                                    # 压缩流完整
xz -dc xxx.img.xz | sha256sum                       # 应等于原始 .img 的 sha256
sha256sum -c SHA256SUMS.img.xz                      # 记录与实际文件一致
```

实测数据：

| 版本 | xz 体积 | 压缩率 | `.img.xz` sha256 |
|---|---|---|---|
| 6.18.57 resolute | 370 MB | 21.5% | `75493e44…0669a9` |
| 6.18.58 resolute | 369 MB | — | `ebeb5492…f86222` |
| 6.18.58 trixie | **366 MB** | — | `17ec1ef7…d89b1a0` |

---

## 二、测试内容

测试分三类：**镜像内静态验收**（不刷机，挂载 rootfs 查）、**板载实测**（刷上真板跑）、**跑分**。

### 2.1 镜像内静态验收（每次编译后必做）

```bash
LOOP=$(losetup -fP --show output/images/xxx.img)
mount ${LOOP}p1 /mnt/vfy     # 注意：单分区镜像，p1 就是 rootfs，没有独立 boot 分区
```

| 检查项 | 命令 | 期望 |
|---|---|---|
| 风扇 PWM 使能 | `dtc -I dtb -O dts /mnt/vfy/boot/dtb/rockchip/rk3566-orangepi-3b-v2.1.dtb \| grep -A3 'pwm@fe6e0030'` | `status = "okay"` |
| PHY LED 节点 | 同上，`grep -c 'led@'` | `2` |
| 温控服务 | `systemctl --root=/mnt/vfy is-enabled fan-ctl.service` | `enabled` |
| 网口灯服务 | `systemctl --root=/mnt/vfy is-enabled eth-yellow-led.service` | `enabled` |
| MOTD 可执行 | `ls -l /mnt/vfy/etc/update-motd.d/21-local-ip` | `0755` |
| MOTD 禁用项 | `grep MOTD_DISABLE /mnt/vfy/etc/default/armbian-motd` | `"clear ip-info"` |
| WiFi 软链 | `ls -l /mnt/vfy/lib/firmware/brcm/ \| grep 43456` | 三条板名软链，目标均存在 |
| 固件干净度 | `ls /mnt/vfy/usr/local/bin/`、`ls -a /mnt/vfy/root/` | 只有业务脚本 + 默认 dotfile |

三个版本的镜像这三类检查全部通过。**这一步不需要刷机，几十秒就能确认，比"编完刷上试试"高效得多。**

### 2.2 CPU

测试条件：`cpufreq governor = performance`，RK3566 四核 A55 @1.8 GHz。

| 项目 | 工具 | 实测 |
|---|---|---|
| 单核 | `sysbench cpu run` | 369 events/s |
| 四核 | `sysbench --threads=4` | 1467.5 events/s（**3.98× 线性**，无降频干扰） |
| 综合 | `7z b` | **6386 MIPS** |
| AES-256 | `openssl speed -evp aes-256-cbc` | 894 MB/s |
| SHA256 | `openssl speed sha256` | 809 MB/s |
| 压力 | `stress-ng --cpu 4 --timeout 60` | 9466 bogo ops/s，0 失败 |

### 2.3 内存

| 项目 | 实测 | 说明 |
|---|---|---|
| 顺序读 | **6.13 GB/s** | |
| 顺序写 | 7.57 GB/s | 写合并生效 |
| 拷贝 | 2.99 GB/s（等效读写 5.97） | |
| 64B 跨步读 | 6.68 GB/s | |

LPDDR4 32-bit @1600 MHz 理论带宽 6.4 GB/s → 顺序读达到 **96%**，内存子系统是满血状态。

### 2.4 GPU

Mali-G52 r1p0（panfrost，594 MHz），**无显示器**下用 EGL surfaceless + 离屏 FBO：

| 项目 | 实测 | 理论/说明 |
|---|---|---|
| 像素填充 | **2.50 Gpix/s** | = 4 px/clk × 594 MHz，**已跑满** |
| 8× 采样填充 | 2.96 Gpix/s | |
| 几何 | 4.4 Mtri/s | |
| draw call | 14382 /s | |

自研 `gpu-bench.c` 会回读像素校验非零，证明是 GPU 真渲染而不是走软渲染。

### 2.5 存储

TF 卡 SC64G（SanDisk 64 GB）：

| 项目 | 实测 |
|---|---|
| 顺序写 | 17.8 MiB/s |
| 顺序读 | **22.8 MiB/s** |
| 4K 随机读 | **2152 IOPS**（QD1 = QD32，SD 无 NCQ） |
| 4K 随机写 | 565 IOPS |

**22.8 MB/s 不是故障**：`mmc@fe2b0000` 在 dts 里只声明了 `cap-sd-highspeed`（50 MHz / 4-bit → 理论 25 MB/s，实测 91%），**没有 `sd-uhs-sdr50/sdr104`**，说明卡槽是纯 3.3 V、不支持 1.8 V 信号切换。要快只能走 eMMC 或 USB3。

### 2.6 网络

| 项目 | 实测 | 达成率 |
|---|---|---|
| 千兆有线（YT8531） | **116.9 MB/s = 935 Mbit/s** | 千兆的 94% |
| WiFi 2.4G（AP6256） | 12.7 MB/s | 150M PHY 的 68% |
| WiFi 5G | 硬件支持（433 Mbit/s 1×1 ac） | 受法规域 DFS 限制，见 3.3.5 |

### 2.7 散热与风扇温控

满载（`yes` ×4）实测：

| 状态 | 温度 | 风扇占空比 |
|---|---|---|
| 满载稳态 | **68.1 °C** | 84% |
| 撤载 30 s | 53 °C | 52% |
| 继续降温 | 50 °C | 40% |
| 45.6 °C | — | **0%（停转）** |
| 空闲稳定 | 44 °C | 0% |

温控曲线（`fan-ctl.py`）：

| CPU 温度 | 占空比 |
|---|---|
| ≤50 °C | 0 |
| 50–55 °C | 0（迟滞区，已转的继续转） |
| **55 °C** | **40%**（起转下限，低于此会堵转） |
| 58 °C | 52% |
| 63 °C | 68% |
| 69 °C | 84% |
| 76 °C+ | 100% |

- PWM 周期 40 000 ns = **25 kHz**，高于人耳可听范围，无啸叫
- 起动瞬间先 100% 吹 1 秒克服堵转，之后落到曲线值
- 采样周期 5 s，滞回 5 °C，避免阈值上反复启停
- 阈值可用环境变量覆盖：`FAN_ON_TEMP=48 FAN_OFF_TEMP=43 fan-ctl.py`

### 2.8 GPIO / USB / 音频 / HDMI

| 项目 | 结果 |
|---|---|
| GPIO | 用板载绿色 LED 做真实输出验证（`--bias` 在本平台无效，见 3.4.6） |
| USB | 7 条总线全在，OTG/Host 5V 使能脚均为高；**无 U 盘，实际速率未测** |
| 音频 | `aplay -l` 可见 card0(HDMI) / card1(RK809)；`arecord -D hw:1,0` 录音后用 Python 算 RMS/峰值 = 5733/32768，麦克风确实收到信号 |
| HDMI | 无显示器：`card0-HDMI-A-1 status=disconnected`、EDID 0 字节、无 modes；dmesg 的 `Cannot find any crtc or sizes` 属正常 |

### 2.9 综合评分

| 维度 | 得分 |
|---|---|
| CPU | 满分区间 |
| 内存 | 满分区间（96% 理论带宽） |
| GPU | 满分区间（填充率跑满） |
| 存储 | 受卡槽限制（SD 3.3V 高速） |
| 网络 | 千兆 94%，WiFi 正常 |
| 散热 | 满载 68 °C，温控闭环正常 |
| **合计** | **938 / 1000**，无硬件故障 |

### 2.10 已部署的测试工具（板载 `/usr/local/bin`）

| 工具 | 用途 |
|---|---|
| `bench-cpu` | CPU / OpenSSL / 7z 一键跑分 |
| `bench-storage` | fio + hdparm 存储测试 |
| `mem-bench` | 自研 C，顺序/跨步带宽 |
| `gpu-bench` | 自研 C，EGL surfaceless + 离屏 FBO 填充/几何测试 |
| `gpio-test` | 真实 GPIO 输出验证 |
| `wifi-scan` | 按 SSID 聚合、按信号排序、标注 2.4G/5G |
| `fan-ctl.py` | 温控守护进程，`--status` / `--set PCT` |

> **这些工具都没有打进镜像**（干净固件要求），是刷机后按需 apt 安装的。

---

## 三、遇到的问题

按类别整理，每条给 **现象 / 根因 / 解决**。带 ⭐ 的是花了最多时间、最值得记的。

### 3.1 编译环境

#### 3.1.1 ⭐ 内核补丁放错目录，白跑 35 分钟

- **现象**：编译日志一切正常，编完挂载镜像发现 `pwm@fe6e0030` 还是 `disabled`，两个补丁一个都没进。
- **根因**：补丁放在了 `userpatches/patch/kernel/archive/rockchip64-6.18/`（多了一层 `patch/`），而这一版 Armbian 实际读的是 `userpatches/kernel/archive/rockchip64-6.18/`。
- **解决**：去掉 `patch/` 层。**判定方法：编译一开始日志就会打印 `[🌱] User patches directory for kernel [ .../userpatches/kernel/archive/rockchip64-6.18 ]`，先看这一行能省下整轮重编。** 补丁是否真生效别信日志，直接查 worktree 源码：

```bash
grep -c 'function = "activity"' arch/arm64/boot/dts/rockchip/rk3566-orangepi-3b-v2.1.dts   # 应为 1
grep -c 'led_hw_is_supported' drivers/net/phy/motorcomm.c                                  # 上游 2 处，打上后 3 处
```

> 层次差异：u-boot 补丁是 `userpatches/u-boot/...`，内核是 `userpatches/kernel/...`，**都没有 `patch/` 这一层**。

#### 3.1.2 `get-pip.py` 拉不下来 / 阿里云镜像版本太旧

- **现象**：`curl bootstrap.pypa.io/get-pip.py` 报 `SSL_read: Connection reset by peer`（约 30 KB/s）；换 `mirrors.aliyun.com/pypi/get-pip.py` 后直接打印 Usage 退出。
- **根因**：① 官方源被 reset；② 阿里云那份是**陈旧版本**，内嵌 pip 太老，不认识 `--break-system-packages` / `--root-user-action`。
- **解决**：用 GitHub 加速前缀拉官方最新版，放到框架期望的路径：

```bash
curl -L https://ghfast.top/https://github.com/pypa/get-pip/raw/main/public/get-pip.py \
  -o /bin/get-pip-26.2.1.py     # 路径来自 PYTHON3_INFO[GET_PIP_BIN]=${USERBASE}/bin/get-pip-<pipver>.py
```

#### 3.1.3 `PIP_INDEX_URL` 环境变量无效

- **现象**：导出 `PIP_INDEX_URL` 后 pip 依然走官方源、依然超时。
- **根因**：框架调用 pip 时用 `env -i`（清空环境）。
- **解决**：写文件 `/etc/pip.conf`：

```ini
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
```

**编译完记得删掉**，否则会改变构建机上其他工具的 pip 行为。

#### 3.1.4 ⭐ `pkill -f 'compile.sh build'` 会把自己杀掉

- **现象**：停编译的命令返回 127 或被信号中断，编译进程还活着。
- **根因**：`pkill -f` 匹配完整命令行，而这条 ssh 命令的命令行里也含 `compile.sh build` 字样，于是匹配到自己。
- **解决**：先拿 PID 再杀：

```bash
pgrep -af 'BOARD=orangepi3b' | grep -v pgrep
kill -TERM <pid>; pkill -KILL -x make
```

**同一个坑在"进度轮询"上又踩了一次**：用 `pgrep -f 'compile.sh build'` 判断构建是否结束，永远返回 RUN（匹配到自己）。轮询应该看**产物文件是否生成**或日志尾行，不要轮询进程名。

#### 3.1.5 `setsid nohup` 启动后台任务，SSH 通道不关

- **现象**：paramiko 报 `PipeTimeout`，以为启动失败。
- **根因**：`setsid nohup ... & disown` 后子进程仍持有 exec 通道的 stdout，通道不关闭。
- **解决**：**任务其实已经启动，别重试**。另开一条 SSH 用 `pgrep` 或看日志确认即可。

#### 3.1.6 镜像 sha256 变了 —— 虚惊一场

- **现象**：`.img` 现算的 sha 与构建时生成的 `.sha` 不一致，一度怀疑镜像损坏。
- **根因**：镜像在算完 sha 之后被**挂载了两次**核对内容，ext4 超级块更新了 `mount count` 和 `last mount time` → 哈希改变。用 `e2fsck -fn` + `cmp` 逐字节比对确认文件系统干净、与源文件完全相同。
- **解决**：`mount count=2` + `last mount time` 与 `.img` mtime 吻合即可判定无害。**校验镜像用「xz 解压流 sha」而不是留存的 `.sha` 文件。**

#### 3.1.7 `COMPRESS_OUTPUTIMAGE=sha,img` 不出 xz

- **现象**：想要 `.img.xz` 但只生成了 `.img` 和 `.sha`，`.txt` 里提到的 `.xz` 文件并不存在。
- **根因**：`sha,img` 只指定这两项，xz 需要额外加 `xz`（或 `all`）。
- **解决**：手工 `xz -6 -T0 -k -f`，然后**复制到目标目录后重新算 sha**（跨文件系统拷贝后必须重算）。

#### 3.1.8 自造的 sha 对照写错了

- **现象**：自己写的部署脚本最后一步 `sha256sum -c` 报 `FAILED`。
- **根因**：把 `.img` 的哈希写进了对照文件，却拿去校验 `.xz` 路径——两个文件本来就不相等。
- **解决**：正确做法是 `xz -dc xxx.img.xz | sha256sum` 与原始 `.img` 的 sha 对比，或者只用 `xz -t` + 目标文件 sha 记录。

### 3.2 设备树与内核

#### 3.2.1 ⭐ 根节点的 `gpio-hog` 永远不生效

- **现象**：在 dtb 根节点下加 `my-hog { gpio-hog; gpios = <...>; output-high; }`，`/proc/device-tree/my-hog/` 里能看到节点，但引脚从不被 claim（`gpioinfo` 仍是 `input`），风扇不转。
- **根因**：mainline `drivers/gpio/gpiolib-of.c: of_gpiochip_scan_gpios()` **只遍历 gpio 控制器节点自己的子节点**。写在根节点上，即使 `compatible="gpio-hog"` 且 phandle 正确，也永远不会被扫描。
- **解决**：持久拉高某个引脚，**用 systemd + `gpioset`**：

```ini
[Service]
Type=simple
ExecStart=/usr/bin/gpioset -c gpiochip0 22=1
Restart=on-failure
```

libgpiod v2 的 `gpioset` 默认**阻塞不退出、持续持有线路**（`gpioinfo` 显示 `output consumer=gpioset`），进程终止才释放。跨重启验证通过。
**判断 hog 是否生效只看 `gpioinfo` 的 consumer，别看好几个地方都"看起来对"。**

#### 3.2.2 ⭐ phandle 找错 GPIO 控制器

- **现象**：hog 指向了错误的控制器，实际控制的是 `gpio4` 的 22 号线，风扇毫无反应。
- **根因**：把 **0x52** 当成了 gpio0。实际上：`gpio@fdd60000`=gpio0(**0x22**)、`fe740000`=gpio1、`fe750000`=gpio2、`fe760000`=gpio3、`fe770000`=gpio4(**0x52**)。
- **解决**：必须读该节点自己的 `phandle = <...>`：

```bash
grep -A12 'gpio@fdd60000 {' rk3566-orangepi-3b-v2.1.dts
```

**不要**用 alias 行（`gpio0 = "/pinctrl/gpio@fdd60000"` 只给路径，不给 phandle）。

> dtb 反编译出来的 dts **没有 label**（dtb 不保存 label），所以引用控制器只能用数字 phandle，`&gpio0` 会报 `Reference to non-existent node or label`；`GPIO_ACTIVE_HIGH` 等宏也没有（无 C 预处理器），直接写数字 `0`。

#### 3.2.3 ⭐ 用 `gpioget` "测量" PWM 引脚，掐断了输出

- **现象**：读引脚电平后发现"PWM 没输出、引脚被拉高"，据此判断错误，白折腾两轮。
- **根因**：任何 `gpioget`/`gpioset`/`gpioinfo` 请求线路都会写 `GPIO_SWPORT_DDR`，**把引脚方向设成输入，直接掐断 PWM 外设输出，而且一直残留到重启**。
- **解决**：不要用 GPIO 工具"测量"复用中的引脚。正确判据是查 pinctrl 归属：

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins | grep 'pin 22'
# pin 22: ... function pwm7 group pwm7-pins
```

或者直接听风扇有没有转。

#### 3.2.4 Rockchip PWM 默认极性是 `inversed`

- **现象**：设 `duty_cycle=0` 风扇反而**满速**。
- **根因**：`polarity=inversed` 时 duty=0 对应引脚全程**高**。
- **解决**：先 `echo normal > /sys/class/pwm/pwmchip0/pwm0/polarity`，duty 才等于高电平时长。

#### 3.2.5 ⭐ `make Image` 不传 `LOCALVERSION` → 75 个模块全加载失败

- **现象**：换掉 `/boot/Image` 重启后系统能起来，但 WiFi、蓝牙、GPU、音频、zram **全部静默失效**，症状极隐蔽（关键驱动都内建，所以系统照样启动）。
- **根因**：源码树是 dirty 的（Armbian patch 流程留下 366 个改动文件），不传 `LOCALVERSION` 就生成 `6.18.57-dirty`，`utsrelease` 与 `/lib/modules/6.18.57-current-rockchip64` 不匹配 → 所有模块加载失败，而 `make` 本身**不报错**。
- **解决**：

```bash
git add -A && git commit -m "wip"    # 先把树弄干净
make -j2 LOCALVERSION=-current-rockchip64 Image modules
```

#### 3.2.6 mainline 里 YT8531 没有 PHY LED 回调

- **现象**：网口链路完全正常（1000 Mb/s 全双工、绿灯正常闪），但**黄灯不亮**；降到 100M 也不亮（排除了"速率灯"解释）。
- **根因**：两层——① dtb 的 `ethernet-phy@1` 节点**没有 `leds` 子节点**；② `drivers/net/phy/motorcomm.c` 里 YT8531 的 `phy_driver` 结构**缺 LED 回调**（YT8521 / YT8531S 都有，唯独 YT8531 漏）。`CONFIG_PHYLIB_LEDS=y` 也没用。
- **解决**：dts 补 `leds{}` + 驱动补三个回调（见 1.5 节）。

#### 3.2.7 ⭐ PHY LED 挂 trigger 后框架空转

- **现象**：`/sys/class/leds/stmmac-0:01::{lan,activity}` 注册成功、trigger 挂上 `stmmac-0:01:link`、`brightness=1`，**灯还是不亮**。读扩展寄存器发现 `LED1_CFG(0xA00D)=0x0620` 仍是出厂默认——trigger 根本没调用 `led_hw_control_set`。
- **解决**：绕过 LED 框架，**直接写 PHY 扩展寄存器**：

```text
寄存器：0xA00D (LED1_CFG)
值：    0x1070 = FDX_ON(1<<12) | ON_10(1<<4) | ON_100(1<<5) | ON_1000(1<<6)
含义：  任意速率、全双工 link 时常亮
访问：  MII 0x1E=页号、0x1F=数据（ioctl SIOCSMIIREG/SIOCGMIIREG，接口 end0，PHY addr 1）
```

参考默认值：`LED0(0xA00C)=0x0610`（活动闪，绿灯）、`LED1=0x0620`、`LED2(0xA00E)=0x0640`。
封装成 `/usr/local/bin/eth-yellow-led` + systemd 服务（`After=network-online.target`），重启自动生效。
> PHY 硬件 LED 亮度**不可调**（`max_brightness=1`），只能改亮/灭/闪烁模式。

#### 3.2.8 70 °C 是 passive 降频点

- **现象**：满载 60 秒后频率在 1800↔1608 MHz 之间摆动，长时间跑分偏低。
- **根因**：thermal trip point 设的是 70/75/95 °C，70 °C 就触发被动降频。**不是过热，是阈值设得低。**
- **解决**：要更稳的持续性能，可把 trip point 抬到 80 °C（改 dtb）；日常使用保持默认即可。

### 3.3 网络与无线

#### 3.3.1 `brcmfmac ... error -2` 不是故障

- **现象**：dmesg 一直刷 `Direct firmware load for brcm/brcmfmac43456-sdio.xunlong,orangepi-3b-v2.1.bin failed with error -2`，据此判断 WiFi 固件缺失。
- **根因**：brcmfmac 的固件加载逻辑是**先找板名专属文件，找不到再回落通用文件**。这里只是缺"板名专属软链"，回落成功（`BCM4345/9 wl0 ... 7.45.96.2`），**WiFi 一直是好的**，缺的是"连网配置"。
- **解决**：补三个软链消除告警（`.bin` / `.txt` / `.clm_blob` → 通用名），补后 dmesg 计数归 0。

#### 3.3.2 Armbian minimal 没有 NetworkManager

- **现象**：照网上教程找 `nmcli`、改 `/etc/network/interfaces`，全都不存在。
- **根因**：Armbian minimal 默认 **systemd-networkd + wpa_supplicant**，没有 NetworkManager、没有 ifupdown（`networking` inactive）。
- **解决**：只能走 wpa_supplicant 模板服务 + networkd `.network` 文件：

```ini
# /etc/wpa_supplicant/wpa_supplicant-wlan0.conf （600 权限）
network={
    ssid="MyAP"
    psk=<PBKDF2-SHA1 哈希>
    country=CN
}
```

```ini
# /etc/systemd/network/25-wlan0.network
[Network]
DHCP=ipv4
LinkLocalAddressing=no
[DHCPv4]
RouteMetric=600
```

`systemctl enable --now wpa_supplicant@wlan0.service`。
PBKDF2 哈希可用 `hashlib.pbkdf2_hmac('sha1', pwd, ssid, 4096, 32).hex()` 算（等价 `wpa_passphrase`），配置文件只存哈希不存明文。
有线 metric 100 + WiFi metric 600 → 双默认路由共存，有线优先，拔线自动切 WiFi。

#### 3.3.3 扫描前必须先 `ip link set wlan0 up`

- **现象**：`iw dev wlan0 scan` 返回空。
- **解决**：接口 down 状态下扫不到任何东西，先 `up` 再扫。

#### 3.3.4 多网卡 `ip -br addr show wlan0 end0` 报错

- **现象**：`either "dev" is duplicate, or "end0" is garbage`。
- **解决**：写成 `ip -br addr show dev end0`，或直接 `ip -br addr`。

#### 3.3.5 ⭐ 5G 频段扫不到 AP —— DFS 信道

- **现象**：`iw phy phy0 info` 明明有 Band 2（5G 共 30 个频点），但 `iw scan` 一个 5G AP 都扫不到。
- **根因**：`country=CN` 下 5150–5350 MHz 属于 **DFS 信道**，brcmfmac 固件的扫描**不会在 DFS 信道上停留**，主动/被动扫描都扫不到 → wpa_supplicant 连"发现"都做不到。
  **对照实验**：`iw reg set US` 后立刻扫出 `XiaoMi_2-5G @5220 MHz(ch44) -20 dBm`；设回 `CN` 或 `00` 全部消失。
- **解决**：CN 域下可用的非 DFS 5G 是 **5725–5850 MHz（ch149/153/157/161/165）** → **把路由器 5G 信道改到 149–165**，这边就能正常扫描连接（合规）。
  改 `country=US` 虽然能用但不合规，不建议。

> 诊断技巧：`iw dev wlan0 scan freq <f>` 对 brcmfmac **无效**（被忽略，返回全频段结果），只能靠 `iw reg set` 做对照来定位 DFS 屏蔽。

### 3.4 用户态与服务

#### 3.4.1 ⭐ 登录状态页消失 = MOTD 脚本没有 x 位

- **现象**：刷完镜像 SSH 登录，只有一行 `Welcome to Armbian`，主机信息、IP、系统状态全没了。
- **根因**：打包出来的 `/etc/update-motd.d/*` **没有可执行位**，而 `run-parts` 只运行可执行文件。不是脚本缺失、不是配置错误。
- **解决**：`customize-image.sh` 里 `chmod 0755 /etc/update-motd.d/*`。

#### 3.4.2 登录慢 2.3 秒 = 每次登录 curl 两个公网 IP

- **现象**：SSH 登录卡 3 秒。
- **根因**：`20-ip-info` 单脚本耗时 **2380 ms**（其余脚本 19~276 ms），因为它每次都 `curl https://ipv4.whatismyip.akamai.com/` 等两个外部服务，各 `--max-time 2`。sshd `UseDNS` 默认关闭，不是元凶。
- **解决**：`MOTD_DISABLE="clear ip-info"` 禁用官方脚本（它头部写着 DO NOT EDIT，升级会覆盖，不要直接改），另写 `21-local-ip` 只读本地网卡信息，零网络请求 → 登录总耗时 3.2 s 降到 1.0 s。

#### 3.4.3 IPv6 一长串 = 多前缀 × 隐私扩展

- **现象**：登录页打印 6 个 IPv6 地址。
- **根因**：`2408:8244:3900:9ba::/64`（运营商 SLAAC + 随机临时地址 + DHCPv6）叠加 `fdb9::/64`（虚拟组网 ULA），Linux 默认开 RFC4941 隐私扩展 → **每个前缀 2 个地址**。
- **解决**：`21-local-ip` 只显示「默认路由出口接口的一个首选 global 地址，且跳过 `fd00::/8` ULA」。
  > 注意：想靠 `HIDE_IP_PATTERN` 过滤 `tun*` 网卡这条路走不通——实测 6 个地址**全在 `end0` 一张物理网卡上**。

#### 3.4.4 风扇停转阈值必须高于空闲温度

- **现象**：空闲时风扇一直 40% 转不停。
- **根因**：阈值定 47 °C，但满载后机壳热透，空闲温度稳在 **47–48 °C**，永远够不到停转条件。
- **解决**：`OFF_TEMP` 47 → **50**，空闲降到 47 °C / duty=0。

#### 3.4.5 libgpiod v2 语法变了

- **现象**：`gpioset gpio0 22=1` 报 `cannot find line 'gpio0'`。
- **解决**：v2 用 `gpioset -c gpiochip0 22=1`。另外 `--hold-period` 单位不是毫秒（设 8000 会把 SSH 卡到超时），别乱用。

#### 3.4.6 `gpioget --bias` 在 RK3566 上是空操作

- **现象**：「上拉/下拉读不同值」的电气自检法完全没反应。
- **根因**：Rockchip 的 `gpio_chip` 没有实现 per-line bias（上下拉归 pinctrl 管）。
- **解决**：该平台不能用 `--bias` 做引脚自检，改用**可观测输出**（绿色 LED、风扇 PWM）做实证。

#### 3.4.7 `/dev/mem` 读 PWM 寄存器是垃圾值

- **现象**：读出来全是 `0x0b`。
- **根因**：`STRICT_DEVMEM` 拦截。
- **解决**：不要拿 `/dev/mem` 当判据。

### 3.5 串口调试

#### 3.5.1 ⭐ 板子"不启动"，其实是 COM 口号变了

- **现象**：刷完新镜像串口全静默，所有波特率（115200/921600/1500000/38400/57600）都是 0 字节，据此判断"镜像坏了/板子没启动"。折腾了一整晚，甚至去验了 idbloader 魔数、FIT magic、分区 UUID、`e2fsck`。
- **根因**：CH340 换了 USB 口，**COM6 重枚举成了 COM7**，一直在监听旧口号。
- **解决**：排查前先重新枚举串口（`serial.tools.list_ports`），**不要沿用历史 COM 号**。改到 COM7 @ **1 500 000** 后立刻抓到完整 firstrun 输出。

> 瑞芯微平台调试串口是 **1 500 000 8N1**，不是 115200。另外用 Python `pyserial` 打开时 `dtr=False / rts=False` 必须显式设置——设成 `True` 会拉住 DTR/RTS，某些板子这两根接 EN/GPIO0，**板子会被一直按住复位**，表现就是"所有波特率全静默"。

---

## 附：关键文件清单

| 文件 | 位置 | 说明 |
|---|---|---|
| `fan-ctl.py` | `/usr/local/bin/` | 风扇温控守护进程 |
| `fan-ctl.service` | `/etc/systemd/system/` | 温控服务，enabled |
| `eth-yellow-led` | `/usr/local/bin/` | 写 PHY 寄存器点亮黄灯 |
| `eth-yellow-led.service` | `/etc/systemd/system/` | enabled，`After=network-online.target` |
| `21-local-ip` | `/etc/update-motd.d/` | 本地 IP 展示，零网络请求 |
| `wpa_supplicant-wlan0.conf` | `/etc/wpa_supplicant/` | 600 权限，只存 PBKDF2 哈希 |
| `25-wlan0.network` | `/etc/systemd/network/` | DHCP + metric 600 |
| `rk3566-orangepi-3b-v2.1.dtb` | `/boot/dtb/rockchip/` | mainline 下必须用 v2.1 这份 |

## 参考

- [Armbian build framework](https://github.com/armbian/build)
- [Orange Pi 3B 用户手册 v1.7](http://www.orangepi.cn/)
- [Linux mainline `drivers/net/phy/motorcomm.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/phy/motorcomm.c)
- [YT8531 数据手册](https://www.motor-comm.com/)（LED1_CFG 等扩展寄存器定义）

---

> 本文所有数值均为 Orange Pi 3B V2.1 实机实测，系统为 Armbian 26.11.0-trunk / kernel 6.18.58-current-rockchip64。跑分绝对值随散热条件和 TF 卡型号会有波动，趋势与达成率参考意义更大。
