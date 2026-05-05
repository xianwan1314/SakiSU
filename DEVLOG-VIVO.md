# DEVLOG: vivo / iQOO 修补流程

> 本文档只记录已经在代码里**实际生效**的事实，不写假设或历史推测。
> 旧的 `DEVLOG-VIVO.md` / `DEBUG_SIGNATURE_ISSUE.md` 已被验证为误判，全部删除。

## 1. 设计原则

vivo / iQOO GKI 设备的特点：

- **`init_boot`** 才包含真正的 ramdisk + init，KernelSU 的 `kernelsu.ko` 与 `ksuinit` 必须写到这里。
- **`vendor_boot`** 携带 vendor ramdisk，里面带有 vivo 自有的 `vr.ko`（vivo restrict / "vendor reject" 模块）。
  这个 ko 会在内核加载阶段做一些限制，必须在刷入前从 vendor ramdisk 中清理掉，且要把对它的引用从
  `modules.load` / `modules.dep` / `modules.softdep` / `modules.load.recovery` 一并去掉。
- vivo 的内核 vermagic 与 stock GKI 不同，对应的 KernelSU LKM 必须使用带 `_vivo` 后缀的 KMI 资源。

因此 vivo 设备需要两步、两条路径：

| 步骤 | 分区 | 命令 | 行为 |
|---|---|---|---|
| ① 清理 vendor 限制 | `vendor_boot` | `ksud boot-patch-vivo` | **仅** 移除 `vr.ko`，**不**写入任何 KernelSU 制品 |
| ② 注入 LKM | `init_boot` (或 `boot`) | `ksud boot-patch --kmi <ver>_vivo` | 注入 `kernelsu.ko` + `ksuinit`，使用 vivo vermagic 资源 |

vivo 开关关闭时 ⇒ 完全等同于标准 ReSukiSU 流程，没有任何分支差异。

## 2. 后端：`userspace/ksud/src/boot_patch.rs`

`patch_vivo()` 现在做且仅做：

1. 把 `vr.ko` 加入 `remove_module` 列表（去重）。
2. 强制 `partition = "vendor_boot"`。
3. 设 `no_install = true`，让通用的 `patch()` 跳过加载/写入 `kernelsu.ko` 与 `ksuinit`。
4. 调用 `patch()`。

为支持 (3)，`patch()` 在 `no_install == true` 时：

- 不再读取 `kernelsu.ko` / `ksuinit` 资源（之前的实现遇到 `no_install + 无 kmod/init` 会 `bail!("")` 直接报空错误）。
- 输出 `- Skipping KernelSU LKM injection (no_install)` 而不是误导性的 `- Adding KernelSU LKM`。
- 后续的 `remove_vendor_modules()` / 重打包 / 写回分区路径完全保留，因此 `vendor_boot` 仍会被正确刷入。

`patch()` 在 `no_install == false` 的标准路径下行为与改动前完全一致（同样的日志、同样的 cpio 操作）。

## 3. 前端：`manager/app/.../ui/util/KsuCli.kt`

`installBoot()` 的命令选择：

```
useVivoRmvr = vivoPatch && partition == "vendor_boot"
useVivoLkm  = vivoPatch && !useVivoRmvr
```

调用 `ksud` 之前会先通过 `onStdout` 打一行明确日志，便于刷机日志一眼看出走的是哪条路：

- `[manager] vivo mode: vendor_boot rmvr (no LKM injection)`
- `[manager] vivo mode: install vivo-vermagic LKM into <partition>`
- `[manager] standard ReSukiSU patch flow on <partition>`

KMI 选择：仅在 `useVivoLkm` 为真且当前 KMI 字符串不带 `_vivo` 时，才追加 `_vivo` 后缀。
vendor_boot rmvr 路径不会附加任何 `--kmi`，因为它根本不写 LKM。

## 4. 前端：`manager/app/.../ui/screen/Install.kt`

- `enableVivoPatch` 是用户开关，只在 GKI 流程下显示。
- 分区下拉默认走 `getDefaultPartition()`，用户手动改过之后 `hasCustomSelected = true` 锁定不再被覆盖。
- `preferVivoKmi = enableVivoPatch && partition != "vendor_boot"` —— vendor_boot rmvr 路径不需要"偏好 KMI"。
- KMI 弹窗 `rememberSelectKmiDialog`：
  - vivo 已开 + 已识别到 KMI ⇒ 偏好 `<currentKmi>_vivo`，列表第一项即偏好项，第二项 fallback 为不带后缀版本。
  - **未识别到 KMI**（`getCurrentKmi()` 返回空）⇒ 不传 `preferredKmi`，弹窗按 `getSupportedKmis()` 原顺序列出全部 KMI，由用户手动选择。这正好覆盖 "无法识别当前设备 KMI 时弹手动选择" 的需求。
- `onClickNext`：当 `lkmSelection == KmiNone` 时强制弹 KMI 选择对话框；选择后写入 `lkmSelection` 并直接 `onInstall()`。**没有循环**：弹窗只在 `KmiNone` 时弹，选完就不再 `KmiNone`。

## 5. 同批签名 (CI)

与 vivo 流程**正交**，但顺便记录确认结果：

- `.github/workflows/build-manager.yml` 的 `generate-key` job 生成临时 `pr-key.jks`，导出 DER 证书并算 size + sha256，输出为 `expected_size` / `expected_hash`。
- 这两个值通过 `expected_pr_build_size` / `expected_pr_build_hash` 透传给 `build-lkm.yml` 与 `build-lkm-vivo.yml`，最终在 `ddk-lkm.yml` 里以 `KSU_EXPECTED_PR_BUILD_SIZE/HASH` 环境变量被 `kernel/Kbuild` 读到，由 ccflags 注入 `apk_sign.c` 的 `apk_sign_keys[]`。
- 同一份 `pr-key.jks` 通过 `PR_KEYSTORE` secret 给 manager 模块签名，因此**同批 CI 产物的 ko ↔ APK 一定匹配**。

## 6. 真正的"release 拿不到 root"原因

不是签名 size/hash 不匹配，也不是 `BuildConfig` 误判，而是：

`kernel/manager/apk_sign.c` 第 319 行：

```c
if (v3_signing_exist || v3_1_signing_exist)
    return false;
```

内核硬性拒绝任何带有 v3 / v3.1 APK Signature Block 的 APK。AGP 默认 release 同时签 v1 + v2 + v3 + v4，debug 只签 v1 + v2 —— 这就解释了为什么 debug 能授权而 release 不能。

修复：`manager/app/build.gradle.kts` 通过 `afterEvaluate { signingConfigs.configureEach { ... } }` 对所有 signingConfig（debug + release）强制 `enableV3Signing = false; enableV4Signing = false`，只保留 v1 + v2。本地 Android Studio 与 GitHub Actions 行为一致。
