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

因此 vivo 设备需要两步、两种镜像：

| 步骤 | 镜像 | 行为 |
|---|---|---|
| ① 装 KernelSU | `init_boot.img` | 注入 `kernelsu.ko` + `ksuinit`，KMI 用 `_vivo` 变体 |
| ② 防 boot loop | `vendor_boot.img` | **仅** 移除 `vr.ko` 及 `modules.*` 中的引用，**不**写入任何 KernelSU 制品 |

**关键设计原则**：单一开关（`vivoPatch`），单一子命令（`boot-patch-vivo`），由 ksud 根据 cpio 内容自动分流。
**不要**根据分区名字预先决定路径，否则 `SelectFile` 流程因为隐藏了分区下拉框就会全部失灵。

vivo 开关关闭时 ⇒ 完全等同于标准 SakiSU 流程，没有任何分支差异。

## 2. 后端：`userspace/ksud/src/boot_patch.rs`

`patch_vivo()` 现在做且仅做：

1. 把 `vr.ko` 加入 `remove_module` 列表（去重，存在则 no-op）。
2. 调用 `patch()`。

**不再**强制 `partition` 或 `no_install`。所有判别下沉到 `patch()`。

`patch()` 内的关键改造（cpio 加载之后、LKM 资源加载之前）：

```rust
if !no_install {
    let looks_like_vendor = cpio
        .entries()
        .keys()
        .any(|p| p.starts_with("lib/modules/") && p.ends_with(".ko"));
    if looks_like_vendor {
        println!("- Auto-detected vendor_boot ...; skipping LKM injection");
        no_install = true;
    }
}
```

→ 只要 cpio 里有 `lib/modules/*.ko` 就判定为 vendor_boot 镜像，自动跳过 LKM 注入。
init_boot / boot 镜像不含 `lib/modules/`，自然走标准注入路径。

LKM 资源加载（`assets::get_asset("ksuinit")` 等）已被移到 cpio 加载和自动检测**之后**，
避免 vendor_boot 路径白白把 ~3MB 的 ko/init 从 RustEmbed 拉出来再丢弃。

`remove_vendor_modules()` 动态扫描 cpio 里所有 `lib/modules/<X.YZ>-gki/` 子目录
（之前硬编码 `["lib/modules", "lib/modules/6.1-gki"]`，导致 5.10 / 5.15 / 6.6 设备完全没生效）：

```rust
let mut module_roots: Vec<String> = vec!["lib/modules".to_string()];
for path in cpio.entries().keys() {
    if let Some(rest) = path.strip_prefix("lib/modules/") {
        let head = rest.split('/').next().unwrap_or("");
        if head.ends_with("-gki") && !module_roots.iter().any(|r| r.ends_with(head)) {
            module_roots.push(format!("lib/modules/{head}"));
        }
    }
}
```

随后对每个 root 处理 `modules.load`/`modules.dep`/`modules.softdep`/`modules.load.recovery`，
删除含 `vr.ko` / `vr` token 的行；模块文件本身从 cpio 移除。

## 3. 前端：`manager/app/.../ui/util/KsuCli.kt`

`installBoot()` 的命令选择简化为单一开关：

```kotlin
val useVivoCompat = vivoPatch
var cmd = if (useVivoCompat) "boot-patch-vivo" else "boot-patch"
```

**不再**读 `partition` 字段做判断。SelectFile 路径下 `partition == null` 也能正确工作。

KMI 选择：`useVivoCompat` 为真且当前 KMI 不带 `_vivo` 时附加后缀。
对 vendor_boot 镜像而言这是无害的（ksud 自动检测后 `no_install=true`，不会真的去取 KMI 对应资源）。

## 4. 前端：`manager/app/.../ui/screen/Install.kt`

- `enableVivoPatch` toggle：用户控制，永久开启即可（vivo 设备不需要每次切换）。
- `partitionSelection`：传 `partitionsState.getOrNull(idx)`（DirectInstall 下拉框选什么传什么），**不**因 `enableVivoPatch` 强制覆盖。
- `preferVivoKmi = enableVivoPatch`：vivo ON 时 KMI 弹框默认带 `_vivo` 后缀。
- `onClickNext`：按标准逻辑判断 `needsKmi`（GKI + 还没选 KMI + 不是 HorizonKernel），**不**因为 `enableVivoPatch` 跳过 KMI 弹框。即使用户挑的是 vendor_boot.img，KMI 选了也会被 ksud 自动判别后忽略，KMI 弹框成本远小于丢失 init_boot 路径入口的代价。

## 5. 完整流程

vivo 用户的两次刷机操作，均在 manager 内 vivo 开关常开状态下完成：

```
[1] Install -> SelectFile -> 选 init_boot.img -> 弹 KMI 选 androidXX-Y.Z_vivo -> 修补 -> 刷
    ksud 输出:
      - Mode: vivo compat (auto-detect init_boot vs vendor_boot)
      - Adding KernelSU LKM
      - KMI: androidXX-Y.Z_vivo
      ... (标准注入流程)

[2] Install -> SelectFile -> 选 vendor_boot.img -> 弹 KMI 随便选 (会被忽略) -> 修补 -> 刷
    ksud 输出:
      - Mode: vivo compat (auto-detect init_boot vs vendor_boot)
      - Auto-detected vendor_boot (lib/modules/*.ko present); skipping LKM injection
      - Skipping KernelSU LKM injection (no_install)
      - Detected vendor module root: lib/modules/<gki-version>-gki
      - Removing vendor module lib/modules/<gki-version>-gki/vr.ko
      - Cleaning reference in lib/modules/<gki-version>-gki/modules.load for vr.ko
      - Cleaning reference in lib/modules/<gki-version>-gki/modules.softdep for vr.ko
      ... (重打包 vendor_boot)
```

## 6. 历史踩坑教训

- **不能根据 partition 名字预判路径**。Manager 的分区下拉框只在 `DirectInstall*` 流程显示，`SelectFile` 隐藏；早期实现 `useVivoRmvr = vivoPatch && partition == "vendor_boot"` 导致 SelectFile 用户永远走不到 rmvr。
- **不能让 vivo 开关 = "只 rmvr"**。这样会切掉 init_boot 的 LKM 注入入口，用户没办法装 KernelSU。
- **rmvr 模块根目录不能硬编码 GKI 版本**。`["lib/modules", "lib/modules/6.1-gki"]` 漏掉了所有非 6.1 设备。
- **同期修复了 `kernel/manager/throne_tracker.c::my_actor()` 的 `strncmp(name, "base.apk", 8)` 前缀匹配 bug**——它会把同目录的 `base.apk.prof` 当成 APK 走签名校验，导致用户应用全部 `is_manager=0`，与 vivo 适配相互掩盖了一段时间。修复：`namelen == 8 && memcmp(...)`。


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
