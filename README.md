# Actions-ImmortalWrt-Qmodem

[![Build](https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem/actions/workflows/build.yml/badge.svg)](https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem/actions/workflows/build.yml)

基于 ImmortalWrt **v24.10.6** 的 x86_64 精简固件，使用 **QModem stable v3.3.0 + 旧版 Lua 界面**，通过 GitHub Actions 云端编译并输出 EFI 镜像。

> ## AI 构建声明
>
> **本仓库的固件配置（`.config`）、feeds 锁定、GitHub Actions 工作流和本文档均由 AI 助手（Qoder）生成。**
>
> - 构建配置来自对同一套 ImmortalWrt v24.10.6 环境的实际编译验证，QModem 部分参照仓库所有者旧项目 `Actions-ImmortalWrt_5G_modem` 的选择。
> - AI 生成的内容可能存在疏漏；固件未在所有硬件组合上实机验证，尤其是蜂窝模组拨号与短信功能。
> - 使用前请自行核对配置、备份数据，并对刷机风险负责。

---

## 1. 固件概览

| 项目 | 内容 |
| --- | --- |
| 基础版本 | ImmortalWrt v24.10.6（OpenWrt 24.10，Linux 6.6，opkg） |
| 目标平台 | x86_64 Generic，ext4 根文件系统 |
| 镜像类型 | `ext4-combined-efi`，EFI + BIOS 混合可启动 |
| 内核分区 | 32 MiB |
| 根分区 | 1024 MiB（1G） |
| 默认主题 | Argon 2.4.3（首次启动自动设为默认） |
| 包管理 | opkg |
| 系统语言 | 简体中文（基础界面 + 所选应用） |

### 预装组件

| 组件 | 说明 |
| --- | --- |
| **QModem stable v3.3.0** | 蜂窝模组管理与拨号，旧版 Lua 界面（`luci-app-qmodem`） |
| QModem SMS | `luci-app-qmodem-sms` + `sms-forwarder`，短信收发与转发 |
| **daed 1.24.0** | eBPF 透明代理内核与 LuCI 面板（替代 OpenClash） |
| **dockerman（旧版 Lua）0.5.26** | 顶级「Docker」菜单，非 JS 版本 |
| samba4 | SMB 文件共享 |
| frpc | FRP 客户端 |
| argon 主题 | 默认 LuCI 主题 |

### 刻意未包含

- `luci-app-qmodem-next`（JS 主界面）——与旧版主界面互斥，本固件只保留旧版
- `luci-app-qmodem-mwan` / `-ttl` / `-ttlfw4` / `-uplink` / `-voip` / `-monitor`
- `luci-app-openclash`（由 daed 替代）
- `autocore` / `lm-sensors` / `libsensors`（lm-sensors 上游补丁在 GNU make 下失败）

---

## 2. 目录结构

```text
.
├── .config                     # 固件配置（可直接用于 make defconfig）
├── feeds.conf.default          # 锁定的 feed 提交，含 QModem stable
├── .github/workflows/build.yml # GitHub Actions 构建流程
├── .gitignore
└── README.md
```

---

## 3. 使用 GitHub Actions 构建

1. Fork 本仓库（或直接使用本仓库的 Actions）。
2. 打开 **Actions → Build ImmortalWrt 24.10.6 (QModem stable, legacy UI) → Run workflow**。
3. 保持默认版本 `v24.10.6`，点击运行。
4. 首次全量编译在 4 核 GitHub Runner 上实测约 **2 小时**（含工具链、内核、Docker、daed、QModem；不同 Runner 负载会有浮动）。
5. 构建成功后会自动创建 GitHub Release（tag 形如 `v24.10.6-qmodem-legacy-r<run_number>`），并标记为 Latest；可直接从 [Releases](https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem/releases/latest) 下载固件。
6. 该次运行页面的 artifact（`immortalwrt-v24.10.6-qmodem-legacy-efi`）同样保留 30 天，内含：
   - `immortalwrt-x86-64-generic-ext4-combined-efi.img.gz`
   - `sha256sums`
   - `*.manifest`
   - `config.buildinfo` / `version.buildinfo`
   - `build.log`（失败时另含 `logs/` 下的软件包编译日志）

> Release 与 artifact 上传使用仓库自带的 `GITHUB_TOKEN`，工作流已声明 `contents: write` 权限，无需额外配置 Secret。

> Runner 默认磁盘较小，工作流第一步会清理 Android SDK、.NET、CodeQL、hostedtoolcache 等目录以腾出空间，这是正常操作。
> 工作流设置了 350 分钟超时，低于 GitHub 单任务 6 小时上限。
> 构建步骤启用了 `pipefail`，`make` 失败会直接让任务失败，不会出现"日志报错但任务显示成功"的假成功。

### 3.1 最近一次成功构建

| 项目 | 内容 |
| --- | --- |
| Release | [v24.10.6-qmodem-legacy-r3](https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem/releases/tag/v24.10.6-qmodem-legacy-r3)（Latest） |
| Actions Run | [42135909734](https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem/actions/runs/42135909734) |
| 结果 | 成功（约 2 小时），自动发布到 Releases |
| 镜像 | `immortalwrt-x86-64-generic-ext4-combined-efi.img.gz`（168 MB） |
| SHA-256 | `3c8e8837dcf3c0bc7cee927c1ea7f4417148bbee00224f681f1b1bae496b4844` |

产物校验结果：

- GPT 分区：32 MiB FAT16 `kernel` 分区 + 1024 MiB ext4 `rootfs` 分区，另含 BIOS boot 分区（EFI/BIOS 混合启动）
- `kernel` 分区内有 `/boot/vmlinuz`（7.3 MB bzImage）、`/boot/grub/grub.cfg`、`/EFI/BOOT/BOOTX64.EFI`
- rootfs 卷标 `rootfs`，大小恰好 1024 MiB
- 包清单包含 `qmodem 3.3.0`、`luci-app-qmodem 3.3.0`、`luci-app-qmodem-sms 3.3.0`、`sms-forwarder 3.3.0`、`daed 1.24.0`、`luci-app-dockerman 0.5.26`，且不含 `luci-app-qmodem-next` / `sms-forwarder-next`
- `gzip -t` 报 `trailing garbage ignored` 属正常现象：镜像尾部附加了 ucert 签名（官方镜像同样如此）

### 手动触发（命令行）

```sh
gh workflow run build.yml -f version=v24.10.6
gh run list
gh run watch
```

---

## 4. 本地构建

环境：Ubuntu 22.04 / 24.04 x86_64，建议 4 核以上、至少 30 GB 可用磁盘。

### 4.1 安装依赖

```sh
sudo apt-get update
sudo apt-get install -y build-essential clang llvm flex bison g++ gawk \
  gcc-multilib g++-multilib gettext git libncurses-dev libssl-dev \
  python3-setuptools python3-pip rsync unzip zlib1g-dev file wget \
  dwarves libelf-dev bc ccache libtool automake autoconf pkg-config zstd cpio xxd
```

> `dwarves` 提供 `pahole`，daed 需要内核 BTF 信息。
> `clang` / `libelf-dev` 用于 eBPF 工具链；`llvm` 必须一起安装，否则 daed 生成 BPF 时找不到 `llvm-strip` / `opt` / `llvm-dis` / `llc`。

### 4.2 获取源码与 feeds

```sh
git clone --depth 1 --branch v24.10.6 \
  https://github.com/immortalwrt/immortalwrt.git openwrt
cd openwrt

# 使用本仓库锁定的 feeds
git clone https://github.com/liuhrme/Actions-ImmortalWrt-Qmodem.git build-config
cp build-config/feeds.conf.default feeds.conf.default

./scripts/feeds update -a
./scripts/feeds install -a
```

### 4.3 应用配置并编译

```sh
cp build-config/.config .config
make defconfig

# 可选：核对关键选项
grep -E 'CONFIG_PACKAGE_(qmodem|luci-app-qmodem|luci-app-qmodem-sms|sms-forwarder|daed|luci-app-dockerman)=' .config

make -j$(nproc) download
make -j$(nproc)
```

产物位于：

```text
bin/targets/x86/64/immortalwrt-x86-64-generic-ext4-combined-efi.img.gz
```

---

## 5. 配置说明

### 5.1 版本与 feed 锁定

- ImmortalWrt：tag `v24.10.6`
- feeds 固定为 v24.10.6 官方 `feeds.conf.default` 中的提交：
  - packages `97af1398`
  - luci `adb3df4d`
  - routing `ae3329ee`
  - telephony `92892fa2`
- QModem 固定为 **stable 分支提交 `6032d22f`**（版本 3.3.0）

QModem 分支策略：

| 分支 | 定位 | 当前版本 |
| --- | --- | --- |
| `main` | 开发/社区测试线 | v3.4.0_rc3 |
| `stable` | 稳定维护线（仅 hotfix） | **v3.3.0（本固件）** |

### 5.2 QModem 选型

本固件选择 **旧版 Lua 界面**：

- `luci-app-qmodem`：模组信息、拨号概览/配置、AT 调试、高级设置
- `luci-app-qmodem-sms`：短信页面（依赖 `luci-app-qmodem` + `sms-forwarder`）
- `sms-tool_q`、`tom_modem`、`qmodem`：底层工具与守护进程

关键选项：

```text
CONFIG_PACKAGE_luci-app-qmodem_INCLUDE_ndisc6=y
CONFIG_PACKAGE_luci-app-qmodem_INCLUDE_vendor-qmi-wwan=y
CONFIG_PACKAGE_luci-app-qmodem_USE_TOM_CUSTOMIZED_QUECTEL_CM=y
CONFIG_PACKAGE_sms-forwarder=y
# CONFIG_PACKAGE_sms-forwarder-next is not set
```

注意事项：

- `sms-forwarder` 与 `sms-forwarder-next` 互斥，不能同时启用。
- `luci-app-qmodem`（Lua）与 `luci-app-qmodem-next`（JS）是两套主界面，不应同时安装。
- 需要 Quectel 厂商 QMI 驱动时保持 `INCLUDE_vendor-qmi-wwan=y`；通用驱动会缺少部分厂商 AT 能力。

### 5.3 daed 与 BTF

daed 依赖内核 BTF：

```text
CONFIG_KERNEL_DEBUG_INFO=y
CONFIG_KERNEL_DEBUG_INFO_BTF=y
CONFIG_KERNEL_CGROUP_BPF=y
CONFIG_KERNEL_BPF_EVENTS=y
CONFIG_KERNEL_XDP_SOCKETS=y
CONFIG_BPF_TOOLCHAIN_HOST=y
```

构建后可用 `readelf -S vmlinux.elf | grep BTF` 确认。

### 5.4 dockerman 与菜单

本固件使用旧版 Lua dockerman（0.5.26），菜单为顶级 `Docker`，保持上游默认排序（order 40），未做任何排序修改。

---

## 6. 刷写与首次启动

### 6.1 校验镜像

```sh
sha256sum immortalwrt-x86-64-generic-ext4-combined-efi.img.gz
```

与 artifact 内的 `sha256sums` 对比。

### 6.2 写入 U 盘 / 硬盘

Linux：

```sh
gunzip -c immortalwrt-x86-64-generic-ext4-combined-efi.img.gz \
  | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
```

Windows / macOS：使用 balenaEtcher、Rufus（DD 模式）等工具直接写入 `.img.gz` 或解压后的 `.img`。

> 镜像是 EFI + BIOS 混合镜像：UEFI 与 Legacy 引导均可。
> 24.10 镜像的 `.gz` 末尾附带构建签名（ucert），`gzip -t` 会提示 `trailing garbage ignored`，属正常现象，解压数据完整。

### 6.3 首次启动

- 默认主题为 Argon（由 argon 的 uci-defaults 自动设置）。
- 默认管理地址通常为 `192.168.1.1`（ImmortalWrt 默认），如与旧系统冲突请提前确认。
- QModem 位于 `Modem → QModem`，短信位于其下的 `SMS` 页面。
- Docker 为顶级菜单 `Docker`。
- 从旧固件升级时建议 **不保留配置** 刷写，避免旧 UCI/短信数据库结构冲突。

---

## 7. 已知限制

1. **GitHub Runner 资源有限**：4 核 / 16 GB 内存，本次实测全量构建约 2 小时；磁盘通过清理系统目录腾出空间。若超时或磁盘不足，可改用自建 Runner 或本地构建。
2. **未实机验证**：蜂窝模组的拨号、短信、锁频等行为与具体型号、USB 模式、运营商相关，构建成功不代表功能全部可用。
3. **QModem 许可证**：QModem 采用带禁止商业使用条款的 MPL 2.0 衍生许可，使用与分发前请阅读其仓库 `LICENSE`。
4. **CI 缓存**：ccache 首次为空，后续运行会明显加快；缓存上限 2 GB。
5. **固件体积**：daed + Docker + QModem 使镜像约 168 MB（gzip 后），解压后约 1.1 GB。

---

## 8. 常见问题

**Q：为什么不用 `luci-app-qmodem-next`？**
A：本仓库目标是旧版 Lua 界面与 24.10 的旧版 dockerman 保持一致。Next 是纯 JS 界面，官方推荐，但与旧界面互斥。

**Q：为什么 QModem 用 stable 而不是 main？**
A：stable 只接收 hotfix，版本为 v3.3.0；main 为 v3.4.0_rc3 开发测试线。追求稳定选 stable。

**Q：为什么没有 `qmodem-smsd`？**
A：v3.3.0 stable 的短信栈由 `sms-tool_q` + `sms-forwarder` + qmodem 后端组成，旧版短信界面不依赖独立的 smsd 包。

**Q：构建失败在下载阶段？**
A：上游源码偶发 404/超时。工作流会先并行下载，失败后回退单线程详细日志；重跑一次通常可恢复。

**Q：构建失败在 `daed` / BPF 生成阶段？**
A：主机缺少完整 LLVM 工具链。只装 `clang` 不够，daed 生成 BPF 时需要 `llvm-strip`、`opt`、`llvm-dis`、`llc`，请安装 `llvm` 包；工作流已包含该依赖并在失败时上传 `openwrt/logs` 下的软件包编译日志。

---

## 9. 致谢

- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)
- [QModem](https://github.com/FUjr/QModem)（Tom / FUjr）
- [luci-theme-argon](https://github.com/jerrykuku/luci-theme-argon)
- [daed](https://github.com/daeuniverse/daed)
- 原 Actions 配置参考：`liuhrme/Actions-ImmortalWrt_5G_modem`

---

## 10. AI 声明（补充）

本仓库不是手工逐行编写的项目：

- `.config`：AI 根据实际构建记录整理并精简。
- `feeds.conf.default`：AI 依据 ImmortalWrt v24.10.6 官方文件与 QModem stable 提交生成。
- `build.yml`：AI 编写，包含磁盘清理、ccache、下载回退与产物上传。
- `README.md`：AI 编写。

如发现配置错误或构建问题，欢迎提交 Issue 或直接修改后重跑 Actions。
