# iOS 构建故障排除指南

## 问题描述

在 GitHub Actions 中构建 iOS 应用时遇到以下错误：

```
Unable to find a destination matching the provided destination specifier:
{ generic:1, platform:iOS }

Ineligible destinations for the "Runner" scheme:
{ platform:iOS, id:dvtdevice-DVTiPhonePlaceholder-iphoneos:placeholder, name:Any iOS Device, error:iOS 18.0 is not installed. }
```

## 根本原因

1. **平台版本不匹配**: GitHub Actions 的 macOS runner 缺少项目要求的 iOS 18.0 平台
2. **部署目标过高**: 项目的 iOS 部署目标设置可能高于环境中可用的版本
3. **Xcode 版本限制**: runner 中的 Xcode 版本可能不支持最新的 iOS 版本

## 解决方案

### 方案 1: 降低部署目标（推荐）

通过修改项目配置，将 iOS 部署目标降低到 GitHub Actions 环境支持的版本：

```bash
# 修改 Xcode 项目配置
sed -i '' 's/IPHONEOS_DEPLOYMENT_TARGET = [0-9]*\.[0-9]*/IPHONEOS_DEPLOYMENT_TARGET = 12.0/g' ios/Runner.xcodeproj/project.pbxproj

# 修改 Flutter 配置
echo "IPHONEOS_DEPLOYMENT_TARGET = 12.0" >> ios/Flutter/Release.xcconfig

# 重新安装依赖
flutter clean && flutter pub get
cd ios && pod install && cd ..
```

### 方案 2: 使用特定的 macOS runner 版本

在 workflow 中指定使用特定版本的 macOS runner：

```yaml
runs-on: macos-13  # 或 macos-12，而不是 macos-latest
```

### 方案 3: 检查并使用可用的 SDK

动态检查可用的 iOS SDK 版本：

```bash
# 列出可用的 SDK
xcodebuild -showsdks | grep iphoneos

# 获取最新可用版本
SDK_VERSION=$(xcodebuild -showsdks | grep iphoneos | tail -1 | sed 's/.*iphoneos//' | xargs)
```

## 当前实施的解决方案

### 🔄 多层次修复策略

我们采用了**渐进式修复方案**，包含多个层次的配置修改：

#### 第一层：全面配置修改
1. **Xcode 项目配置**: 修改 `project.pbxproj` 中的 `IPHONEOS_DEPLOYMENT_TARGET`
2. **Flutter 配置**: 更新 `ios/Flutter/Release.xcconfig`
3. **CocoaPods 配置**: 修改 `ios/Podfile` 中的平台版本
4. **环境变量**: 设置构建时的环境变量

#### 第二层：依赖重建
1. **完全清理**: 使用 `flutter clean` 清理所有缓存
2. **CocoaPods 重置**: 使用 `pod deintegrate` 和 `pod install --repo-update`
3. **依赖重新获取**: 重新下载所有依赖

#### 第三层：备用构建策略
1. **主要方案**: 使用 iOS 12.0 作为部署目标
2. **备用方案**: 如果失败，自动降级到 iOS 11.0
3. **错误恢复**: 提供详细的错误诊断和日志

### 📊 不稳定性分析

根据日志分析，iOS 构建不稳定的主要原因：

1. **配置修改时机问题**:
   - 之前的修改可能在 CocoaPods 安装之后被覆盖
   - Flutter 构建过程中会重新生成某些配置

2. **CocoaPods 缓存干扰**:
   - 旧的 Pod 配置可能包含高版本的部署目标
   - 需要完全重新安装依赖

3. **多配置文件冲突**:
   - Xcode 项目、Flutter 配置、CocoaPods 配置需要保持一致
   - 任何一个配置不匹配都可能导致构建失败

## 验证步骤

构建成功后，可以通过以下方式验证：

1. **检查构建产物**: 确认 `build/ios/iphoneos/Runner.app` 存在
2. **检查 IPA 文件**: 确认 `ProxyPin-iOS.ipa` 创建成功
3. **验证兼容性**: 生成的应用应该兼容 iOS 12.0 及以上版本

## 预防措施

为了避免类似问题，建议：

1. **固定 runner 版本**: 不使用 `macos-latest`，而是指定具体版本
2. **保守的部署目标**: 使用较低的 iOS 部署目标以确保兼容性
3. **定期更新**: 定期检查 GitHub Actions 环境的更新
4. **本地测试**: 在类似环境中进行本地测试

## 相关链接

- [GitHub Actions macOS runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)
- [Flutter iOS 构建配置](https://docs.flutter.dev/deployment/ios)
- [Xcode 版本兼容性](https://developer.apple.com/support/xcode/)
