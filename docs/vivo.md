# vivo/iQOO Compatibility Guide

[简体中文](./zh/vivo.md) | [Back to docs](./README.md)

This document summarizes the current SakiSU behavior for vivo/iQOO devices. The Chinese version contains more historical background; this page focuses on user-facing behavior and implementation rules.

## Read This First

- Unlocking the bootloader and flashing boot-related partitions can brick the device or wipe data. Back up the original `boot`, `init_boot`, and `vendor_boot` images first.
- Do not mix images from different firmware versions.
- A bad `vendor_boot` image can affect normal boot, recovery, and fastbootd. Keep the original image ready for recovery.
- Early 3.x/4.x vivo anti-root implementations embedded in vendor kernels are outside SakiSU's current scope.

## What vivo Mode Means

The manager switch is named **"去除vr或适配vivo特性"**. When enabled, Manager invokes:

```text
ksud boot-patch-vivo
```

`ksud` then decides what to do from the image content:

| Image | Behavior |
|---|---|
| `vendor_boot.img` | Detect vendor ramdisk modules, remove `vr.ko` and its `modules.*` references, and skip KernelSU LKM injection. |
| `init_boot.img` or boot ramdisk | Keep normal KernelSU LKM injection and prefer the `_vivo` KMI/LKM variant. |

The two operations are independent. You can remove `vr.ko` without injecting KernelSU, or inject KernelSU without touching `vendor_boot`.

## Why `_vivo` KMI Exists

On many vivo 5.x-6.1 GKI-era devices, `vendor_boot` contains module sets for both the official kernel and the generic GKI kernel. vivo official kernels distinguish them with `vermagic`; a module built without the `vivo` marker may fail to load on the official kernel.

SakiSU provides `_vivo` LKM variants for that path. When vivo mode is enabled, Manager prefers `_vivo` KMI values and appends `_vivo` before invoking `ksud` if needed.

## Manager Workflow

### Remove `vr.ko` from `vendor_boot`

1. Open SakiSU Manager.
2. Enable **vivo修补**.
3. Choose file patching and select the matching `vendor_boot.img`.
4. Normally no KMI dialog is shown.
5. Flash the output image back to `vendor_boot`:

```text
fastboot flash vendor_boot kernelsu_patched_vivo_xxx.img
fastboot reboot
```

### Patch `init_boot` with KernelSU LKM

1. Open SakiSU Manager.
2. Enable **vivo修补**.
3. Choose file patching and select the matching `init_boot.img` or compatible `boot.img`.
4. If a KMI dialog appears, choose the matching `_vivo` entry for the official vivo kernel.
5. Flash the output image back to the corresponding partition:

```text
fastboot flash init_boot kernelsu_patched_vivo_xxx.img
fastboot reboot
```

Use `boot` instead of `init_boot` only on devices that actually use the `boot` partition for this image.

## Backend Rules

The implementation is kept partition-agnostic:

1. `patch_vivo()` adds `vr.ko` to the remove list.
2. The normal `patch()` path parses the boot image and ramdisk.
3. If the cpio contains `lib/modules/*.ko`, SakiSU treats it as `vendor_boot`, skips LKM injection, and removes `vr.ko` from every discovered module root.
4. `remove_vendor_modules()` scans both `lib/modules` and `lib/modules/<version>-gki`, cleaning `modules.load`, `modules.dep`, `modules.softdep`, and `modules.load.recovery`.
5. If the image is not vendor ramdisk, SakiSU continues with normal LKM injection.

The helper command:

```text
ksud boot-info classify-image <image>
```

prints `vendor_boot`, `init_boot`, or `unknown`. Manager uses it to avoid unnecessary KMI dialogs for clear `vendor_boot` images.

