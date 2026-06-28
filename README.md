# SakiSU

<img align="right" src="docs/SakiSU_blue.svg" width="220px" alt="SakiSU Icon">

**简体中文** | [English](docs/README.md) | [vivo/iQOO 适配教程](docs/zh/vivo.md)

SakiSU 是基于 [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) 的下游分支，保留 KernelSU/SukiSU 的主要能力，并重点补充 vivo/iQOO 设备上的内核级 root 适配。

当前开发与测试主要在 `dev` 分支进行；稳定后再整理并合入 `main`。

## 重点功能

- KernelSU/SukiSU 系 root 管理、模块系统和 App Profile。
- 支持 GKI 2.0 设备，并保留 non-GKI/GKI 1.0 相关能力。
- vivo/iQOO 兼容模式：同一个开关完成 `vr.ko` 移除或 vivo 官方内核 LKM 适配。
- Manager 构建沿用现有签名策略，CI 会在 dev 上验证 manager、ksud、ksuinit、LKM 与格式检查。

## vivo/iQOO 快速说明

Manager 中的 vivo 开关含义是“去除vr或适配vivo特性”。

- 选择 `vendor_boot.img` 时，SakiSU 会自动识别 vendor ramdisk，移除 `vr.ko` 及其 `modules.*` 引用，不注入 KernelSU LKM。
- 选择 `init_boot.img` 或兼容 boot ramdisk 时，SakiSU 走正常 LKM 注入流程，并优先使用 `_vivo` KMI/LKM 变体以适配 vivo 官方内核的 vermagic 机制。
- 两个操作互不依赖：你可以只移除 `vr.ko`，也可以单独修补 `init_boot`，实际刷入哪个分区取决于你选择的镜像。

完整背景、风险说明和操作教程见 [docs/zh/vivo.md](docs/zh/vivo.md)。

## 文档

- [中文文档](docs/zh/README.md)
- [English documentation](docs/README.md)
- [vivo/iQOO 适配教程](docs/zh/vivo.md)
- [实现记录](DEVLOG-VIVO.md)

## 鸣谢

- [ReSukiSU/ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)：上游
- [SukiSU-Ultra/SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra)：上游血统
- [KernelSU](https://github.com/tiann/KernelSU)：内核级 root 方案基础

