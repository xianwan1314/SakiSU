# SakiSU

<img align="right" src="SakiSU_blue.svg" width="220px" alt="SakiSU Icon">

**English** | [简体中文](./zh/README.md) | [vivo/iQOO guide](./vivo.md)

SakiSU is a downstream fork based on [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU). It keeps the KernelSU/SukiSU lineage while carrying SakiSU-specific compatibility work, especially for vivo/iQOO devices.

[![Latest release](https://img.shields.io/github/v/release/XingChenRS/SakiSU?label=Release&logo=github)](https://github.com/XingChenRS/SakiSU/releases/latest)
[![Channel](https://img.shields.io/badge/Follow-Telegram-blue.svg?logo=telegram)](https://t.me/SakiSU)
[![License: GPL v2](https://img.shields.io/badge/License-GPL%20v2-orange.svg?logo=gnu)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html)
[![GitHub License](https://img.shields.io/github/license/tiann/KernelSU?logo=gnu)](/LICENSE)

## Features

1. Kernel-based `su` and root access management.
2. Module system based on [Magic Mount](https://github.com/5ec1cff/KernelSU).
   > Note: module mounting is delegated to the installed metamodule.
3. [App Profile](https://kernelsu.org/guide/app-profile.html).
4. GKI 2.0 support, with inherited non-GKI and GKI 1.0 support paths.
5. KPM support.
6. vivo/iQOO compatibility mode: remove `vr.ko` from `vendor_boot` or use `_vivo` LKM variants for boot/init_boot patching.

## vivo/iQOO Compatibility

The manager switch is named **"去除vr或适配vivo特性"**. It intentionally covers two independent operations:

| Selected image | SakiSU behavior |
|---|---|
| `vendor_boot.img` | Detects vendor ramdisk content, removes `vr.ko` and its `modules.*` references, and skips KernelSU LKM injection. |
| `init_boot.img` or boot ramdisk | Keeps the normal KernelSU LKM injection path and prefers a `_vivo` KMI/LKM variant. |

See [vivo/iQOO guide](./vivo.md) for background, warnings, and step-by-step usage.

## Compatibility

- SakiSU is primarily aimed at Android GKI 2.0 devices with kernel 5.10+.
- Older kernels may still work through inherited build paths, but they usually require manual kernel work.
- Currently supported Android ABIs include `arm64-v8a`, `armeabi-v7a`, and partial `x86_64`.
- vivo/iQOO support has been tested around the GKI era. Early 3.x/4.x anti-root implementations embedded in vendor kernels are outside SakiSU's current scope.

## Build and CI

The `dev` branch is used for testing before mainline preparation. Push CI builds the manager, `ksud`, `ksuinit`, standard LKM assets, vivo LKM assets, and formatting checks.

Signing behavior follows the current code path. SakiSU does not reintroduce the old "force v2-only signing" assumption unless the verifier policy changes again.

## Documentation

- [Chinese documentation](./zh/README.md)
- [vivo/iQOO guide](./vivo.md)
- [vivo implementation notes](../DEVLOG-VIVO.md)

## License

- Files under `kernel` are licensed under [GPL-2.0-only](https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html).
- Other parts are licensed under [GPL-3.0-or-later](https://www.gnu.org/licenses/gpl-3.0.html), except where explicitly stated otherwise.
- Launcher and anime-styled image assets keep their original authorizations and restrictions.

## Credits

- [ReSukiSU/ReSukiSU](https://github.com/ReSukiSU/ReSukiSU): upstream
- [SukiSU-Ultra/SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra): upstream lineage
- [KernelSU](https://github.com/tiann/KernelSU): kernel-assisted root foundation
- [Magic Mount](https://github.com/5ec1cff/KernelSU): module mounting lineage
- [KernelPatch](https://github.com/bmax121/KernelPatch): KPM/APatch-related kernel module work
