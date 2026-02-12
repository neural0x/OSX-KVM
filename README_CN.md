### 注意

本 `README.md` 记录了创建 `Virtual Hackintosh`（虚拟黑苹果）系统的过程。

注意：本仓库包含的所有 blob 和资源都是可重新推导的（包含所有指令！）。

💚 想要寻求关于此内容的 **商业** 支持？如果只是关于 **商业支持选项**，可以通过 [邮件联系本项目的原作者 Kholia](mailto:dhiru.kholia@gmail.com?subject=[GitHub]%20OSX-KVM%20Commercial%20Support%20Request&body=Hi%20-%20We%20are%20interested%20in%20purchasing%20commercial%20support%20options%20for%20your%20project.) 进行交流。注意：项目赞助者可以访问 `Private OSX-KVM` 仓库并获得直接支持。

在 `内容缓存` 方面遇到困难？我们可以提供帮助。

使用 `Proxmox` 和 macOS：
- 务必查看 [Nick 的博客](https://www.nicksherlock.com/)
- 这里有一个更适合 `Proxmox` 的最新版本 [OpenCore-ISO](https://github.com/LongQT-sea/OpenCore-ISO)

是的，我们现在支持离线 macOS 安装 - 参见 [此文档](./run_offline.md) 🎉


### 回馈

本项目永远需要你的帮助、时间和关注。我正在寻找以下工作项的帮助（pull-requests！）：

* 关于在流行的云提供商（Hetzner, GCP, AWS）上运行 macOS 的文档。参见 `这合法吗？` 章节及相关参考资料。

* 记录（分享）你如何使用本项目来构建 + 测试开源项目 / 完成你的工作。

* 记录如何使用本项目进行 XNU 内核调试和开发。

* 记录启动一堆无头（headless）macOS 虚拟机（构建农场）的过程。

* 记录使用 [munki](https://github.com/munki/munki) 向此类 `构建农场` 部署软件的用法。

* 开箱即用或更轻松地启用 VNC + SSH 支持。

* 永远欢迎稳健性改进！

* （不那么）疯狂的想法 - 通过 OpenCV 自动化 macOS 安装。


### 要求

* 一个现代的 Linux 发行版。例如 Ubuntu 24.04 LTS 64位或更高版本。

* QEMU >= 8.2.2

* 需要支持 Intel VT-x / AMD SVM 的 CPU (`grep -e vmx -e svm /proc/cpuinfo`)

* macOS Sierra 及以上版本需要支持 SSE4.1 的 CPU

* macOS Ventura 及以上版本需要支持 AVX2 的 CPU

注意：已知较旧的 AMD CPU 存在问题，但现代 AMD Ryzen 处理器工作正常（即使是 macOS Tahoe）。


### 安装准备

* 安装 QEMU 和其他软件包。

  ```
  sudo apt-get install qemu-system uml-utilities virt-manager git \
      wget libguestfs-tools p7zip-full make dmg2img tesseract-ocr \
      tesseract-ocr-eng genisoimage vim net-tools screen -y
  ```

  此步骤可能需要根据你的 Linux 发行版进行调整。

* 在你的 QEMU 系统上克隆此仓库。接下来的步骤将使用此仓库中的文件。

  ```
  cd ~

  git clone --depth 1 --recursive https://github.com/cycleuser/OSX-KVM.git

  cd OSX-KVM
  ```

  可以通过以下命令拉取仓库更新：

  ```
  git pull --rebase
  ```

  本仓库大量使用基于 rebase 的工作流。

* KVM 可能需要在宿主机上进行以下调整才能工作。

  ```
  sudo modprobe kvm; echo 1 | sudo tee /sys/module/kvm/parameters/ignore_msrs
  ```

  要使此更改永久生效，可以使用以下命令。
  如果不确定，请使用 `lscpu`。

  ```
  sudo cp kvm.conf /etc/modprobe.d/kvm.conf  # 仅限 intel 机器

  sudo cp kvm_amd.conf /etc/modprobe.d/kvm.conf  # 仅限 amd 机器
  ```

* 将用户添加到 `kvm` 和 `libvirt` 组（可能需要）。

  ```
  sudo usermod -aG kvm $(whoami)
  sudo usermod -aG libvirt $(whoami)
  sudo usermod -aG input $(whoami)
  ```

  注意：执行此命令后请重新登录。

* 获取 macOS 安装程序。

  ```
  ./fetch-macOS-v2.py
  ```

  你可以在这里选择你想要的 macOS 版本。执行此步骤后，你应该在当前文件夹中拥有 `BaseSystem.dmg` 文件。

  注意：如果速度较慢，让 `>= Big Sur` 的设置在 `国家选择` 屏幕和其他类似地方停留一段时间。初始 macOS 设置向导最终会成功。

  运行示例：

  ```
  $ ./fetch-macOS-v2.py
  1. High Sierra (10.13)
  2. Mojave (10.14)
  3. Catalina (10.15)
  4. Big Sur (11.7)
  5. Monterey (12.6)
  6. Ventura (13)
  7. Sonoma (14) - RECOMMENDED
  8. Sequoia (15)
  9. Tahoe (26)

  Choose a product to download (1-9): 9
  ```

  注意：现代 NVIDIA GPU 支持 HighSierra，但不支持更高版本的 macOS。

* 将下载的 `BaseSystem.dmg` 文件转换为 `BaseSystem.img` 文件。

  ```
  dmg2img -i BaseSystem.dmg BaseSystem.img
  ```

* 创建一个将安装 macOS 的虚拟硬盘镜像。如果你将磁盘镜像名称从 `mac_hdd_ng.img` 更改为其他名称，则需要更新启动脚本以指向新的镜像名称。

  ```
  qemu-img create -f qcow2 mac_hdd_ng.img 512G
  ```

  注意：为了获得最佳效果，请在快速 SSD/NVMe 磁盘上创建此 HDD 镜像文件。

* 现在你准备好安装 macOS 了 🚀


### 安装

- CLI 方法（主要）。只需运行 `OpenCore-Boot.sh` 脚本即可开始安装过程。

  ```
  ./OpenCore-Boot-new.sh
  ```

  注意：此脚本适用于所有最近的 macOS 版本。

- 使用 macOS 安装程序中的 `磁盘工具` 对连接到 macOS 虚拟机的虚拟磁盘进行分区和格式化。对于现代 macOS 版本，请使用 `APFS`（默认）。

-以此类推，安装 macOS 🙌

原来的 `OpenCore-Boot.sh` 脚本有点旧，已被 'OpenCore-Boot-new.sh' 取代。新脚本已在 AMD Ryzen 9 7955HX 和 Ubuntu 24.04 上测试通过，截图如下。

![](./screenshots/osx-kvm-amd.png)

- (可选) 将此 macOS 虚拟机磁盘与 libvirt (virt-manager / virsh 等) 一起使用。

  - 编辑 `macOS-libvirt-Catalina.xml` 文件并更改各种文件路径（在该文件中搜索 `CHANGEME` 字符串）。通常以下命令可以搞定。

    ```
    sed "s/CHANGEME/$USER/g" macOS-libvirt-Catalina.xml > macOS.xml

    virt-xml-validate macOS.xml
    ```

  - 通过运行以下命令创建虚拟机。

    ```bash
    virsh --connect qemu:///system define macOS.xml
    ```

  - 如果需要，授予 libvirt-qemu 用户必要的权限，

    ```
    sudo setfacl -m u:libvirt-qemu:rx /home/$USER
    sudo setfacl -R -m u:libvirt-qemu:rx /home/$USER/OSX-KVM
    ```

  - 启动 `virt-manager` 并启动 `macOS` 虚拟机。


### 无头 macOS

- 使用提供的 [boot-macOS-headless.sh](./boot-macOS-headless.sh) 脚本。

  ```
  ./boot-macOS-headless.sh
  ```


### 设定正确的期望

干得好，你建立了一个 `Virtual Hackintosh` 系统！这样的系统可以用于各种目的（例如软件构建、测试、逆向工程），如果加上本仓库中记录的一些调整，它可能就是你所需要的一切。

然而，这样的系统缺乏图形加速、可靠的声音子系统、USB 3 功能和其他类似的东西。要启用这些功能，请查看我们的 [笔记](notes.md)。我们希望恢复我们在这一领域的测试和文档工作。如果你能够资助这一领域的工作，请 [联系我们](mailto:dhiru.kholia@gmail.com?subject=[GitHub]%20OSX-KVM%20Funding%20Support)。

拥有“超越原生苹果硬件”的性能是可能的，但这确实需要工作、耐心和一点运气（也许？）。


### 安装后

* 参见 [网络笔记](networking-qemu-kvm-howto.txt) 了解如何在你的虚拟机中设置网络，包括出站和入站（通过 SSH、VNC 等远程访问你的虚拟机）。

* 要直通 GPU 和其他设备，参见 [这些笔记](notes.md#gpu-passthrough-notes)。

* 需要不同的分辨率？查看本仓库中包含的 [笔记](notes.md#change-resolution-in-opencore)。

* iMessage 有问题？查看本仓库中包含的 [笔记](notes.md#trouble-with-imessage)。

* 强烈推荐的 macOS 调整 - https://github.com/sickcodes/osx-optimizer


### 这合法吗？

“秘密”的 Apple OSK 字符串在互联网上广泛可用。它也包含在公开的法庭文件中 [可在此处获取](http://www.rcfp.org/sites/default/files/docs/20120105_202426_apple_sealing.pdf)。我不是律师，但看起来 Apple 试图将 OSK 字符串视为商业秘密的尝试并没有成功。由于这些原因，OSK 字符串免费包含在本仓库中。

请查看 [Dortania 的 OpenCore 安装指南中的“Hackintoshing 的合法性”文档片段](https://dortania.github.io/OpenCore-Install-Guide/why-oc.html#legality-of-hackintoshing)。

Gabriel Somlo 也对 [在 QEMU/KVM 下运行 macOS 涉及的法律方面有一些想法](http://www.contrib.andrew.cmu.edu/~somlo/OSXKVM/)。

你可能也会觉得 [这篇文章 '宣布推出用于 macOS 的 Amazon EC2 Mac 实例'](https://aws.amazon.com/about-aws/whats-new/2020/11/announcing-amazon-ec2-mac-instances-for-macos/)很有趣。

注意：你有责任理解并接受（或不接受）Apple EULA。

注意：这不是法律建议，所以如果你有任何疑虑，请自行进行适当的评估并与你的律师讨论（文本来源：Dortania）


### 动机

我的目标是以一种简单、可复现的方式启用基于 macOS 的教育任务、构建 + 测试、内核调试、逆向工程和 macOS 安全研究，而不需要（过于沉重地）“投资”于 Apple 的封闭生态系统。

这些 `Virtual Hackintosh` 系统并不旨在取代真正的物理 macOS 系统。

就个人而言，这个仓库是我“退出”Apple 生态系统的一种方式。它帮助我测试和比较了 `Canon CanoScan LiDE 120` 扫描仪和 `Brother HL-2250DN` 激光打印机的互操作性。这些设备现在在现代版本的 Ubuntu 上工作得相当不错（为自由软件欢呼）。此外，很久以前，我不得不彻底擦除我（当时）全新的 `MacBook Pro (Retina, 15-inch, Late 2013)` 并安装 Xubuntu——因为 `OS X` 内核一直在上面崩溃！

背景故事：前世我在加拿大是一个（穷）学生，Apple 让 [我在破解 Apple Keychains 上的工作](https://github.com/openwall/john/blob/bleeding-jumbo/src/keychain_fmt_plug.c) 比它需要的难得多。这就是我如何对 Hackintosh 系统产生兴趣的。
