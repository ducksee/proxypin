# ProxyPin 发布指南

## 概述

更新后的 GitHub Actions workflow 现在支持自动构建和发布多平台版本，包括：
- Android APK (arm64-v8a, armeabi-v7a, x86_64)
- Windows 可执行文件
- Linux 可执行文件
- macOS 应用程序
- iOS IPA 文件（无签名）

## 自动发布流程

### 1. 版本发布触发条件

当推送带有版本标签的 commit 时，会自动触发发布流程：

```bash
# 创建并推送版本标签
git tag v1.2.1
git push origin v1.2.1
```

### 2. 容错构建流程

系统采用**容错机制**，即使某个平台构建失败，也不会影响其他平台和发布流程：

1. **多平台并行构建**：
   - Android: 构建 3 个架构的 APK
   - Windows: 构建 Release 版本
   - Linux: 构建 Release 版本
   - macOS: 构建 Release 版本
   - iOS: 构建无签名 IPA 文件
   - ✨ **每个平台独立构建，互不影响**

2. **智能文件打包**：
   - 自动检测哪些平台构建成功
   - 只打包成功构建的平台文件
   - 失败的平台会在日志中显示跳过信息

3. **弹性 GitHub Release**：
   - 即使部分平台失败，仍会创建 Release
   - 只上传成功构建的文件
   - Release 页面显示各平台构建状态
   - 自动生成详细的构建报告

### 3. 发布的文件格式

每次发布会包含**成功构建**的平台文件：

- `ProxyPin-Windows-v{version}.zip` - Windows 版本
- `ProxyPin-Linux-v{version}.tar.gz` - Linux 版本  
- `ProxyPin-macOS-v{version}.zip` - macOS 版本
- `ProxyPin-Android-arm64-v8a-v{version}.apk` - Android ARM64 版本
- `ProxyPin-Android-armeabi-v7a-v{version}.apk` - Android ARMv7 版本
- `ProxyPin-Android-x86_64-v{version}.apk` - Android x86_64 版本
- `ProxyPin-iOS-v{version}.ipa` - iOS 版本（无签名）

> ⚠️ **注意**: 如果某个平台构建失败，对应的文件不会出现在 Release 中，但不影响其他平台的发布。

## 手动触发构建

除了标签触发外，还可以通过以下方式手动触发构建：

1. 在 GitHub 仓库页面进入 Actions 标签
2. 选择 "CI" workflow
3. 点击 "Run workflow" 按钮

## macOS 和 iOS 特别说明

### macOS 版本
- macOS 构建禁用了代码签名，生成无签名的应用程序
- 用户首次运行时可能需要在"系统偏好设置 > 安全性与隐私"中允许运行
- 或者通过命令行移除隔离属性：`xattr -rd com.apple.quarantine /path/to/ProxyPin.app`

### iOS 版本  
- iOS 构建使用模拟器目标，生成无签名的 IPA 文件
- 主要用于开发测试和代码分发
- 如需在真机上运行，用户需要：
  - 使用开发者证书重新签名
  - 或使用企业证书进行分发
  - 或通过 Xcode 重新构建并安装

## 版本号管理

版本号应该遵循语义版本控制 (Semantic Versioning)：
- `v1.0.0` - 主版本号.次版本号.修订号
- 例如：`v1.2.1`, `v2.0.0`, `v1.3.0`

## 容错机制详解

### 构建失败处理
- **单平台失败**: 其他平台继续构建，不受影响
- **多平台失败**: 只要有一个平台成功，就会创建 Release
- **全部失败**: Release 任务仍会运行，但不会上传任何文件

### 构建状态监控
每个 Release 页面都会显示详细的构建状态：
```
构建状态
- ✅ Android: 成功
- ❌ Windows: 失败  
- ✅ Linux: 成功
- ✅ macOS: 成功
- ❌ iOS: 失败
```

## 故障排除

### 查看构建状态
1. 进入 GitHub 仓库的 Actions 页面
2. 找到对应的 workflow 运行记录
3. 查看各个平台的构建日志
4. 检查 Release 页面的构建状态报告

### 常见问题
1. **标签格式错误**: 确保使用 `v*.*.*` 格式 (如 `v1.2.1`)
2. **依赖问题**: 检查各平台特定的依赖是否满足
3. **签名问题**: macOS 和 iOS 使用无签名构建
4. **版本号**: 建议同步更新 pubspec.yaml 中的版本号

### 重试机制
- 如果某个平台偶发失败，可以重新推送相同标签触发重试
- 或者使用 GitHub Actions 页面的 "Re-run jobs" 功能

## 开发流程建议

1. 在 `pubspec.yaml` 中更新版本号
2. 提交代码变更
3. 创建并推送版本标签
4. 等待自动构建完成
5. 检查 GitHub Release 页面
