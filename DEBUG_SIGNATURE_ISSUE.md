## Debug vs Release 签名信任问题诊断

### 问题分析

**现象**: 
- ✅ debug APK 可以获得权限，LKM 工作正常
- ❌ release APK 无法获得权限

**根本原因**: 内核中硬编码的签名与 release 版本不匹配

### 签名验证流程

```
Kernel apk_sign.c: apk_sign_keys[] 数组
├─ EXPECTED_SIZE_RESUKISU=0x377, EXPECTED_HASH_RESUKISU=d3469712b6...
├─ 其他官方签名（如果启用 CONFIG_KSU_MULTI_MANAGER_SUPPORT）
├─ EXPECTED_PR_BUILD_SIZE/HASH（CI 编译时注入）
└─ 自定义 EXPECTED_SIZE/HASH（编译时定义）

Manager App 验证:
├─ KsuCli.isOfficialSignature() 调用 ksud debug get-sign
├─ 与 trustedManagerSignatures() 比对
├─ BuildConfig.TRUSTED_MANAGER_CERT_* 应该包含发行版本签名
└─ 但 gradle.properties 中此字段为空！
```

### 为什么 debug 工作

Debug APK:
- 文件大小: 0x377 (887 字节)
- 签名哈希: d3469712b6214462764a1d8d3e5cbe1d6819a0b629791b9f4101867821f1df64
- **完全匹配** kernel/manager/manager_sign.h 中的 EXPECTED_SIZE_RESUKISU
- 因此内核自动信任

### 为什么 release 不工作

Release APK:
- 启用了 minify、resource shrinking 等优化
- 文件大小与 debug 不同，不是 0x377
- 签名哈希也不同（不同的签名密钥或优化导致）
- **不匹配**任何硬编码的签名
- 内核拒绝授予权限

### 解决方案

**方案 A: 将 Release 签名添加到内核硬编码列表（推荐）**

1. 构建 release APK 并获取其签名:
```bash
cd manager/
./gradlew assembleRelease
# 获取 app/build/outputs/apk/release/*.apk
```

2. 使用 ksud 获取签名:
```bash
adb shell ksud debug get-sign /path/to/release.apk
# 输出: size: 0xXXX, hash: abcd1234...
```

3. 更新 kernel/manager/manager_sign.h:
```c
// 在 EXPECTED_SIZE_RESUKISU 下方添加
#define EXPECTED_SIZE_RELEASE 0xXXX     // 从上面获取
#define EXPECTED_HASH_RELEASE "abcd1234..."
```

4. 更新 kernel/manager/apk_sign.c 的 apk_sign_keys 数组:
```c
static apk_sign_key_t apk_sign_keys[] = {
    { EXPECTED_SIZE_RESUKISU, EXPECTED_HASH_RESUKISU },
    { EXPECTED_SIZE_RELEASE, EXPECTED_HASH_RELEASE },  // 添加这行
    #ifdef CONFIG_KSU_MULTI_MANAGER_SUPPORT
    ...
};
```

5. 重新编译内核并测试

**方案 B: 修复 BuildConfig 证书注入（完整方案）**

1. 确保 build-manager.yml 的证书导出正确写入 gradle.properties:
```bash
# 验证 gradle.properties 已更新
cat manager/gradle.properties | grep TRUSTED_MANAGER_CERT
```

2. 如果为空，修复 build-manager.yml 的导出步骤:
```yaml
- name: Export trusted manager cert metadata
  run: |
    # ... 证书导出逻辑 ...
    SIZE_HEX=$(printf '0x%04x' "$SIZE_DEC")
    # 重要: 追加到 gradle.properties (不要覆盖)
    echo "" >> gradle.properties
    echo TRUSTED_MANAGER_CERT_SIZE="$SIZE_HEX" >> gradle.properties
    echo TRUSTED_MANAGER_CERT_HASH="$HASH" >> gradle.properties
```

3. 验证 BuildConfig 被正确注入:
```bash
./gradlew assembleRelease
# 查看生成的 BuildConfig.java
cat app/build/generated/source/buildConfig/release/com/resukisu/resukisu/BuildConfig.java | grep TRUSTED
```

4. 测试 APK

### 建议的完整修复步骤

1. **立即测试方案 A（快速）**:
   - 获取当前 release APK 的签名
   - 添加到内核硬编码列表
   - 重新编译内核
   - 测试

2. **同时准备方案 B（长期）**:
   - 修复 gradle.properties 证书注入机制
   - 验证所有构建类型（debug/release）都能正确注入
   - 确保 CI 构建在生成 APK 后能自动获取签名

### 调试命令

```bash
# 获取 APK 签名
adb shell ksud debug get-sign /data/app/*/base.apk

# 查看内核日志
adb shell dmesg | grep -i "apk_sign\|signature\|manager"

# 检查 logcat
adb logcat | grep "KsuCli\|isOfficialSignature"
```
