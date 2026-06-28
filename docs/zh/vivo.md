# vivo/iQOO 适配教程

[English](../vivo.md) | [返回中文首页](./README.md)

这篇文档整理 vivo/iQOO 在 GKI 时代的 root 限制、SakiSU 当前实现，以及按现有代码逻辑使用 Manager 修补镜像的方法。

## 先读风险

- 解锁 bootloader、刷写 boot/init_boot/vendor_boot 都有变砖或丢数据风险。动手前先备份原始分区。
- 不要混用不同系统版本的 `vendor_boot.img`、`init_boot.img` 或 `boot.img`。
- `vendor_boot` 修错可能影响正常启动、recovery 和 fastbootd。能进 fastboot 时优先刷回原始镜像恢复。
- 3.x/4.x 时代 vivo 将反 root 逻辑嵌入内核，需要逆向内核处理，不属于 SakiSU 当前处理范围。有早期内核或 APatch 方案需求，可参考酷安 @酷狗贼 的相关研究。

## vivo root 限制背景

早期 vivo/iQOO 的反 root 逻辑嵌入在厂商内核中，处理难度高，往往需要逆向内核。GKI 标准引入后，厂商私有驱动和功能不应继续塞进通用内核，vivo 将这部分能力迁移到 `vendor_boot` 里的厂商模块中，其中最关键的是 `vr.ko`。

`vendor_boot` 以 ramdisk 的形式向通用内核提供厂商驱动、挂载配置和初始化脚本。设备启动时，内核会从 `vendor_boot` 中按清单加载模块。`vr.ko` 被加载后会隐藏自身，因此开机后通常无法直接从已加载模块列表中看到它。

SakiSU 的目标不是修改 `vr.ko`，而是让它不再被加载：清理 `modules.load`、`modules.dep`、`modules.softdep`、`modules.load.recovery` 等清单中对 `vr.ko` 的引用，并移除 cpio 中存在的 `vr.ko` 文件。

## 官方内核与 `_vivo` KMI

在 vivo 5.x 到 6.1 的 GKI 设备上，`vendor_boot` 常见两套模块：

- `lib/modules/` 下的官方内核模块。
- `lib/modules/<version>-gki/` 下的 GKI 通用内核模块。

为了区分官方内核和 GKI 通用内核，vivo 官方内核会校验模块的 `vermagic`。标准 Linux 模块加载路径会从 ELF `.modinfo` 中读取 `vermagic=`，再与内核内嵌的期望字符串比较；不匹配则加载失败。vivo 样本中的期望串可能类似：

```text
6.1.145-android14-11-maybe-dirty SMP preempt mod_unload modversions vivo aarch64
```

这里的 `vivo` 字段会让普通 GKI LKM 无法被 vivo 官方内核接受。因此，在官方内核上使用 KernelSU LKM 时，需要带 `_vivo` 后缀的 KMI/LKM 变体。SakiSU Manager 在 vivo 修补开启时会优先展示 `_vivo` KMI，并在执行命令时自动补上 `_vivo` 后缀。

6.6 以后，部分新机型可能不再使用“两套 ko”机制，官方内核加载第三方 LKM 的行为也可能变化。该阶段仍需要更多设备验证。

## SakiSU 实现了什么

Manager 中的 vivo 开关文案是 **“去除vr或适配vivo特性”**。开启后，Manager 统一调用：

```text
ksud boot-patch-vivo
```

后端会自动判断镜像类型：

1. `patch_vivo()` 会把 `vr.ko` 加入待移除模块列表。
2. 后续仍走正常 `patch()` 路径。
3. `patch()` 先解析 ramdisk cpio，再决定是否加载 KernelSU LKM 资源。
4. 如果 cpio 中存在 `lib/modules/*.ko`，就按 `vendor_boot` 处理：跳过 LKM 注入，只执行 rmvr。
5. `remove_vendor_modules()` 会动态发现 `lib/modules` 和 `lib/modules/<version>-gki`，清理 `modules.load`、`modules.dep`、`modules.softdep`、`modules.load.recovery`。
6. 如果不是 vendor ramdisk，就按 `init_boot`/boot 路径继续注入 KernelSU LKM。

Manager 还会在需要时调用：

```text
ksud boot-info classify-image <image>
```

它会输出 `vendor_boot`、`init_boot` 或 `unknown`，用于决定是否需要弹出 KMI 选择框。正常情况下，`vendor_boot.img` 不会再强制弹 KMI；`init_boot.img` 或无法确认的镜像仍会保留 KMI 选择，以避免错误注入。

## Manager 使用教程

### 准备

1. 确认设备已解锁 bootloader。
2. 准备和当前系统版本一致的 `init_boot.img`、`vendor_boot.img` 或 `boot.img`。
3. 备份原始分区，例如 `boot`、`init_boot`、`vendor_boot`、`vbmeta`。
4. 安装 SakiSU Manager，并确认使用的是包含 vivo 修补功能的构建。

### 移除 `vendor_boot` 中的 `vr.ko`

1. 打开 SakiSU Manager，进入安装页。
2. 开启 **vivo修补**。
3. 选择文件修补，选择当前系统对应的 `vendor_boot.img`。
4. 正常情况下不会弹出 KMI 选择框。
5. 等待输出 `kernelsu_patched_vivo_*.img`。
6. 重启到 bootloader，将输出镜像刷回 `vendor_boot`：

```text
fastboot flash vendor_boot kernelsu_patched_vivo_xxx.img
fastboot reboot
```

如果你的设备要求指定槽位，请按实际分区和槽位刷入。刷错系统版本的 `vendor_boot` 可能无法启动。

### 修补 `init_boot` 注入 KernelSU LKM

1. 打开 SakiSU Manager，进入安装页。
2. 开启 **vivo修补**。
3. 选择文件修补，选择当前系统对应的 `init_boot.img`。少数旧设备可能使用 `boot.img`。
4. 弹出 KMI 选择时，优先选择带 `_vivo` 后缀、且与本机内核版本匹配的项。
5. 等待输出 `kernelsu_patched_vivo_*.img`。
6. 刷回对应分区：

```text
fastboot flash init_boot kernelsu_patched_vivo_xxx.img
fastboot reboot
```

如果设备没有 `init_boot` 分区而使用 `boot` 分区，请刷入 `boot`，不要把 `init_boot` 镜像刷到 `vendor_boot`。

### 两步都需要吗

这取决于你的目标：

- 只想移除 vivo 反 root 限制：修补并刷入 `vendor_boot`。
- 想在 vivo 官方内核上使用 KernelSU LKM：修补并刷入 `init_boot`，选择 `_vivo` KMI。
- 想同时使用 KernelSU 并去除 `vr.ko`：分别修补 `init_boot` 和 `vendor_boot`，并分别刷回对应分区。

SakiSU 的 vivo 开关可以一直打开。后端会根据镜像内容自动选择 rmvr 或 LKM 注入路径。

## 手动处理 `vendor_boot`

如果你想手动理解这套流程，可以使用 Magisk 官方 Linux 版 `magiskboot`。Android 构建的 Magisk App 内含 `libmagiskboot.so`，它就是可用于 Linux/Android 环境的 magiskboot 实现。

基本流程：

```text
./magiskboot unpack vendor_boot.img
```

解包后会得到 `ramdisk.cpio`。在 cpio 中检查：

```text
lib/modules/
lib/modules/<version>-gki/
```

这些目录下常见清单包括：

```text
modules.load
modules.dep
modules.softdep
modules.load.recovery
```

需要删除所有直接引用 `vr.ko` 或间接引用 `vr.ko` 的行。`vr.ko` 文件本身是否保留不是重点，只要它不再被清单加载即可；SakiSU 会在自动流程中顺手移除文件。清单一旦漏删，可能导致启动、recovery 或 fastbootd 异常。

重新打包时使用：

```text
./magiskboot repack vendor_boot.img
```

手动流程只建议用于验证和研究。普通用户优先使用 SakiSU Manager。

## 常见问题

### 选择 `vendor_boot.img` 时仍弹出 KMI

这通常表示 Manager 的预分类没有拿到明确结果。为了安全，UI 会保留 KMI 选择。即使选择了 KMI，后端在识别到 `lib/modules/*.ko` 后也会跳过 LKM 注入，只执行 `vr.ko` 清理。

### 官方内核无法加载 KernelSU LKM

优先检查是否选择了 `_vivo` 后缀 KMI。vivo 官方内核会通过 vermagic 区分官方内核模块和 GKI 通用模块，普通 GKI LKM 可能被拒绝加载。

### 刷入后不开机

先刷回原始镜像恢复。常见原因包括镜像版本不匹配、刷错分区、`vendor_boot` 清单残留、或设备本身的启动链路还有额外校验。

### APatch、SKRoot 等方案能否配合

SakiSU 的 rmvr 只处理 `vendor_boot` 中的 `vr.ko` 加载问题。去除 `vr.ko` 后，理论上不会阻止你继续研究 APatch、SKRoot 等内核级方案，但不同设备和内核版本仍需要单独验证。

