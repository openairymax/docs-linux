Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agent-linux（AirymaxOS）安装器设计
> **文档定位**：agent-linux（AirymaxOS，极境智能体操作系统）系统安装器工程设计文档，覆盖安装模式、分区方案、引导加载器配置、kickstart 自动化、ISO/QCOW2 镜像构建、介质写入、cloud-init 云定制、安装测试与错误处理\
> **文档版本**：v1.0.3\
> **最后更新**： 2026-10-26\
> **上级文档**：[agent-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 概述

### 1.1 设计目标

agent-linux（AirymaxOS）作为智能体操作系统发行版，其安装器是用户首次接触产品的入口。本文档定义安装器工程设计，聚焦以下目标：

1. **三模式安装**：最小安装（Minimal）/ 完整安装（Full）/ 自定义安装（Expert）三种模式覆盖不同场景
2. **多场景覆盖**：支持裸机安装（ISO）、虚拟机部署（QCOW2）、容器化部署（Docker）、云镜像部署（cloud-init）、PXE 网络安装
3. **kickstart 自动化 100%**：所有安装场景均可通过 kickstart 文件自动化，无人值守完成
4. **分区智能**：根据用户选择自动生成分区方案，支持 Btrfs 子卷、LVM、RAID，预留 PMEM 持久化区域与 CXL 内存区域
5. **引导加载器安全**：GRUB + Secure Boot + TPM 2.0 度量，确保引导链可信

### 1.2 基线约束

本设计严格遵循 Linux 6.6 内核基线，并执行以下硬约束：

| 约束项 | 取值 | 说明 |
|--------|------|------|
| 安装器框架 | Anaconda 衍生 | 主流 Linux 发行版标准 |
| 文件系统 | Btrfs（默认）/ XFS（备选） | Btrfs 支持子卷与快照回滚 |
| 引导加载器 | GRUB 2.06+ | 支持 Secure Boot |
| 内核版本 | Linux 6.6 | agent-linux 1.0.1 锁定基线 |
| 字符串安全 | `strscpy` | 安装器代码中字符串操作必须安全（OS-BAN-004） |

### 1.3 术语规范

本文档严格遵循术语规范：agentrt（用户态运行时）= 微核心（micro-core）；agent-linux（OS 发行版）= 微内核（micro-kernel）。安装器设计独立于 agentrt（[IND] 完全独立层），不与 agentrt 共享安装器代码。文中不使用 主流 Linux 发行版/主流 Linux 发行版 字样，安装器实践对照一律表述为"主流 Linux 发行版"。

### 1.4 安装场景矩阵

| 场景 | 介质 | 自动化方式 | 适用 | 优先级 |
|------|------|------------|------|--------|
| 裸机安装 | ISO | kickstart | 物理服务器 | P0 |
| 虚拟机部署 | QCOW2 | cloud-init | KVM / VMware | P0 |
| 容器化部署 | Docker 镜像 | Dockerfile | K8s / Podman | P0 |
| 云镜像部署 | raw 镜像 | cloud-init | AWS / 阿里云 | P1 |
| PXE 网络安装 | PXE + kickstart | 大规模部署 | 数据中心 | P1 |

### 1.5 安装器组件

```bash
# 安装器主要组件
/usr/lib/airymaxos/installer/
├── anaconda-airymaxos        # 定制 Anaconda 安装器
├── pykickstart-airymaxos     # kickstart 解析器
├── airymaxos-dracut-modules  # dracut 模块（initramfs 生成）
├── airymaxos-iso-builder     # ISO 构建工具
├── airymaxos-qcow2-builder   # QCOW2 构建工具
└── airymaxos-cloud-init      # cloud-init 配置模板
```

---

## 2. 三种安装模式

### 2.1 模式概览

| 模式 | 用途 | 安装大小 | 安装包数 | 安装时间 |
|------|------|----------|----------|----------|
| **最小安装（Minimal）** | 服务器/容器宿主 | 1.2GB | 280 | 8 分钟 |
| **完整安装（Full）** | 桌面/开发工作站 | 4.5GB | 1240 | 22 分钟 |
| **自定义安装（Expert）** | 高级用户 | 可变 | 用户选择 | 可变 |

### 2.2 最小安装包列表

最小安装包含运行 agent-linux 所需的最小包集：

```
最小安装包组：
  - @core                          # 基础系统
  - @base                          # 基础工具
  - airymaxos-kernel               # 内核
  - airymaxos-bootloader           # 引导加载器
  - airymaxos-initramfs            # initramfs
  - airymaxos-services             # 12 个 Daemon（最小子集）
  - airymaxos-security             # Cupolas 安全穹顶
  - airymaxos-system               # 系统编排工具
  - systemd                       # 服务管理
  - dnf                           # 包管理器
  - openssh-server                # SSH 服务
  - NetworkManager                # 网络管理
  - firewalld                     # 防火墙
```

### 2.3 完整安装包列表

完整安装在最小安装基础上增加：

```
完整安装额外包组：
  - @airymax-desktop              # 桌面环境（UKUI 4.x）
  - @development-tools            # 开发工具
  - airymaxos-cognition           # CoreLoopThree 认知运行时
  - airymaxos-memory              # MemoryRovol 记忆卷载
  - airymaxos-cloudnative         # K8s/容器集成
  - airymaxos-tests-linux         # 基准测试套件
  - airymaxos-docs                # 双语文档
  - airymaxos-sdk-python          # Python SDK
  - airymaxos-sdk-go              # Go SDK
  - airymaxos-sdk-rust            # Rust SDK
  - airymaxos-sdk-java            # Java SDK
  - airymaxos-devel               # 开发头文件
  - vim, emacs                    # 编辑器
  - git, make, cmake              # 版本控制与构建
```

---

## 3. 分区方案

### 3.1 默认分区方案（Btrfs 子卷）

agent-linux 默认采用 Btrfs 子卷方案，支持快照回滚：

| 分区 | 挂载点 | 文件系统 | 大小 | 说明 |
|------|--------|----------|------|------|
| `/dev/sda1` | `/boot/efi` | FAT32 | 200MB | EFI 系统分区（UEFI） |
| `/dev/sda2` | `/boot` | XFS | 1GB | 内核与 initramfs |
| `/dev/sda3` | `/` | Btrfs | 剩余空间 | 根分区（含子卷） |

### 3.2 Btrfs 子卷布局

```
/dev/sda3 (Btrfs)
├── @                          # 根子卷 → /
│   ├── usr/
│   ├── etc/
│   ├── var/
│   └── ...
├── @home                      # 用户家目录 → /home
├── @var                       # 可变数据 → /var
├── @opt                       # 第三方软件 → /opt
├── @srv                       # 服务数据 → /srv
├── @tmp                       # 临时文件 → /tmp
└── @snapshots                 # 快照 → /.snapshots
    ├── pre-install/            # 安装前快照
    └── post-install/           # 安装后快照
```

### 3.3 服务器分区方案（LVM + PMEM）

服务器场景推荐 LVM 方案，便于动态调整分区大小，并预留 PMEM 持久化区域：

| 逻辑卷 | 挂载点 | 文件系统 | 大小 |
|--------|--------|----------|------|
| `lv_root` | `/` | XFS | 100GB |
| `lv_home` | `/home` | XFS | 50GB |
| `lv_var` | `/var` | XFS | 200GB |
| `lv_var_lib_agentrt` | `/var/lib/agentrt` | Btrfs | 剩余空间 |
| `lv_swap` | swap | swap | 16GB |
| `lv_log` | `/var/log` | XFS | 20GB |
| `lv_tmp` | `/tmp` | XFS | 20GB |

### 3.4 PMEM 持久化分区

当系统配有 PMEM 设备时，MemoryRovol L1 原始卷独占 PMEM 分区：

| 分区 | 挂载点 | 文件系统 | 大小 | 说明 |
|------|--------|----------|------|------|
| `/dev/pmem0` | `/var/lib/agentrt/memory_rovol/L1_raw` | XFS (DAX) | 800GB | MemoryRovol L1 |

PMEM 与 CXL 自动检测脚本：

```bash
#!/bin/bash
# /usr/lib/airymaxos/installer/scripts/detect_pmem.sh
# 检测 PMEM 设备并配置 DAX 挂载

set -euo pipefail

PMEM_MOUNT=/var/lib/agentrt/memoryrovol/pmem

# 检测 PMEM 区域
for region in $(ndctl list -R 2>/dev/null | jq -r '.[].dev'); do
    echo "[检测] PMEM 区域: $region"
    NS_DEV=$(ndctl create-namespace -r "$region" -m fsdax -f 2>/dev/null \
             | jq -r '.dev' || echo "")

    if [ -n "$NS_DEV" ] && [ -e "/dev/$NS_DEV" ]; then
        echo "[创建] DAX 命名空间: $NS_DEV"
        mkfs.xfs -m dax=inode -f "/dev/$NS_DEV"
        mkdir -p "$PMEM_MOUNT"
        echo "/dev/$NS_DEV $PMEM_MOUNT xfs dax 0 0" >> /etc/fstab
        mount "$PMEM_MOUNT"
    fi
done

# 检测 CXL 内存
for memdev in $(cxl list -M 2>/dev/null | jq -r '.[].memdev'); do
    echo "[检测] CXL 内存设备: $memdev"
done
```

### 3.5 大页（hugepage）分区配置

```bash
# /etc/agentrt/sysctl.d/99-hugepage.conf
# 为 IPC 零拷贝区域预留 2MB 大页

vm.nr_hugepages = 1024

# 启用透明大页
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 大页挂载点
hugetlbfs /var/lib/agentrt/hugepages hugetlbfs mode=1770,gid=1000 0 0
```

---

## 4. 引导加载器配置

### 4.1 GRUB 目录结构

agent-linux 使用 GRUB 2.06+ 作为引导加载器，支持 UEFI 与 BIOS 两种模式：

```
/boot/efi/EFI/airymaxos/
├── grubx64.efi                 # GRUB EFI 主程序
├── grub.cfg                    # GRUB 配置
├── MokManager.efi              # Secure Boot 密钥管理
└── shimx64.efi                 # Secure Boot shim
```

### 4.2 GRUB 配置文件

```bash
# /etc/default/grub —— agent-linux GRUB2 默认配置

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="agent-linux (AirymaxOS)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_TERMINAL="console"
GRUB_CMDLINE_LINUX="crashkernel=auto \
    agentrt.sched_cprime=1 \
    agentrt.memoryrovol=1 \
    agentrt.cxl=1 \
    agentrt.mglru=1 \
    transparent_hugepage=always \
    rd.driver.pre=cxl_acpi,cxl_mem \
    intel_iommu=on iommu=pt \
    loglevel=7"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
GRUB_ENABLE_CRYPTODISK=y
```

安装器不手工维护该文件：anaconda 安装引导装载程序时以 `/etc/system-release` 派生 `GRUB_DISTRIBUTOR`（`sed 's, release .*$,,g' /etc/system-release`）后写入，再调用 `grub-mkconfig`。因此该路径是安装期的硬依赖——`airymaxos-release` 必须同时提供 `/etc/airymaxos-release`（单行 `<NAME> release <VERSION>`，由 `release/os-release` 派生）与指向它的 `/etc/system-release`（上游 release 包已被 `Obsoletes` 排除，无人代为提供）。文件缺失时 `sed` 以非零退出，而 `grub-mkconfig` 以 `set -e` 运行并在开头 source `/etc/default/grub`，整次配置生成随之中止，anaconda 抛 `BootLoaderError` 并停在交互问句上。

### 4.3 GRUB 菜单条目

```bash
# /boot/grub2/grub.cfg（自动生成）

menuentry "agent-linux (AirymaxOS) 1.0.1 (Linux 6.6.0-alk)" {
    set root=(hd0,gpt2)
    linux /boot/vmlinuz-6.6.0-alk \
        root=/dev/mapper/airymaxos--vg-root \
        ro crashkernel=auto \
        agentrt.sched_cprime=1 \
        agentrt.memoryrovol=1 \
        agentrt.cxl=1 \
        agentrt.mglru=1 \
        transparent_hugepage=always \
        rd.driver.pre=cxl_acpi,cxl_mem \
        intel_iommu=on iommu=pt \
        loglevel=7
    initrd /boot/initramfs-6.6.0-alk.img
}

menuentry "agent-linux (AirymaxOS) 1.0.1 (recovery mode)" {
    set root=(hd0,gpt2)
    linux /boot/vmlinuz-6.6.0-alk \
        root=/dev/mapper/airymaxos--vg-root \
        ro single systemd.unit=rescue.target
    initrd /boot/initramfs-6.6.0-alk.img
}
```

### 4.4 Secure Boot 配置

agent-linux 启用 Secure Boot，所有内核模块与 GRUB EFI 程序必须签名：

```bash
# 1. 生成 Secure Boot 密钥
openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Platform Key/" \
    -keyout PK.key -out PK.crt -days 3650 -nodes

openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Key Exchange Key/" \
    -keyout KEK.key -out KEK.crt -days 3650 -nodes

openssl req -new -x509 -newkey rsa:2048 \
    -subj "/CN=AirymaxOS Kernel Signing Key/" \
    -keyout db.key -out db.crt -days 3650 -nodes

# 2. 签名内核
sbsign --key db.key --cert db.crt \
    --output /boot/vmlinuz-6.6.0-alk.signed \
    /boot/vmlinuz-6.6.0-alk

# 3. 注册密钥到 UEFI 固件
mokutil --import PK.crt
mokutil --import KEK.crt
mokutil --import db.crt

# 4. 验证 shim 链
mokutil --list-enrolled | grep "AIRYMAXOS"
```

### 4.5 TPM 2.0 度量

agent-linux 在引导阶段启用 TPM 2.0 度量，记录引导链各阶段哈希：

```bash
# 查看 TPM 度量值
tpm2_pcrread -L sha256:0,1,2,3,4,5,6,7

# PCR 寄存器含义：
# PCR0:  UEFI 固件
# PCR1:  UEFI 配置
# PCR2:  Option ROM
# PCR3:  Option ROM 配置
# PCR4:  引导加载器（GRUB）
# PCR5:  GPT 分区表
# PCR6:  电源状态
# PCR7:  Secure Boot 状态
```

### 4.6 UEFI 品牌化（NVRAM 条目与 ESP 落点）

anaconda 由 `/etc/system-release` 派生 `GRUB_DISTRIBUTOR`，NVRAM 条目因此叫
`openEuler`，固件菜单与 `BootOrder` 都是上游名字。条目标签不能就地改名，
只能在装机段新建一条 `AirymaxOS` 并置顶，实现为
`distro/boot/efi_brand.sh`，做两件事：

1. 在 `${ESP}/EFI/airymaxos` 放一整套已签名的二级加载器
   （`shimx64.efi`/`grubx64.efi`/`mmx64.efi`/`fbx64.efi` 等，按源目录存在项
   复制）与 `BOOTX64.CSV`。shim 只从自身目录取 `grubx64.efi`，签名链也在同
   目录闭合，所以必须是完整一套，不能只放 shim。`grubx64.efi` 内嵌 prefix
   仍指向 `/EFI/openEuler`，生效的菜单配置不变。
2. 用 `efibootmgr -c` 新建条目（先删同名旧条目，避免重装后 `AirymaxOS0` 与
   `BootOrder` 指向旧路径），把新条目插到 `BootOrder` 首位，再回读断言首项。

**时机**：只有装机段可行。默认视图的 chroot 段看不到 ESP 与 NVRAM；首启
systemd 单元被 SELinux 拒绝（`init_t` 在 `dosfs_t` 上 `create`，验收 run
40574831975 的 AVC）。

**执行环境**：目标根而不是安装器环境。`efibootmgr`/`findmnt`/`lsblk` 由
被测系统提供，与真机一致；安装器自带的环境没有这些工具，早期版本直接调用
时脚本走进 skip 分支，而配方对退出码 0 无条件报 OK（验收 run 40587295652
的 ESP 内因此没有 `EFI/airymaxos`）。因此 `%post --nochroot` 段先把
efivarfs、ESP（按目标根 `/etc/fstab` 的 `/boot/efi` 来源挂回）与
`/dev /proc /sys` 挂进目标根，再 `chroot <root>` 执行脚本；段内挂载记入
owned 列表，`trap ... EXIT` 以 `umount -R` 还回，不污染安装器环境。

**判定**（脚本退出码 + 状态文件双重口径）：

| 情况 | 退出码 | 安装段行为 |
|------|--------|------------|
| 非 EFI 装机 | 0（skip） | 打 `AIRYMAX-EFI-SKIP`，继续安装 |
| 此前已品牌化（状态文件存在） | 0（skip） | 同上 |
| ESP 未挂载 / 缺 `efibootmgr` / 无签名 shim | 1 | 打 `AIRYMAX-EFI-FAIL`，`--erroronfail` 终止安装 |
| 退出码 0 但状态文件缺失 | 1（段内断言） | 同上 |

状态文件 `/var/lib/airymaxos/efi-brand` 是「已品牌化」的唯一事实；品牌日志
落 `/var/log/airymax/efi-brand.log`。`--log` 指定的文件在安装器环境，装机
结束即消失，不用于归因。验收侧 `ci/run-smoke.sh` 的 T11 同时断言 ESP 套件、
状态文件与 `BootOrder` 首位，失败诊断附 `state=` 与品牌日志末行。

### 4.7 启动画面

交付机第一次冷启动的观感由两处共同决定，缺任一处都会退回逐行日志或黑屏：

| 位置 | 取值 | 缺了会怎样 |
|------|------|------------|
| kickstart `bootloader --append` | `rhgb quiet` | plymouth 不接管屏幕，内核日志逐行滚动 |
| `airymaxos-release` 的 `/usr/share/plymouth/themes/default.plymouth` 符号链接 | 指向 `tribar/tribar.plymouth` | 落到上游默认 `text` 主题 |

plymouth 按 `plymouthd.conf` → `plymouthd.defaults` → `default.plymouth` 链接
→ `text` 逐级解析主题。前两级连同其所在目录都由 `plymouth` 包持有，第二个
包再声明同名文件会让整个安装事务以 `conflicts between attempted installs`
失败（0.0.3-13 的实测结果，验收 run 40639862403 两条固件腿均停在此处）；
`default.plymouth` 未被上游打包占用，正是发行版声明默认主题的位置。openEuler
的 `plymouthd.defaults` 写着 `spinner`，其包内却没有该主题目录，前两级因此
都落空，链接才是实际生效的那一处。dracut 的 `plymouth-populate-initrd` 走同
一条链，主题随之进入 initramfs；`ShowDelay=0` 也来自该 defaults，启动早期
没有额外的黑屏窗口。构建期 `%check` 断言链接本身与目标名两段：目标拼错时
链接悬空，plymouthd 静默回落 `text`，除观感外不留任何日志痕迹。

`quiet` 只把控制台 loglevel 压到 4：告警以上仍打印，全部记录照样进 journal
与 `ttyS0`，安装测试等待的 `login:` 哨兵来自 agetty，不受影响。按 Esc 或
删掉这两个参数即回到逐行日志，现场排障不必重装。主题缺失或渲染后端不可
用时 plymouth 自行回落 `text`，写错只影响观感，不阻断启动；`plymouth`
包因此写进工作站包清单，而不是靠桌面闭包间接带入。

minimal 变体不加 `rhgb`：它没有画面可换，留下逐行日志更便于取证。

---

## 5. kickstart 自动安装配置

> **0.0.3 交付态（2026-10-18）**：本章 §5.1/§5.2 保留 1.0 目标形态（`cdrom` 源、
> Btrfs/LVM 布局）；0.0.3 实际交付的配方为
> `distro/kickstart/airymaxos-minimal.ks` 与 `airymaxos-workstation.ks`，两者以
> openEuler 24.03 LTS-SP2 官方 `url` 仓库在线组装、`autopart --type=plain
> --fstype=ext4` 收敛首版装机面。口径收敛决策见《0.0.3 极境OS任务方案》
> M-03 实施注记；Btrfs/LVM 随 0.1.x 的 M-14/M-15 再评估。以下示例中的账户、
> 服务与包名以交付配方为准（见 §5.4~§5.6）。

### 5.1 最小安装 kickstart（Btrfs 子卷方案）

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-minimal.ks
# agent-linux (AirymaxOS) 最小安装 kickstart 配置

install
cdrom

lang en_US.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc

network --bootproto=dhcp --device=eth0 --activate
network --hostname=airymaxos-min

rootpw --iscrypted $6$rounds=5000$saltsalt$hashhashhash
user --name=airymax --groups=wheel --shell=/bin/bash \
    --iscrypted --password=$6$rounds=5000$saltsalt$hashhashhash

# 分区方案：Btrfs 子卷
clearpart --all --initlabel --disklabel=gpt
part /boot/efi --fstype=efi --size=200
part /boot --fstype=xfs --size=1024
part btrfs.01 --fstype=btrfs --size=1 --grow

btrfs none --label=airymaxos btrfs.01
btrfs / --subvol --name=@ btrfs.01
btrfs /home --subvol --name=@home btrfs.01
btrfs /var --subvol --name=@var btrfs.01
btrfs /opt --subvol --name=@opt btrfs.01
btrfs /.snapshots --subvol --name=@snapshots btrfs.01

bootloader --location=mbr --boot-drive=sda \
    --append="agentrt.sched_cprime=1 agentrt.memoryrovol=1"

%packages --nobase
@core
@base
airymaxos-kernel
airymaxos-system
%end

%post --log=/root/airymax-post.log
set -eu
systemctl set-default multi-user.target
# 无可登录账户的介质不具备「OS 能运行」的前提：自检失败即中止安装
[ "$(passwd -S root | awk '{print $2}')" = "P" ]
chage -d 0 root
btrfs subvolume snapshot / /.snapshots/post-install/$(date +%Y%m%d)
%end

reboot
```

### 5.2 完整安装 kickstart（LVM 方案）

```kickstart
# /usr/lib/airymaxos/installer/airymaxos-full.ks
# agent-linux (AirymaxOS) 完整安装 kickstart 配置

install
cdrom

lang zh_CN.UTF-8
keyboard us
timezone Asia/Shanghai --isUtc

network --bootproto=dhcp --device=eth0 --activate
network --hostname=airymaxos

rootpw --iscrypted $6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
user --name=airymax --groups=wheel,airymaxos --iscrypted \
    --password=$6$yyyyyyyyyyyy

# 分区方案：LVM + PMEM 预留
clearpart --all --initlabel --disklabel=gpt
part /boot/efi --fstype=efi --size=600
part /boot --fstype=ext4 --size=1024
part pv.01 --size=1 --grow

volgroup airymaxos-vg --pesize=4096 pv.01
logvol / --vgname=airymaxos-vg --name=root --fstype=xfs --size=102400
logvol /home --vgname=airymaxos-vg --name=home --fstype=xfs --size=51200
logvol /var --vgname=airymaxos-vg --name=var --fstype=xfs --size=204800
logvol /var/lib/agentrt --vgname=airymaxos-vg --name=agentrt --fstype=xfs --size=102400
logvol swap --vgname=airymaxos-vg --name=swap --fstype=swap --size=16384

bootloader --location=mbr --append="crashkernel=auto rd.agentrt.sched_cprime=1 rd.agentrt.memoryrovol=1"

%packages
@airymaxos-base
@airymaxos-kernel
@airymaxos-services
@airymaxos-cognition
@airymaxos-memory
@airymaxos-security
@airymaxos-system
airymaxos-kernel-core
airymaxos-kernel-modules
airymaxos-kernel-tools
airymaxos-cognition-core
airymaxos-memory-core
airymaxos-security-cupolas
airymaxos-sdk-python
airymaxos-sdk-rust
airymaxos-sdk-go
airymaxos-sdk-java
# CJK 字体与输入法
airymaxos-fonts-cjk-zh-cn
airymaxos-fonts-cjk-ja
ibus
ibus-libpinyin
-exclude
-kernel-debug
%end

%post --log=/root/airymax-post.log
set -eu

# 启用 12 个 daemon。0.0.3 不执行此段：二进制随 airymaxos-services 交付，
# 提前启用会让冷启动进入 degraded（见 §5.6 与 M-17）。
for svc in gateway cogn dev sched audit net logger sec mem vfs config \
           macro-superv; do
    systemctl enable "agentrt-${svc}.service"
done

# 配置默认 locale 与时区
localectl set-locale LANG=zh_CN.UTF-8
timedatectl set-timezone Asia/Shanghai

# 启用 MemoryRovol PMEM 挂载点
mkdir -p /var/lib/agentrt/memoryrovol/pmem
if [ -e /dev/pmem0 ]; then
    mkfs.xfs -m dax=inode -f /dev/pmem0
    echo "/dev/pmem0 /var/lib/agentrt/memoryrovol/pmem xfs dax 0 0" >> /etc/fstab
fi

# 启用 MGLRU 多代 LRU
echo "y" > /sys/kernel/mm/lru_gen/enabled
echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms

# 配置大页
echo "vm.nr_hugepages = 1024" > /etc/sysctl.d/99-airymaxos-hugepage.conf
echo "hugetlbfs /var/lib/agentrt/hugepages hugetlbfs mode=1770,gid=1000 0 0" >> /etc/fstab
mkdir -p /var/lib/agentrt/hugepages

%end

reboot
```

### 5.3 PMEM 专用 kickstart 片段

当系统配有 PMEM 设备时，kickstart 须包含 PMEM 分区配置：

```bash
# PMEM 分区配置片段
%post

PMEM_DEV=$(ndctl list -R | jq -r '.[0].dev' | sed 's/namespace/pmem/')

if [ -n "$PMEM_DEV" ]; then
    mkfs.xfs -m dax=inode /dev/${PMEM_DEV}p1
    mkdir -p /var/lib/agentrt/memory_rovol/L1_raw
    echo "/dev/${PMEM_DEV}p1 /var/lib/agentrt/memory_rovol/L1_raw xfs dax 0 0" \
        >> /etc/fstab
    mount /var/lib/agentrt/memory_rovol/L1_raw
    xfs_io -c "statx -v" /var/lib/agentrt/memory_rovol/L1_raw \
        | grep STATX_ATTR_DAX
fi
%end
```

### 5.4 账户与初始凭据（SSoT）

0.0.3 的无人值守安装没有交互建号环节，账户必须在安装结束后立即可用，否则
控制台与 UKUI 桌面都没有入口。账户契约的唯一来源是
`distro/kickstart/airymaxos-minimal.ks` 与 `airymaxos-workstation.ks`，两剖面
保持一致：

| 项目 | 取值 | 约束 |
|------|------|------|
| root 口令 | SHA-512 crypt 哈希 + `--iscrypted` | 禁止明文；哈希常量后缀 `0x01` 标识初始代次 |
| root 首次登录 | `chage -d 0 root` | 安装后强制改密，仅用于首次控制台接管 |
| 初始用户 | `airymax`，`wheel` 组 | lightdm/UKUI 的登录主体，口令同样以哈希落盘 |
| 账户自检 | `%post` 中 `acct_ok <user>`（`passwd -S` 状态必须为 `P`） | 失败即 `fail <token>`，串口标记带 `reason=` 归因 |

初始口令哈希是仓库内的唯一凭据载体，明文不入库、不入文档。轮换初始凭据时
同时改写两份配方并递增哈希盐后缀，避免两剖面漂移。生产环境应在首次登录后
立即改密；root 的过期策略由 `chage -d 0` 强制。

### 5.5 工作站图形会话（lightdm + UKUI）

工作站剖面的验收判据是「会话定义 / lightdm 单元 / 登录会话」三级登录链成立。
第三级按座位策略取并集：启用 autologin 时 lightdm 不创建 greeter，直接以
`lightdm-autologin` 拉起 `Class=user` 的 X11 会话；未启用 autologin 才停在
`Class=greeter`。交付链路：

1. `%post` 以 `systemctl --root=/ set-default graphical.target` 设定默认目标，判据是
   `/etc/systemd/system/default.target` 的符号链接落盘状态（安装器 chroot 内没有
   systemd 总线，见 §9.3.9）；
2. kickstart `services --enabled=lightdm airymax-desktop-ready` 显式启用显示
   管理器与就绪探针，包自身 `%post` 不做启用（OS-OPS-010 的同一条原则）；
3. `airymaxos-release` 安装 `/etc/lightdm/lightdm.conf.d/50-airymax.conf`，以
   `[Seat:*] user-session=ukui` 指定会话；`ukui-greeter` 自带
   `95-ukui-greeter.conf` 接入 lightdm；
4. `airymax-desktop-ready.service`（`Type=oneshot`，
   `ConditionFileIsExecutable=/usr/sbin/lightdm`，`After=`/`Wants=lightdm.service`，
   `TimeoutStartSec=180`，`WantedBy=graphical.target`）运行 `desktop_ready.sh`：
   先静态校验 `/usr/share/xsessions/ukui.desktop` 存在且其 `Exec` 可解析
   可执行，随后在 90 秒内每 2 秒轮询，要求 `lightdm` 处于 active 且存在
   `Service=lightdm*`、`Class∈{greeter,user}`、`Type=x11`、
   `State∈{active,online}` 的 logind 会话，成功打印
   `AIRYMAX-DESKTOP-READY`；会话定义不成立立即打印
   `AIRYMAX-DESKTOP-FAIL reason=session`，等待超时打印
   `AIRYMAX-DESKTOP-FAIL unit=lightdm wait=<n>s`。探针只判定「会话已建立」，
   桌面进程是否存活由 `run-smoke.sh` T1 单独取证，不设第二事实源。两种失败串均由
   `install-test.sh` 的 `BOOT_FAIL` 规则即时捕获；
5. 桌面自带 AI 控制面（`kylin-ai*`）在 `%packages` 中通配排除，agent 控制面
   唯一入口保持为 Gateway（R1-06 口径）。

该探针失败会让单元进入 failed，从而使镜像 CI 阻断——这是有意的验收语义，
不是可忽略的告警。桌面资产的安装归属、六条集成铁律与冻结点核对表见
`distro/desktop/README.md`。

### 5.6 0.0.3 的启用边界（M-17）

`agentrt.target` 通过 `Requires=` 聚合 12 个 daemon 与 `agentrt-firstboot.service`。
0.0.3 只交付 systemd 单元与运行时脚本，12 个 `/usr/sbin/*_d` 随 0.1.x 的
`airymaxos-services` 交付，首启依赖的 LSM 就绪探针同样尚未闭环。此时若启用
任一单元，冷启动会因 `ExecStart` 缺失或探针未就绪进入 degraded，违背
「OS 能正常运行」的验收口径。

因此：

- `distro/presets/90-airymax.preset` 以 `disable` 钉住 `agentrt.target`、
  `agentrt-dsl.target` 与 `agentrt-*.service`，堵住 `systemctl preset-all`
  对含 `[Install]` 单元的默认 enable 语义（共 16 个单元；
  `airy-incident@.service` 无 `[Install]`，不列入）；
- `airymaxos-system` 的 `%post` 不调用 `systemctl preset`，安装器与管理员是
  唯一的显式启用入口；
- 工作站 kickstart 的 `%post` 不启用 `agentrt.target`，也不调用 `daemon-reload`
  （该动词依赖总线，对尚未启动的已安装系统没有意义）；OS-OPS-010 的启用动作随
  `airymaxos-services` 同版本落地；
- 启用边界由 `distro/tools/preset_lint.py`（P001~P006）静态校验，并纳入
  `distro/ci/run-lint.sh` 统一门禁。

---

## 6. 安装流程

### 6.1 安装流程图

```
┌───────────────────────────────────────────────────────────────┐
│ 1. 引导加载（BIOS/UEFI）                                       │
│    └→ 加载 airymaxos-installer kernel + initramfs             │
├───────────────────────────────────────────────────────────────┤
│ 2. Anaconda 启动                                              │
│    └→ 读取 kickstart 配置（无人值守）/ 进入 TUI（交互式）      │
├───────────────────────────────────────────────────────────────┤
│ 3. 硬件检测                                                   │
│    └→ CPU / 内存 / 磁盘 / NIC / NUMA / CXL / PMEM 检测        │
├───────────────────────────────────────────────────────────────┤
│ 4. 选择安装模式                                               │
│    └→ 最小 / 完整 / 自定义                                    │
├───────────────────────────────────────────────────────────────┤
│ 5. 分区与文件系统                                             │
│    └→ 创建 Btrfs 子卷 / LVM + PMEM DAX 挂载点                │
├───────────────────────────────────────────────────────────────┤
│ 6. RPM 包安装                                                │
│    └→ dnf install --installroot=/mnt/sysimage airymaxos-*     │
├───────────────────────────────────────────────────────────────┤
│ 7. 引导加载器安装                                             │
│    └→ grub2-install + grub2-mkconfig + Secure Boot 签名       │
├───────────────────────────────────────────────────────────────┤
│ 8. 后置配置                                                   │
│    └→ locale + 时区 + 用户 + 网络 + sched_tac 用户态调度器 + 快照 │
├───────────────────────────────────────────────────────────────┤
│ 9. 重启进入系统                                              │
│    └→ 加载 airymaxos-kernel + 启动 systemd + 启动 12 daemon   │
└───────────────────────────────────────────────────────────────┘
```

### 6.2 安装日志

安装日志位于 `/var/log/agentrt-install.log`：

```
2026-07-09 10:00:00 [INFO] agent-linux 安装器启动
2026-07-09 10:00:01 [INFO] 安装模式: 完整安装
2026-07-09 10:00:02 [INFO] 分区方案: Btrfs 子卷
2026-07-09 10:00:05 [INFO] 网络: DHCP, eth0
2026-07-09 10:00:10 [INFO] 用户配置完成
2026-07-09 10:00:15 [INFO] 开始安装包（1240 个）
2026-07-09 10:08:30 [INFO] 包安装完成
2026-07-09 10:08:35 [INFO] GRUB 配置完成
2026-07-09 10:08:40 [INFO] 服务启用: gateway_d, cogn_d, ...
2026-07-09 10:08:45 [INFO] dnf 仓库配置完成
2026-07-09 10:08:50 [INFO] MGLRU 配置完成
2026-07-09 10:08:55 [INFO] 安装后快照创建完成
2026-07-09 10:09:00 [INFO] agent-linux 完整安装完成
```

---

## 7. ISO 与 QCOW2 镜像构建

### 7.1 ISO 构建脚本

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_iso.sh
# 构建 agent-linux 安装 ISO 镜像

set -euo pipefail
VERSION=${1:-1.0.1}
ARCH=${2:-x86_64}
WORK_DIR=/var/tmp/airymaxos-iso
ISO_NAME=airymaxos-${VERSION}-${ARCH}.iso

echo "[1/6] 准备工作目录"
rm -rf "$WORK_DIR"
mkdir -p "$WORK_DIR"/{LiveOS,isolinux,images,Packages,repodata,EFI/BOOT}

echo "[2/6] 复制内核与 initramfs"
cp /boot/vmlinuz-6.6.0-alk "$WORK_DIR/isolinux/vmlinuz"
cp /boot/initramfs-6.6.0-alk.img "$WORK_DIR/isolinux/initrd.img"

echo "[3/6] 复制 kickstart 与 GRUB EFI"
cp /usr/lib/airymaxos/installer/airymaxos-default.ks "$WORK_DIR/isolinux/ks.cfg"
cp /boot/efi/EFI/airymaxos/grubx64.efi "$WORK_DIR/EFI/BOOT/BOOTX64.EFI"

echo "[4/6] 创建 isolinux 配置"
cat > "$WORK_DIR/isolinux/isolinux.cfg" <<EOF
DEFAULT airymaxos
PROMPT 0
TIMEOUT 30

LABEL airymaxos
    MENU LABEL agent-linux (AirymaxOS) ${VERSION}
    KERNEL /isolinux/vmlinuz
    APPEND initrd=/isolinux/initrd.img \
        inst.stage2=hd:LABEL=AIRYMAXOS \
        inst.ks=hd:LABEL=AIRYMAXOS:/isolinux/ks.cfg \
        agentrt.memoryrovol=1 airy.sec_d=1
EOF

echo "[5/6] 复制 RPM 包并生成仓库"
find /root/rpmbuild/RPMS -name "*.rpm" -exec cp {} "$WORK_DIR/Packages/" \;
createrepo_c "$WORK_DIR/Packages/"
cp /etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos "$WORK_DIR/"

echo "[6/6] 生成 ISO 镜像"
mkisofs -o "$ISO_NAME" \
    -b isolinux/isolinux.bin \
    -c isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot \
    -e EFI/BOOT/BOOTX64.EFI \
    -no-emul-boot \
    -J -R -V AIRYMAXOS -T \
    "$WORK_DIR/"

implantisomd5 "$ISO_NAME"
sha256sum "$ISO_NAME" > "${ISO_NAME}.sha256"

# GPG 签名 ISO
gpg --detach-sign --armor -u "iso-sign@airymaxos.dev" "$ISO_NAME"

echo "[完成] ISO 镜像已生成: $ISO_NAME"
```

### 7.2 QCOW2 镜像构建

```bash
#!/bin/bash
# /usr/lib/airymaxos/scripts/build_qcow2.sh
# 构建 agent-linux 虚拟机 QCOW2 镜像

set -euo pipefail
VERSION=${1:-1.0.1}
ARCH=${2:-x86_64}
IMG_NAME=airymaxos-${VERSION}-${ARCH}.qcow2
SIZE=20G

echo "[1/4] 创建 QCOW2 镜像（${SIZE}）"
qemu-img create -f qcow2 "$IMG_NAME" "$SIZE"

echo "[2/4] 启动虚拟机执行 kickstart 安装"
qemu-system-${ARCH} \
    -m 4096 -smp 4 \
    -drive file="$IMG_NAME",format=qcow2,if=virtio \
    -cdrom airymaxos-${VERSION}-${ARCH}.iso \
    -kernel /boot/vmlinuz-6.6.0-alk \
    -initrd /boot/initramfs-6.6.0-alk.img \
    -append "inst.ks=file:/isolinux/ks.cfg \
             console=ttyS0 \
             agentrt.sched_cprime=1" \
    -nographic \
    -serial mon:stdio

echo "[3/4] 安装 cloud-init（用于云镜像定制）"
virt-customize -a "$IMG_NAME" --install cloud-init

echo "[4/4] 压缩镜像"
qemu-img convert -O qcow2 -c "$IMG_NAME" "${IMG_NAME}.compressed"
mv "${IMG_NAME}.compressed" "$IMG_NAME"

echo "[完成] QCOW2 镜像已生成: $IMG_NAME"
qemu-img info "$IMG_NAME"
```

### 7.3 介质写入（烧录）

安装介质从 ISO 到 U 盘的这一段不属于「安装」，但它是整条交付链上唯一
由人手工接触块设备的环节，也是最容易把系统盘写掉的一步。本节只定设计，
操作步骤的唯一权威源是 `agent-linux` 仓的 `tools/airy-flash/README.md`。

#### 7.3.1 自研工具还是用 `dd`

结论：**自研一个薄层，不引入第三方刻录器**。理由是发行物本身已经解决
了「怎么引导」，缺的是「这台设备能不能写、这个包是不是发布物、写完是否
逐字节一致、过程能否复盘」这四件事，而它们恰好都是有明确判据的判定：

| 备选 | 能解决 | 不解决 | 取舍 |
|---|---|---|---|
| `dd` 手写 | 顺序写入 | 写错盘、包不对、写完不知成没成、无记录 | 作为底层实现被封装，不作为操作入口 |
| Rufus / balenaEtcher 等 GUI | 介质发现与写入 | 无 `--json`、无退出码契约、不可在无人值守轮次复现，且引入一个不受本仓审计的二进制 | 仅在非 Linux 平台作为等价手段出现在对照表里 |
| 自研 `airy-flash` | 上述四项 + 机器可读结果 | 不重造分区与引导 | 采用 |

自研的边界写在工具入口注释里：**ISO 由 `mkksiso` 保留 isohybrid 分区表与
EFI 镜像，顺序写入即可双固件引导**。再加一套分区/引导引擎等于引入第二处
事实源，与 SSoT 原则冲突。

#### 7.3.2 可复现烧录的取证链

「可复现」在这里不是「每次烧出来的 U 盘字节相同」——那是 `sha256` 保证的
——而是**每块盘都能指回一次确定的流水线产出**。链条上的每一环都可独立验证：

| 环节 | 证据 | 由谁保证 |
|---|---|---|
| 源字节 | ISO 的 `sha256` | `image-build.yml` 产 `.sha256` |
| 取件无损 | `.sha256` 校验通过 | `sha256sum -c` |
| 取件同源 | `iso-oci.txt` 的 `sha256=` 等于本地 `.sha256` | `ci/ci.sh iso-publish` 写入 |
| registry 无损 | OCI layer digest 等于文件 `sha256` | `ci/gates/iso-publish.sh` 远端断言 |
| 写入正确 | 回读窗口逐字节一致 | `airy-flash` 的 `verify` 阶段 |
| 装机可用 | `AIRYMAX-SMOKE-PASS` / `AIRYMAX-DESKTOP-READY` | `ci/ci.sh accept --firmware both` |

任一环断裂即中止，不做「先烧了再说」。摘要来源缺失时工具默认拒绝写入，
`--allow-unverified` 只在调试时显式使用，且报告里该阶段标 `[SKIP]`。

取件通道不属于取证链的一环。同一 tag 的清单在任一 registry 镜像上都指向同
一个 layer digest，带宽差异只影响等多久，不影响取回的是什么，因此允许按可达
性择优（先试地区镜像，失败回落 `ghcr.io`），判据始终是「层摘要等于发布值」。
实测（B50，2026-10-28）直连 `ghcr.io` 为 21 KB/s（3.76 GB 需 49 小时），经
`ghcr.nju.edu.cn` 为 9.8 MB/s（6 分钟），两者清单里的 digest 完全相同，
落盘后 `sha256sum -c` 一致。把这条写进流程的意义是：出口带宽不可控时，
可复现性不跟着一起丢。

#### 7.3.3 与验收的分工

`airy-flash` 证明「盘上的字节对」，不证明「系统起得来」。两者不重叠、
也不能互相替代：

- 引导与桌面：QEMU 上以 `--firmware both` 跑 BIOS / UEFI 双固件冷启动，
  产品 ISO 另断言 `AIRYMAX-DESKTOP-READY`。
- 工具自身：`tools/airy-flash/tests/` 在 `python-verify` 门禁里跑，用临时
  目录合成 sysfs 树与挂载表，不需要 root 与真实块设备。
- 真机：作为验收的最后一跳（`acceptance-report` 的真机小节），产出
  `flash-report.json` 与回读结论，随发行记录归档。

#### 7.3.4 通用化约束

工具不含验证机的分支：任何在 Linux 上以块设备出现的介质（U 盘、SD 卡、
USB 硬盘盒、NVMe 转接盘）走同一套预检与写入路径，节点名不做假设。系统盘
由 `/proc/mounts` 反查承载 `/`、`/boot`、`/boot/efi`、`/usr`、`/var` 的磁盘
得到，而不是靠 `sdX` 命名或设备型号猜测——这条约束保证换一台机器、换一种
介质，判定逻辑不变。非 Linux 平台不提供桩实现，只给出用平台原生工具复现
同一顺序的对照表，见 `tools/airy-flash/README.md`。

#### 7.3.5 装机收尾与首次启动

写入正确只说明介质可用，能否进桌面还要看安装器收尾这一段。安装配方末行是
`reboot`，但「安装器请求重启」与「机器真的重启」是两件事：B50 实机
（2026-10-28）上 anaconda 收尾落到 `reboot: System halted`，机器停住 37
分钟，直到人工断电重上电。由此定下三条口径：

- 无人值守轮次不把「自重启」当作流程节点。串口终态出现即视为安装结束，
  后续引导由外部动作驱动：`efibootmgr -n <条目号>` 指定一次性启动项，再
  上电。用 `-n` 而不是改 `BootOrder`，是为了让写坏 `BootOrder` 时也不至于
  连同原系统一起锁死。
- 品牌化脚本在装机段把 `AirymaxOS` 条目置顶（§4.6），因此正常路径上一次性
  启动项与默认启动项指向同一条；`-n` 只是把「本次一定进安装器」这件事从
  固件默认值里摘出来。
- `halt` 与 `reboot` 只差一次 ACPI 事件，事后区分二者只能靠日志，因此持久
  日志是这条流程的前置条件而不是可选项：`airymaxos-release` 交付
  `/etc/systemd/journald.conf.d/50-airymax-journald.conf`（`Storage=persistent`），
  `journalctl -b -1` 才能回答「上一次启动停在哪」。

#### 7.4 站点派生介质（保留既有系统的装机）

产品 ISO 的分区策略是 `clearpart --all`，整盘归 AirymaxOS，这是发行物的正确
语义。但验证机与工作站常已有别的系统（例如前两个分区属于 Ubuntu），此时需要
的是「同一份字节，只改分区作用域」，而不是给产品 ISO 加一个分支开关。

做法是派生一份**站点介质**：除 `/ks.cfg` 外不改动任何字节。

```
xorriso -indev product.iso -outdev site.iso \
        -map site.ks /ks.cfg -boot_image any keep
truncate -s "$(stat -c %s product.iso)" site.iso   # 补回原长度，见下
```

不用 `mkksiso`：它会在产品 ISO 的每条 `append`/`linuxefi` 行尾再追加一个
`inst.ks=`，与产品 ISO 已有的同名键并存；重复键的解析结果在构建侧无法证伪，
且改动面不可逐字节复核。`xorriso -map` 只替换一个成员，卷标、分区项（含 ESP
的 `0xef` 项）与两个引导配置保持原样，派生后可挂载比对 `/ks.cfg` 与源逐字节
一致。

`truncate` 不是可有可无的收尾。isohybrid 镜像的 MBR/GPT 尾部项按输入长度
布局，而 `-outdev` 重新打包会丢掉末尾的填充扇区：2026-10-28 实测派生结果比
产品 ISO 短 1.98 MB（3755.9 MB vs 3763.8 MB），并触发 `libburn: SORRY: Read
start address ... larger than number of readable blocks`——被切掉的正是 ESP
所在的那一段。文件短了，`xorriso` 只是「容忍」这个问题，整盘写入后固件读
不到完整的 EFI 分区，介质直接不可引导。因此派生后一律补回原长度，`xorriso`
的 `SORRY` 消退、回读校验返回 0 才算通过。

站点 ks 的硬约束：

- `clearpart --none` 关闭整盘清除，每个分区行用 `--onpart=<disk>pN` 显式绑定，
  并加 `ignoredisk --only-use=<disk>` 把安装器能看到的盘收敛到目标盘；盘名与
  分区号是派生工具的显式输入，不由安装器猜。
- 派生之前先在机器上核对一遍：`lsblk` 的四个目标分区用途与容量、`blkid` 的
  标签与文件系统类型都要与输入一致，对不上就不派生。生成期的核对比装机期的
  失败更便宜——后者要等一个完整的安装事务跑完。
- 每个系统保留各自的 ESP，站点配方只写自己的那一个；向既有 ESP 叠加第二套
  引导会与固件里的既有启动项争用同一目录。
- 站点配方同样受 `ks_lint` 门禁约束（`%post` 首行 `set -eu` 等），派生后先跑
  门禁再写盘。

定位：站点介质是**装机现场产物**，不是发行物——它不进 `ghcr.io`、不入库，
也不作为验收证据；验收与发行使用的始终是流水线产出的原始 ISO。这条边界保证
「产品字节」与「某台机器的装法」不被混为一谈。

---

## 8. cloud-init 云镜像定制

### 8.1 默认 cloud-init 配置

```yaml
# /etc/cloud/cloud.cfg.d/99-airymaxos.cfg
# agent-linux cloud-init 配置

system_info:
  default_user:
    name: airymax
    groups: [wheel, airymaxos]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash

cloud_init_modules:
  - seed_random
  - bootcmd
  - write_files
  - growpart
  - resizefs
  - disk_setup
  - mounts
  - set_hostname
  - update_hostname
  - update_etc_hosts
  - rsyslog
  - users_groups
  - ssh

cloud_config_modules:
  - locale
  - set_passwords
  - ssh-import-id
  - package_update_upgrade_install
  - timezone
  - runcmd

locale: zh_CN.UTF-8
timezone: Asia/Shanghai

packages:
  - airymaxos-services-gateway
  - airymaxos-services-cogn
  - airymaxos-services-dev
  - airymaxos-cognition-core

runcmd:
  - systemctl enable agentrt-kernel-load-bpf.service
  - systemctl enable agentrt-gateway.service
  - systemctl enable agentrt-cogn.service
  - systemctl enable agentrt-dev.service
  - |
    echo "y" > /sys/kernel/mm/lru_gen/enabled
    echo "1000" > /sys/kernel/mm/lru_gen/min_ttl_ms
```

### 8.2 用户自定义 cloud-init 配置

```yaml
# user-data 示例（用户上传到云平台）
#cloud-config
hostname: agentrt-node-01
fqdn: agentrt-node-01.example.com

users:
  - name: airymax
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel,airymaxos
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAA... user@example.com

write_files:
  - path: /etc/agentrt/services/cogn_d.conf
    content: |
      [llm]
      provider = openai
      api_key = sk-xxxxxxxxxxxx
      locale = zh_CN.UTF-8

runcmd:
  - echo "agent-linux (AirymaxOS) cloud-init 完成"
  - systemctl restart cogn_d.service
```

---

## 9. 安装测试

### 9.1 测试矩阵

| 测试场景 | 安装模式 | 架构 | 文件系统 | Secure Boot | TPM | 通过标准 |
|---------|----------|------|----------|-------------|-----|----------|
| 最小安装 | Minimal | x86_64 | Btrfs | 关 | 无 | 引导成功 |
| 最小安装 | Minimal | aarch64 | Btrfs | 关 | 无 | 引导成功 |
| 完整安装 | Full | x86_64 | LVM | 开 | 2.0 | 引导 + 服务启动 |
| 完整安装 | Full | aarch64 | Btrfs | 开 | 2.0 | 引导 + 服务启动 |
| PMEM 安装 | Full | x86_64 | Btrfs + DAX | 开 | 2.0 | MemoryRovol L1 启用 |
| kickstart | Minimal | x86_64 | Btrfs | 关 | 无 | 无人值守完成 |
| QCOW2 部署 | Minimal | x86_64 | Btrfs | 关 | 无 | cloud-init 完成 |

### 9.2 自动化安装测试

```bash
#!/bin/bash
# tests-linux/install/test_install.sh [IND]
# 自动化安装测试

set -euo pipefail

ISO_PATH="$1"
TEST_VM_NAME="airymaxos-test-$$"

if [ -z "$ISO_PATH" ]; then
    echo "Usage: test_install.sh <iso_path>"
    exit 1
fi

# 1. 创建测试 VM
virt-install \
    --name "$TEST_VM_NAME" \
    --memory 4096 \
    --vcpus 2 \
    --disk size=40 \
    --cdrom "$ISO_PATH" \
    --os-variant fedora38 \
    --noautoconsole \
    --wait 30

# 2. 等待安装完成（最长 30 分钟）
TIMEOUT=1800
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
    STATE=$(virsh domstate "$TEST_VM_NAME" 2>/dev/null || echo "unknown")
    if [ "$STATE" = "shut off" ]; then
        echo "安装完成，VM 已关机"
        break
    fi
    sleep 30
    ELAPSED=$((ELAPSED + 30))
    echo "等待安装完成... ${ELAPSED}s"
done

# 3. 启动 VM 验证
virsh start "$TEST_VM_NAME"
sleep 30

# 4. SSH 验证
ssh -o StrictHostKeyChecking=no agentrt@localhost \
    "uname -r && systemctl is-active gateway_d" || exit 1

# 5. 清理
virsh destroy "$TEST_VM_NAME"
virsh undefine "$TEST_VM_NAME"

echo "OK: 安装测试通过"
```

### 9.3 无人值守安装的串口判定契约

无头验收不依赖 virsh 状态轮询，也不依赖 SSH 回连，只读 QEMU 串口日志。这要求 kickstart 与判定脚本之间存在一份稳定契约，否则「安装成功」与「安装失败」都会退化为超时。

#### 9.3.1 `%post --log` 的输出不经过串口

anaconda 把 `%post --log=FILE` 指定的文件作为该节区的标准输出，脚本的 stdout 与 stderr 全部写入目标系统的 `FILE`（例如 `/root/airymax-post.log`）。该重定向发生在脚本执行之前，串口设备不在这条路径上。

因此下面这种写法在串口日志里看不到任何痕迹：

```bash
%post --log=/root/airymax-post.log
set -eu
echo "AirymaxOS install complete"
```

判定脚本按串口内容做 `grep` 时永远匹配不到，即使安装已经成功；唯一的退出方式是等待超时，代价是数十分钟的构建时间与无法区分「慢」和「坏」。

#### 9.3.2 完成标记必须显式写 `/dev/console`

`mark()` 同时写 stdout（保留在 `--log` 文件里供事后取证）与 `/dev/console`（进入串口）。内核按 `console=` 的注册顺序把 `/dev/console` 路由到最后一个可用的控制台：本项目的启动参数为 `console=ttyS0,115200 console=ttyAMA0,115200`，故 x86_64 落到 `ttyS0`，aarch64 落到 `ttyAMA0`。为覆盖重定向异常的场景，再以 `ttyS0`、`ttyAMA0` 作为显式兜底，任一设备可写即停止：

```bash
mark()
{
    local dev
    printf '%s\n' "$*"
    for dev in /dev/console /dev/ttyS0 /dev/ttyAMA0; do
        [ -w "${dev}" ] || continue
        if printf '%s\n' "$*" >"${dev}" 2>/dev/null; then
            break
        fi
    done
    return 0
}
```

#### 9.3.3 失败必须主动上报，而不是等超时

`set -eu` 让首条失败的断言立即终止节区，但 anaconda 仍会把安装判为完成。用 `trap` 在退出时补发失败标记，把「节区失败」转成「串口可见的失败」：

```bash
trap 'rc=$?; [ "${rc}" -eq 0 ] || mark "AIRYMAX-INSTALL-FAIL rc=${rc}"' EXIT
```

`trap` 只给出退出码，`rc=1` 无法区分是哪条断言失败。除兜底 `trap` 之外，所有
显式断言统一经 `fail()` 上报，标记携带 `reason=<断言名>`：

```bash
fail()
{
    trap - EXIT
    mark "AIRYMAX-INSTALL-FAIL reason=$1"
    exit 1
}
```

`reason` 取稳定的断言名（`release-pkg`、`system-release`、`system-pkg`、
`root-account`、`airymax-account`、`root-expire`、`default-target`），与配方里的
检查项一一对应；读串口即可定位失败点，不必下载 `/root/airymax-post.log` 复现。
未预期的错误仍由 `trap` 以 `rc=N` 上报，两条路径互不影响。

判定侧把该标记纳入失败模式（与 `Kernel panic`、`Fatal glibc error`、`dracut-initqueue.*timeout` 同级），命中即判负并立即结束，不必等 `--install-timeout`：

```bash
readonly INST_FAIL='Kernel panic|Fatal glibc error|dracut-initqueue.*timeout|AIRYMAX-INSTALL-FAIL'
```

成功路径最后一步调用 `mark` 输出 `AirymaxOS <profile> install complete (baseline <date>)`，该字符串是判定侧唯一的成功依据。

#### 9.3.4 rpm 事务错误同样属于安装失败

`trap` 只能覆盖 kickstart 自身脚本的退出；rpm 在 `%pre`/`%post`/`%posttrans` 等
scriptlet 中失败时，anaconda 只打印错误并停在 `按 ENTER 退出:` 等待输入，进程
既不退出也不重启。此时串口上既无完成标记也无失败标记，判定侧只能等到
`--install-timeout` 才判负，失败原因被埋在数十分钟的超时里。

因此判定侧把 rpm 事务级错误与 `INST_FAIL` 合并为安装失败模式：

```bash
RPM_FAIL='Error in (PREIN|POSTIN|PREUN|POSTUN|PRETRANS|POSTTRANS|'
RPM_FAIL="${RPM_FAIL}TRIGGERIN|TRIGGERUN|TRIGGERPOSTUN) scriptlet|"
RPM_FAIL="${RPM_FAIL}An error occurred during the transaction"
readonly INST_ERR="${INST_FAIL}|${RPM_FAIL}"
```

`scriptlet` 前缀覆盖全部八个 scriptlet 类型，避免遗漏 `%trigger*`；事务级
错误覆盖依赖解析失败等不产生 scriptlet 报错的路径。命中任一即判负，脚本
同时回显串口尾行，使失败现场在 CI 日志中直接可读，不必下载 artifact 复现。

#### 9.3.5 交互问句同样是安装失败

anaconda 在配置阶段遇到无法自处理的错误时会转为交互问句，而不是退出进程：

```
安装引导装载程序时发生以下错误。系统将无法引导。要忽略该错误并继续安装吗？
failed to write boot loader configuration
请回答 'yes' 或 'no':
```

无头安装没有 stdin，问句永远等不到回答，串口上既无完成标记也无失败标记。这类问句本身即失败证据，判定侧与 `INST_ERR` 合并：

```bash
ASK_FAIL='failed to write boot loader configuration|'
ASK_FAIL="${ASK_FAIL}Please answer .yes. or .no.|"
ASK_FAIL="${ASK_FAIL}请回答 .yes. 或 .no."
readonly ASK_FAIL
readonly INST_ERR="${INST_FAIL}|${RPM_FAIL}|${ASK_FAIL}"
```

`trap` 与 rpm 事务规则都覆盖不到这条路径：前者只捕获 kickstart 自身脚本的退出，后者只匹配 scriptlet 报错，而 anaconda 自己抛出的异常两者都不产生。问句按中英文各留一条模式，避免安装界面语言或 TUI/文本模式差异导致漏判。

#### 9.3.6 安装腿失败时跳过启动腿

安装失败后磁盘不可引导，继续执行 `boot_run()` 只会在空盘上耗尽
`--boot-timeout`，并把一次失败拖成两次等待。判定侧在安装腿失败后为该轮
置 `skip` 标记：启动腿输出 `not ok`（保留 TAP 编号连续性）与
`skipped: install failed`，manifest 记 `boot=skip`，不再启动 QEMU。

#### 9.3.7 长耗时阶段必须回显进度

无头验收只读串口日志，安装与启动两条腿在失败前可能静默数十分钟，CI 日志
里看不到任何推进迹象。判定侧在等待循环内每 30 秒打印一次当前轮次、已耗时
与该日志最后一条非空行（剥除 ANSI 序列与回车覆盖），使「慢」与「卡死」在
日志中可区分，也便于在超时前终止明显走偏的运行。进度回显只读日志，不参与
判定。

#### 9.3.8 图形会话需要 DRM/KMS 设备

安装成功只证明用户态可引导；UKUI 会话还要求 lightdm 能拿到一个 DRM/KMS 节点。本项目内核基线 `arch/x86/configs/x86_64_defconfig` 只启用 `CONFIG_DRM_VIRTIO_GPU=y`，未启用 `DRM_BOCHS`，也没有 `FB_VESA`；`openeuler_defconfig` 中的 `DRM_BOCHS=m` 不参与本项目的 x86_64 构建（spec 以 `make defconfig` 加 `airy_defconfig` 片段配置）。

缺省 std VGA 因此没有驱动，`/dev/dri/card0` 不会出现，T1 图形会话必败。两种架构都接入 virtio-gpu：

| 架构 | QEMU 参数 | 说明 |
|------|-----------|------|
| x86_64 | `-vga virtio` | 替换缺省的 std VGA，而不是并列追加第二个显示设备 |
| aarch64 | `-machine virt -device virtio-gpu-pci` | `virt` 机型没有缺省 VGA，直接追加 PCI 显示设备 |

`DRM_VIRTIO_GPU` 的 Kconfig 依赖为 `DRM && VIRTIO_MENU && MMU`，并 `select VIRTIO`、`DRM_KMS_HELPER`、`DRM_GEM_SHMEM_HELPER`、`VIRTIO_DMA_SHARED_BUFFER`；这些在基线配置中均已满足，无需追加片段。

#### 9.3.9 安装器 chroot 内没有 systemd 总线

`%post` 运行在目标系统的 chroot 里，此时 systemd 尚未启动、`/run/systemd/system`
不存在。`systemctl` 的单元文件类动词（`set-default`、`enable`、`disable`、`mask`）
会尝试连接总线并以非零码失败；不同 systemd 版本对此的退出码与报错并不一致，
既可能让 `set -e` 中断安装，也可能静默留下未生效的配置。规则：

| 约束 | 做法 |
|------|------|
| 单元文件动词 | 一律显式 `systemctl --root=/ <verb> ...`，走离线（文件）语义，不依赖总线 |
| 默认 target | 由各剖面 kickstart 唯一设定（工作站 `graphical.target`、最小化 `multi-user.target`），`airymaxos-release` 的策略脚本不重复设定（SSoT） |
| 成功判据 | 不看 `systemctl` 退出码，看落盘事实：`readlink -f /etc/systemd/system/default.target` 必须指向 `/usr/lib/systemd/system/<target>`，否则 `fail default-target` |
| `daemon-reload` | 总线动词，对尚未启动的已安装系统没有意义，`%post` 不调用 |

`set-default` 工具退出码不可信时先回显 `AIRYMAX-INSTALL-NOTE set-default rc=N`，
再以符号链接判定，使「工具语义差异」与「真实失败」在串口上可区分。

---

## 10. 错误处理

### 10.1 安装失败处理

| 错误场景 | 处理方式 |
|---------|---------|
| 磁盘空间不足 | 提示用户调整分区 |
| 网络不可用 | 安装中止并提示检查网络；ISO 内离线仓库（`/airymax/Packages`，cost=1）只承载 `airymaxos-*` 自有包，基础用户空间与 UKUI 闭包来自 kickstart 声明的 openEuler 网络仓库，完整离线闭包随 M-12 落地 |
| Secure Boot 校验失败 | 提示用户禁用 Secure Boot 或注册密钥 |
| 包签名验证失败 | 中止安装，提示仓库损坏 |
| GRUB 安装失败 | 回滚分区变更，提示用户手动安装 |
| PMEM 配置失败 | 跳过 PMEM 挂载，继续安装（降级模式） |

### 10.2 错误码定义

安装器子系统错误码遵循 ErrorCodeSystem，定义于 [SC] 共享契约层：

```c
#define AIRY_INST_EDISK      (-940)  /* 磁盘错误 */
#define AIRY_INST_EPART      (-941)  /* 分区错误 */
#define AIRY_INST_EBOOT      (-942)  /* 引导加载器错误 */
#define AIRY_INST_ENET       (-943)  /* 网络错误 */
#define AIRY_INST_EPKG      (-944)  /* 包安装错误 */
#define AIRY_INST_ESIGN     (-945)  /* 签名校验失败 */
#define AIRY_INST_ESECURE   (-946)  /* Secure Boot 错误 */
#define AIRY_INST_ETPM      (-947)  /* TPM 错误 */
#define AIRY_INST_EPMEM     (-948)  /* PMEM 配置错误 */
#define AIRY_INST_EKICKSTART (-949) /* kickstart 解析错误 */
```

### 10.3 集中错误处理

安装器代码采用 `goto out_free_xxx` 集中错误处理（OS-KER-003/004）：

```c
int airy_installer_run(struct install_config *cfg)
{
    struct install_ctx *ctx;
    int err;

    ctx = installer_alloc_ctx();
    if (!ctx)
        return -ENOMEM;

    err = installer_init_network(ctx, cfg);
    if (err)
        goto out_free_ctx;

    err = installer_partition_disk(ctx, cfg);
    if (err)
        goto out_free_network;

    err = installer_install_packages(ctx, cfg);
    if (err)
        goto out_free_partition;

    err = installer_configure_grub(ctx, cfg);
    if (err)
        goto out_free_packages;

    err = installer_enable_services(ctx, cfg);
    if (err)
        goto out_free_grub;

    err = installer_create_snapshot(ctx);
    if (err)
        goto out_free_services;

    pr_info("agent-linux: 安装完成\n");
    return 0;

out_free_services:
    installer_release_services(ctx);
out_free_grub:
    installer_release_grub(ctx);
out_free_packages:
    installer_release_packages(ctx);
out_free_partition:
    installer_release_partition(ctx);
out_free_network:
    installer_release_network(ctx);
out_free_ctx:
    kfree(ctx);
    return err;
}
```

---

## 11. 五维原则映射

| 原则 | 在安装器设计中的体现 |
|------|---------------------|
| **S-4 涌现性管理** | 三种安装模式 + 五种部署场景管理用户场景涌现性 |
| **C-2 增量演化** | kickstart 模板按版本演进 |
| **E-1 安全内生** | Secure Boot + GPG 签名 + TPM 度量默认启用 |
| **E-3 资源确定性** | PMEM/CXL/大页在安装期即配置完成 |
| **E-6 错误可追溯** | 安装日志全程记录 + 分级错误码 |
| **E-7 文档即代码** | kickstart 配置即安装契约文档 |
| **E-8 可测试性** | 自动化无人值守安装 + 测试矩阵 |
| **K-2 接口契约化** | kickstart 是安装接口契约 |
| **IRON-9 v3 [IND]** | 安装器独立于 agentrt |

---

## 12. IRON-9 v3 同源映射

| 组件 | agent-linux（[IND]） | agentrt（[IND]） |
|------|--------------------------|------------------|
| 安装器 | Anaconda + kickstart | 无（用户态运行时） |
| 引导加载器 | GRUB2 / systemd-boot | 无 |
| cloud-init | 完整支持 | 无 |
| ISO/QCOW2 | 完整支持 | 无 |
| Secure Boot | 完整支持 | 无 |
| TPM 度量 | 完整支持 | 无 |

agent-linux 的安装器与 agentrt 完全独立（[IND] 完全独立层），各自演进。

---

## 13. 相关文档

- `190-distribution/README.md`（发行版管理主索引）
- `190-distribution/01-rpm-packaging.md`（RPM 打包设计）
- `190-distribution/02-dnf-repo-design.md`（dnf 仓库设计）
- `190-distribution/04-update-mechanism.md`（更新机制设计）
- `100-operations/README.md`（部署运维）
- `110-security/README.md`（Secure Boot 与 TPM）
- `170-performance/02-memory-performance.md`（PMEM/CXL 分区）
- `agent-linux` 仓 `tools/airy-flash/README.md`（介质写入的操作权威源，§7.3 只定设计）

---

## 14. 参考材料

- Anaconda 安装器文档
- pykickstart 项目
- GRUB 2 手册
- UEFI Secure Boot 规范
- TPM 2.0 规范
- Btrfs 文件系统手册
- cloud-init 项目
- mkisofs / xorriso 文档
- qemu-img 工具文档
- 主流 Linux 发行版安装器实践

---

## 15. 版本演进

| 版本 | 安装模式 | 分区 | 引导 | kickstart | 备注 |
|------|---------|------|------|-----------|------|
| 0.1.1 | 设计 | — | — | — | 仅文档 |
| 1.0.1 | 三模式 | Btrfs/LVM | GRUB + SB + TPM | 完整 | 开发 |
| 1.1.0 | + Cloud | + RAID | + Measured Boot | + 模板 | 云环境 |
| 2.0.0 | + Container | + OverlayFS | + AI 辅助 | + 智能配置 | 完整 |

---

> **文档结束** | agent-linux（AirymaxOS）安装器设计 v1.0.3
