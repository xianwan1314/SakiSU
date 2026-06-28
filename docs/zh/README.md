# SakiSU

<img align="right" src="SakiSU_blue.svg" width="220px" alt="SakiSU Icon">

[English](../README.md) | **简体中文** | [vivo/iQOO 适配教程](./vivo.md)

SakiSU 是基于 [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) 的下游分支，保留 KernelSU/SukiSU 的主要能力，并加入面向 vivo/iQOO 设备的兼容性改动。

[![最新发行](https://img.shields.io/github/v/release/XingChenRS/SakiSU?label=Release&logo=github)](https://github.com/XingChenRS/SakiSU/releases/latest)
[![频道](https://img.shields.io/badge/Follow-Telegram-blue.svg?logo=telegram)](https://t.me/SakiSU)
[![协议: GPL v2](https://img.shields.io/badge/License-GPL%20v2-orange.svg?logo=gnu)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
[![GitHub 协议](https://img.shields.io/github/license/tiann/KernelSU?logo=gnu)](/LICENSE)

## 特性

1. 基于内核的 `su` 和 root 权限管理。
2. 基于 [Magic Mount](https://github.com/5ec1cff/KernelSU) 的模块系统。
   > 注意：模块挂载已由元模块接管。
3. [App Profile](https://kernelsu.org/zh_CN/guide/app-profile.html)。
4. 面向 GKI 2.0 设备，并保留 non-GKI 与 GKI 1.0 相关能力。
5. KPM 支持。
6. vivo/iQOO 兼容模式：一键移除 `vendor_boot` 中的 `vr.ko`，或在 boot/init_boot 修补时适配 vivo 官方内核 LKM。

## vivo/iQOO 适配

Manager 中的开关文案是 **“去除vr或适配vivo特性”**。它不是单纯的 rmvr 开关，而是按你选择的镜像自动走不同路径：

| 选择的镜像 | SakiSU 行为 |
|---|---|
| `vendor_boot.img` | 自动识别 vendor ramdisk，移除 `vr.ko` 及 `modules.*` 引用，不注入 KernelSU LKM。 |
| `init_boot.img` 或兼容 boot ramdisk | 正常注入 KernelSU LKM，并优先使用 `_vivo` KMI/LKM 变体。 |

这两个操作互不依赖。你可以只修补 `vendor_boot` 去除 `vr.ko`，也可以只修补 `init_boot` 注入 KernelSU；如果使用 vivo 官方内核，通常需要选择带 `_vivo` 后缀的 KMI。

完整背景、风险说明和操作教程见 [vivo/iQOO 适配教程](./vivo.md)。

## 兼容状态

- SakiSU 主要面向 Android GKI 2.0 设备（内核 5.10+）。
- 旧内核路径继承自上游，但 3.x/4.x 时代的 vivo 内核内嵌反 root 不在 SakiSU 当前处理范围内。
- 目前支持 `arm64-v8a`、`armeabi-v7a`，以及部分 `x86_64`。
- vivo/iQOO 的 `vr.ko` 移除面向存在 `vendor_boot` 的 GKI 时代设备；`_vivo` LKM 适配面向 vivo 官方内核的 vermagic 机制。

## 快速使用

1. 解锁 bootloader，并备份原始 `boot`、`init_boot`、`vendor_boot` 等分区。
2. 从当前系统包或设备中提取对应镜像，不要混用不同系统版本的镜像。
3. 打开 SakiSU Manager，进入安装页，按需开启 vivo 修补。
4. 修补 `vendor_boot.img` 时，正常情况下不会弹出 KMI 选择，输出镜像刷回 `vendor_boot`。
5. 修补 `init_boot.img` 时，选择适配本机的 KMI；vivo 官方内核优先选择 `_vivo` 后缀。

详细步骤见 [vivo/iQOO 适配教程](./vivo.md)。

## 构建与测试

`dev` 分支用于 main 合入前测试。当前 CI 在 push 时验证 Manager APK、`ksud`、`ksuinit`、标准 LKM、vivo LKM、Rustfmt、Clippy 与 clang-format。

签名逻辑按现有代码执行；除非内核侧校验策略再次变化，否则不回退到旧的强制 v2-only 假设。

## 许可证

- `kernel` 目录下文件为 [GPL-2.0-only](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)。
- 除特别声明外，其余部分为 [GPL-3.0-or-later](https://www.gnu.org/licenses/gpl-3.0.html)。
- 启动图标和动漫风格素材保留原作者授权与使用限制。

## 鸣谢

- [ReSukiSU/ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)：上游
- [SukiSU-Ultra/SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra)：上游血统
- [KernelSU](https://github.com/tiann/KernelSU)：内核级 root 方案基础
- [Magic Mount](https://github.com/5ec1cff/KernelSU)：模块挂载相关血统
- [KernelPatch](https://github.com/bmax121/KernelPatch)：KPM/APatch 相关内核模块工作
