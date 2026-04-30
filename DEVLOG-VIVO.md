# SakiSU Vivo 设备兼容性开发日志

## 概述

本文档记录了 SakiSU 针对 vivo/iQOO 设备的兼容性修复工作。SakiSU 是 KernelSU 针对 vivo GKI 内核的专用分支，主要解决包列表解析、管理器签名验证、KMI（KernelSU Module Info）选择等问题。

---

## Vivo 适配时间线

### Phase 0: 基础设施准备 (2026-04-30 22:19 ~ 22:40)

#### Commit: `442e45fc` - fix(vivo): add rmvr vendor_boot patch flow and harden manager/ksud paths
**日期**: 2026-04-30 22:19:23  
**目标**: 建立vivo独特的vendor_boot补丁流程  
**修改内容**:
- 添加rmvr (removable vendor_boot) 补丁流程
- 加强管理器和ksud路径处理
- 为vivo设备区分不同的启动补丁策略

#### Commit: `6f3494f0` - feat(manager): split vivo switch behavior by partition (rmvr vs vivo kmi)
**日期**: 2026-04-30 22:40:17  
**目标**: 根据分区类型区分vivo行为  
**修改内容**:
- 在Install.kt中实现分区感知的vivo行为分支
- 区分rmvr (removable vendor_boot) 和 vivo kmi (KMI变体) 两种补丁方式
- 为vendor_boot和system_boot分别设置不同的KMI选择逻辑

**关键代码位置**:
- `manager/app/src/main/java/com/resukisu/resukisu/ui/screen/Install.kt:enableVivoPatch`
- 分区选择逻辑在第260-275行

---

### Phase 1: 包列表解析修复 (2026-04-30 22:43 ~ 23:10)

#### Commit: `cf1a3998` - fix(manager): robust packages.list uid parsing for vivo format
**日期**: 2026-04-30 22:43:01  
**目标**: 修复vivo格式packages.list的UID解析  
**背景**:
- vivo设备的packages.list格式与标准Android不同
- 包含大量短格式共享UID记录，导致解析失败

#### Commit: `57c8cc43` - fix(manager): use sscanf to parse packages.list, handle consecutive whitespace
**日期**: 2026-04-30 22:54:55  
**目标**: 使用sscanf替代strsep处理多空格  
**问题根因**:
- 原始代码使用`strsep(buf, " \t")`逐个分隔符处理
- 当存在连续多个空格时，strsep会产生空字符串，导致UID解析失败
- 测试数据: 用户packages.list有442行，其中419+行是短格式共享UID记录

**修改内容**:
```c
// 旧方式 (throne_tracker.c)
char *p = buf;
char *package = strsep(&p, " \t");
char *uid_str = strsep(&p, " \t");  // 多空格时可能为空

// 新方式
sscanf(buf, "%255s %u", data->package, &res)  // 自动跳过多个空白字符
```

**关键文件**: `kernel/manager/throne_tracker.c:340-355`

#### Commit: `bfa64efb` - fix(manager): force manager rescan when no registered uid in packages.list
**日期**: 2026-04-30 23:10:08  
**目标**: 当packages.list中未找到已注册UID时强制重新扫描  
**逻辑**:
- 在`do_track_throne()`中检查是否有任何已注册的UID
- 如果没有找到 → 触发强制扫描，寻找管理器APK
- 防止包列表快照为空时管理器权限丢失

**关键文件**: `kernel/manager/throne_tracker.c:379-435`

---

### Phase 2: 管理器签名与权限验证 (2026-04-30 23:35 ~ 23:47)

#### Commit: `a14803b7` - fix(manager): trust CI-signed builds and prefer vivo KMI
**日期**: 2026-04-30 23:35:47  
**目标**: 建立CI构建签名信任链 + vivo KMI优先选择  
**修改内容**:

1. **签名信任链** (`manager/app/src/main/java/com/resukisu/resukisu/ui/util/KsuCli.kt`):
   - 添加`trustedManagerSignatures()`辅助函数
   - 从BuildConfig读取CI证书元数据 (TRUSTED_MANAGER_CERT_SIZE/HASH)
   - 内置硬编码的官方签名
   
2. **Vivo KMI优先选择** (`manager/app/src/main/java/com/resukisu/resukisu/ui/screen/Install.kt`):
   - `rememberSelectKmiDialog`接受`preferredKmi`参数
   - vivo模式下自动优先推荐`${currentKmi}_vivo`变体
   - 重新排序KMI列表，将优先项放在最前
   
3. **界面改进**:
   - 从HomePage移除"PR构建"警告卡
   - 降低`MINIMAL_SUPPORTED_KERNEL`至30700，扩展支持范围
   
4. **构建流程**:
   - build-manager.yml新增证书导出步骤
   - 计算证书大小和SHA256哈希
   - 写入gradle.properties供编译时注入

**关键文件**:
- `manager/app/src/main/java/com/resukisu/resukisu/ui/util/KsuCli.kt:30-41`
- `manager/app/src/main/java/com/resukisu/resukisu/ui/screen/Install.kt:264-275`
- `.github/workflows/build-manager.yml:198-221`

**预期效果**:
- ✅ HomePage不显示"非官方管理器"警告
- ✅ 安装界面自动预选vivo KMI

#### Commit: `c5736b04` - fix(manager): normalize CI signature trust and commit KMI selection immediately
**日期**: 2026-04-30 23:47:54  
**目标**: 规范化签名格式 + KMI选择立即提交  
**修改内容**:

1. **签名格式规范化** (`KsuCli.kt`):
   - 新增`normalizeManagerSignature()`函数
   - 将`size: 0x0377`转换为`size:$size`格式（移除0x前缀）
   - 处理哈希大小写不一致
   
2. **KMI选择流程** (`Install.kt:1123`):
   - 修改ListDialog的选择回调
   - 用户点击KMI项 → 立即调用`onSelected()` + `dismiss()`
   - 目标: 防止对话框重复弹出

**关键代码**:
```kotlin
// KMI对话框选择回调
{ _, option ->
    selection = option.titleText
    onSelected(selection)  // 立即处理选择
    dismiss()              // 关闭对话框
}
```

**预期效果**:
- ✅ KMI对话框选择后立即关闭
- ✅ 进入下一步安装流程

---

### Phase 3: 问题诊断与调试日志 (2026-05-01 00:01)

#### Commit: `122a097e` - debug: add detailed logging for signature verification and KMI dialog flow
**日期**: 2026-05-01 00:01:49  
**目标**: 诊断两个待解决的问题  
**背景问题**:
1. ❌ "非官方管理器"警告依然显示（c5736b04后）
2. ❌ vivo KMI对话框反复弹出（c5736b04后）

**修改内容**:

1. **KsuCli.kt调试日志**:
   - `trustedManagerSignatures()`: 打印BuildConfig证书字段值
   - `isOfficialSignature()`: 打印ksud返回的原始签名、规范化后的签名、匹配结果
   
2. **Install.kt调试日志**:
   - KMI对话框选择回调: 记录选中项
   - `onClickNext`条件: 打印lkmSelection状态
   - 对话框生命周期: 记录`onFinishedRequest`和`onCloseRequest`调用

**日志示例**:
```
[DEBUG] BuildConfig.TRUSTED_MANAGER_CERT_SIZE='...'
[DEBUG] ksud debug get-sign raw output='...'
[DEBUG] isOfficialSignature result: false
[DEBUG] KMI selected: ...
[DEBUG] KMI dialog item selected
```

**诊断步骤** (待执行):
1. 构建并安装最新APK (commit 122a097e)
2. 使用`adb logcat | grep "DEBUG|KsuCli|Install"`收集日志
3. 分析BuildConfig注入是否成功
4. 追踪KMI对话框回调顺序

---

## 已知问题与排查

### 问题 #1: Release 版本权限获取失败（关键发现！）

**表现**: 
- ✅ Debug APK 可以获得权限，LKM 工作
- ❌ Release APK 无法获得权限

**根本原因** (已确认):
内核中硬编码的签名验证与 release 版本不匹配
- Debug APK 文件大小: 0x377 (887 字节) → **精确匹配** `EXPECTED_SIZE_RESUKISU`
- Release APK 文件大小: ≠ 0x377 → **不匹配** → 拒绝权限

**核心代码位置**:
- 内核硬编码签名: `kernel/manager/manager_sign.h` (第24行)
- 签名验证: `kernel/manager/apk_sign.c:apk_sign_keys[]`

**解决方案**:
1. 获取 release APK 签名:
   ```bash
   adb shell ksud debug get-sign /path/to/release.apk
   # 输出: size: 0xXXX, hash: abcd1234...
   ```
2. 将签名添加到内核硬编码列表 (`manager_sign.h`)
3. 重新编译内核

**详细方案**: 见 DEBUG_SIGNATURE_ISSUE.md

### 问题 #2: "非官方管理器"警告持续显示

**表现**: HomePage仍显示红色警告卡"非官方管理器"  
**根本原因**: BuildConfig 证书字段未注入
- gradle.properties 中 TRUSTED_MANAGER_CERT_* 为空
- build-manager.yml 的证书导出可能未正确执行或写入位置错误
- 即使在 CI 构建时生成，本地 gradle.properties 仍为空

**依赖于**: 问题 #1 的解决
- 如果 release 签名已添加到内核列表，可用方案 A
- 长期方案: 修复 gradle.properties 证书注入，使用方案 B

### 问题 #3: Vivo KMI对话框反复弹出

**表现**: 选择KMI项后，对话框关闭但随即重新弹出  
**可能原因**:
- [ ] `onFinishedRequest`和`selection lambda`都被执行，导致双重调用
- [ ] `lkmSelection`状态在重新组合时丢失
- [ ] `onClickNext`条件未正确更新，再次满足显示条件
- [ ] Compose重新组合导致对话框状态重置

**排查方法**:
```bash
# 观察日志顺序
adb logcat | grep -E "KMI dialog|lkmSelection|onClickNext"

# 检查状态转换
# 应该看到: item selected → onSelected() → dismiss → (不应该再show)
```

---

## 关键发现 (2026-05-01)

### Debug vs Release 签名验证不匹配

**问题**: Release APK无法获得权限，Debug APK可以  

**诊断**: 通过代码审查发现：
1. 内核硬编码了官方签名列表 (`kernel/manager/manager_sign.h`)
2. Debug APK 大小恰好是 `EXPECTED_SIZE_RESUKISU = 0x377`
3. Release APK 大小不同（minify导致），不匹配硬编码值
4. BuildConfig 中的 TRUSTED_MANAGER_CERT_* 字段也未被正确注入

**验证步骤**:
```
1. kernel/manager/apk_sign.c 行33: apk_sign_keys[] 数组
2. kernel/manager/manager_sign.h 行24: EXPECTED_SIZE_RESUKISU = 0x377
3. manager/gradle.properties: 完全缺少 TRUSTED_MANAGER_CERT_* 字段
```

**修复策略** (优先级):
- 🔴 **P0** - 添加 Release 签名到内核硬编码列表
- 🟡 **P1** - 修复 gradle.properties 证书注入机制
- 🟢 **P2** - 完成 KMI 对话框重复弹出诊断

---

### 包列表解析流程

```
kernel/manager/throne_tracker.c:do_track_throne()
├─ 打开 /data/system/packages.list
├─ 逐行解析 (现在使用 sscanf)
│  ├─ 成功: 提取 package name + UID
│  └─ 失败: 跳过（记录日志）
├─ 查找已注册的管理器UID
└─ 若无找到: 触发管理器扫描
```

### 管理器权限验证流程

```
manager/ui/util/KsuCli.kt:isOfficialSignature()
├─ 执行: ksud debug get-sign <apk-path>
├─ 获取: 签名字符串 (size: 0x377, hash: ...)
├─ 规范化: normalizeManagerSignature()
├─ 比对: trustedManagerSignatures() 集合
└─ 返回: Boolean
   ├─ true  → HomePage不显示警告
   └─ false → 显示"非官方管理器"警告
```

### Vivo KMI选择流程

```
manager/ui/screen/Install.kt:InstallScreen()
├─ enableVivoPatch = isVivoFamilyDevice() && vivo补丁启用
├─ preferVivoKmi = enableVivoPatch && partition != "vendor_boot"
├─ rememberSelectKmiDialog()
│  ├─ preferredKmi 设置为 "${currentKmi}_vivo" (若启用)
│  └─ 重新排序KMI列表 (优先项在前)
├─ 用户选择 → onSelected() → lkmSelection 更新 → onInstall()
└─ 后续步骤
```

---

## 关键文件映射

| 功能 | 文件路径 | 关键行 |
|------|---------|--------|
| packages.list解析 | `kernel/manager/throne_tracker.c` | 310-375 |
| 包UID查询 | `kernel/manager/throne_tracker.c` | 379-435 |
| 签名验证 | `manager/.../ui/util/KsuCli.kt` | 30-60, 146-160 |
| KMI选择UI | `manager/.../ui/screen/Install.kt` | 260-280, 1085-1135 |
| CI构建流程 | `.github/workflows/build-manager.yml` | 173-221 |

---

## 构建与测试

### 本地构建
```bash
cd manager/
./gradlew assembleRelease -Pcommit=$(git rev-parse HEAD)
```

### 设备安装
```bash
adb install -r build/outputs/apk/release/app-release.apk
```

### 日志收集
```bash
# 完整日志
adb logcat > full_logs.txt &

# 过滤关键日志
adb logcat | grep -E "KsuCli|Install|throne_tracker|isOfficialSignature"
```

---

## 下一步行动项

- [ ] 执行调试日志收集（使用commit 122a097e）
- [ ] 分析BuildConfig注入是否成功
- [ ] 确认ksud签名返回格式
- [ ] 修复KMI对话框重复弹出
- [ ] 验证非官方警告消失
- [ ] 整合upstream KernelSU的packages.list解析方法
- [ ] 生成vivo正式签名证书（替代CI临时证书）

---

**最后更新**: 2026-05-01 00:01  
**维护者**: GitHub Copilot  
**状态**: 调试中 (Phase 3)
