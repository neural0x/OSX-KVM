# 虚拟机黑苹果项目 OSX-KVM 简单解析：如何在 KVM 上运行 macOS

`OSX-KVM` 是一个利用 Linux KVM (Kernel-based Virtual Machine) 和 QEMU 在标准 x86 硬件上运行 macOS 的开源项目。该项目的核心在于通过 QEMU 模拟出一套 macOS 能识别并接受的硬件环境，并通过 OpenCore (OC) 引导加载程序来修补内核和注入驱动，从而实现“黑苹果”虚拟化。

本文将简单介绍其实现原理。至于安装过程，可以参考[之前的这篇文章](https://mp.weixin.qq.com/s/-3gKfcGq1JcO02lBWO6TzQ?token=1624196960&lang=zh_CN)。[咱们之前的fork提供了针对AMD处理器的脚本调整，不过官方版本也已经有了类似的调整，于是咱们的分支就与官方同步了，现在加上了个中文文档](https://github.com/cycleuser/OSX-KVM/blob/master/README_CN.md)。


## 1. 项目核心架构

整个项目由三个主要部分组成：

1.  **硬件模拟层 (QEMU/KVM)**：
    *   提供虚拟 CPU、内存、芯片组 (Q35)、USB 控制器等硬件。
    *   通过 KVM 加速，使虚拟机性能接近原生。
2.  **引导加载层 (OpenCore)**：
    *   这是最关键的一层。macOS 只能在 Apple 硬件上启动，OpenCore 负责欺骗 macOS，让它以为自己运行在真正的 Mac 上（例如 MacPro7,1 或 iMac19,1）。
    *   它负责加载内核扩展 (Kexts)，修补 ACPI 表，模拟 NVRAM 等。
3.  **资源获取层 (`fetch-macOS-v2.py`)**：
    *   直接从 Apple 服务器下载 macOS 恢复镜像 (`BaseSystem.dmg`)，无需 Apple ID 或 Mac 电脑。

## 2. 关键文件解析

### 2.1 镜像获取：`fetch-macOS-v2.py`
这是一个 Python 脚本（基于 `macrecovery`），它通过解析 Apple 的软件更新目录 (catalog)，下载并解密 macOS 的恢复镜像 (`BaseSystem.dmg` 和 `BaseSystem.chunklist`)。

*   **作用**：获取 macOS 安装程序的“种子”。
*   **流程**：`Makefile` 调用此脚本 -> 下载 `.dmg` -> `qemu-img` 将其转换为 QEMU 原生支持的 `.img` (raw) 格式。

### 2.2 引导介质：`OpenCore/OpenCore.qcow2`
这是一个预配置好的虚拟磁盘，包含 EFI 分区。
*   **内容**：`EFI/BOOT/BOOTx64.efi` (OpenCore 核心) 和 `EFI/OC/config.plist` (OpenCore 配置文件)。
*   **作用**：QEMU 启动时首先加载这个磁盘，进入 OpenCore 菜单，然后由 OpenCore 引导 macOS 安装盘或系统盘。

### 2.3 启动脚本：`OpenCore-Boot.sh`
这是项目的核心控制脚本。它将成百上千个 QEMU 参数组合起来，构建出一个 macOS 兼容的虚拟机。

## 3. `OpenCore-Boot.sh` 剖析

让我们逐段分析这个脚本是如何构建虚拟机的：

### 3.1 基础配置与 CPU 模拟
```bash
MY_OPTIONS="+ssse3,+sse4.2,+popcnt,+avx,+aes,+xsave,+xsaveopt,check"

# ...

args=(
  -enable-kvm -m "$ALLOCATED_RAM" 
  -cpu Skylake-Client,-hle,-rtm,kvm=on,vendor=GenuineIntel,+invtsc,vmware-cpuid-freq=on,"$MY_OPTIONS"
  -machine q35
  # ...
)
```
*   **`-enable-kvm`**: 启用内核级虚拟化加速。
*   **`-cpu Skylake-Client`**: macOS 对 CPU 架构有严格要求。旧版脚本使用 `Penryn`，新版为了支持 macOS Sonoma/Sequoia，通常建议使用 `Skylake-Client` 或 `Haswell-noTSX`。
*   **`vendor=GenuineIntel`**: 强制将 CPU 厂商ID设为 Intel，因为 macOS 内核当年曾经有多年的 Intel CPU 支持。
*   **`+invtsc`**: 保证时间戳计数器同步，防止系统时钟漂移。
*   **`-machine q35`**: 模拟 Intel Q35 芯片组，这是 macOS 支持较好的较新芯片组架构（相比 i440fx）。

### 3.2 模拟 Apple SMC (System Management Controller)
```bash
-device isa-applesmc,osk="ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc"
```
*   **原理**：这是在非 Apple 硬件上运行 macOS 的法律和技术核心。真实 Mac 有一个 SMC 芯片，用于电源管理等。macOS 启动时会查询 SMC 并验证一段特定的密钥。
*   **实现**：QEMU 的 `isa-applesmc` 设备模拟了这个芯片，并直接传入了这段被硬编码的 OSK 密钥，从而通过 macOS 的正版验证。

### 3.3 固件 (UEFI)
```bash
-drive if=pflash,format=raw,readonly=on,file="$REPO_PATH/$OVMF_DIR/OVMF_CODE_4M.fd"
-drive if=pflash,format=raw,file="$REPO_PATH/$OVMF_DIR/OVMF_VARS-1920x1080.fd"
```
*   macOS 需要 UEFI 环境启动，不支持传统的 BIOS。
*   项目使用 **OVMF (Open Virtual Machine Firmware)**，这是基于 Tianocore EDK II 的开源 UEFI 实现。
*   `OVMF_CODE.fd` 是固件代码，`OVMF_VARS.fd` 存储 UEFI 变量（如启动顺序、屏幕分辨率）。

### 3.4 存储与引导顺序
```bash
-device ich9-ahci,id=sata
-drive id=OpenCoreBoot,if=none,snapshot=on,format=qcow2,file="$REPO_PATH/OpenCore/OpenCore.qcow2"
-device ide-hd,bus=sata.2,drive=OpenCoreBoot
-device ide-hd,bus=sata.3,drive=InstallMedia
-drive id=InstallMedia,if=none,file="$REPO_PATH/BaseSystem.img",format=raw
-drive id=MacHDD,if=none,file="$REPO_PATH/mac_hdd_ng.img",format=qcow2
-device ide-hd,bus=sata.4,drive=MacHDD
```
这里挂载了三个磁盘，顺序至关重要：
1.  **OpenCoreBoot (`OpenCore.qcow2`)**：作为第一个 SATA 设备 (`sata.2`)，BIOS 会优先引导它。
2.  **InstallMedia (`BaseSystem.img`)**：这是下载的 macOS 恢复镜像。
3.  **MacHDD (`mac_hdd_ng.img`)**：这是你创建的空磁盘，用于安装 macOS。

**启动流程**：
QEMU -> 加载 OVMF -> 启动 OpenCore (SATA 2) -> OpenCore 扫描其他磁盘 -> 发现 macOS BaseSystem (SATA 3) -> 用户选择安装 -> 安装到 MacHDD (SATA 4)。

### 3.5 网络与外设
```bash
-netdev user,id=net0,hostfwd=tcp::2222-:22 -device virtio-net-pci,netdev=net0,id=net0,mac=52:54:00:c9:18:27
```
*   **网络**：使用 `virtio-net-pci` 半虚拟化网卡，性能最好。macOS 原生好像是不支持 VirtIO，这里估计是 OpenCore 会注入相关驱动。
*   **端口转发**：`hostfwd=tcp::2222-:22` 允许你通过 `ssh -p 2222 root@localhost` 从宿主机连接到虚拟机。

## 4. 总结：从零到启动

整个 `OSX-KVM` 的实现流程可以概括为：

1.  **伪装**：通过 QEMU 参数 (`-cpu`, `-device isa-applesmc`) 将 Linux KVM 伪装成一台 Intel Mac。
2.  **引导**：利用 `OpenCore.qcow2` 接管启动过程，进一步修补硬件信息，使其符合 macOS 的严格要求。
3.  **安装**：加载 `BaseSystem.img` 恢复环境，将 macOS 安装到虚拟硬盘 `mac_hdd_ng.img` 中。
4.  **运行**：系统安装完成后，每次启动依然通过 OpenCore 引导进入已安装的 macOS 系统。
