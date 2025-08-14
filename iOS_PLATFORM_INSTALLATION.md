# iOS 平台手动安装指南

## 概述

当在 GitHub Actions 中遇到 "iOS 18.0 is not installed" 错误时，可以尝试手动安装 iOS 18.0 平台。本指南提供了多种安装方法和解决方案。

## 🛠️ 安装方法

### 方法 1: 使用 xcodebuild 命令行工具

```bash
# 下载并安装 iOS 平台
sudo xcodebuild -downloadPlatform iOS

# 或者指定特定版本
sudo xcodebuild -downloadPlatform iOS -version 18.0
```

**优点**：
- 官方推荐方法
- 直接安装到 Xcode 中

**缺点**：
- 可能需要较长时间
- 在 CI 环境中成功率不确定

### 方法 2: 使用 xcrun simctl

```bash
# 列出可用的运行时
xcrun simctl list runtimes available

# 安装特定的 iOS 运行时
xcrun simctl runtime install "iOS 18.0"
```

**优点**：
- 专门用于模拟器运行时
- 命令简洁

**缺点**：
- 主要针对模拟器
- 对设备构建帮助有限

### 方法 3: 使用 mas-cli (Mac App Store CLI)

```bash
# 安装 mas-cli
brew install mas

# 更新 Xcode (可能包含新的平台)
mas upgrade
```

**优点**：
- 可以更新整个 Xcode
- 包含所有最新平台

**缺点**：
- 需要 Apple ID 登录
- 在 CI 环境中难以实现

## 🚧 GitHub Actions 中的限制

### 环境限制
1. **网络限制**: GitHub Actions runner 的网络可能限制大文件下载
2. **时间限制**: 平台安装可能需要很长时间，超过 job 时间限制
3. **权限限制**: 某些操作可能需要管理员权限
4. **存储限制**: iOS 平台文件较大，可能超过存储限制

### 实际可行性
- ✅ **理论上可行**: 命令存在且有效
- ⚠️ **实际成功率低**: 由于上述限制，成功率不高
- 🔄 **替代方案更可靠**: 降级部署目标通常更稳定

## 🎯 推荐的解决策略

### 策略 1: 智能安装 + 降级备用

我们在 workflow 中实施的策略：

```yaml
- name: Install iOS 18.0 platform (if needed)
  run: |
    if ! xcodebuild -showsdks | grep -q "iphoneos18.0"; then
      echo "尝试安装 iOS 18.0..."
      sudo xcodebuild -downloadPlatform iOS || echo "安装失败"
    fi
    
    # 检查安装结果，失败则使用降级方案
    if ! xcodebuild -showsdks | grep -q "iphoneos18.0"; then
      echo "使用降级方案: iOS 12.0"
    fi
```

### 策略 2: 使用特定的 macOS Runner 版本

```yaml
runs-on: macos-13  # 而不是 macos-latest
```

不同版本的 macOS runner 包含不同版本的 Xcode：
- `macos-13`: Xcode 14.x
- `macos-12`: Xcode 13.x  
- `macos-latest`: 最新版本（可能不稳定）

### 策略 3: 预安装检查和条件构建

```yaml
- name: Check iOS platform availability
  id: ios_check
  run: |
    if xcodebuild -showsdks | grep -q "iphoneos18.0"; then
      echo "ios18_available=true" >> $GITHUB_OUTPUT
    else
      echo "ios18_available=false" >> $GITHUB_OUTPUT
    fi

- name: Build iOS (iOS 18.0)
  if: steps.ios_check.outputs.ios18_available == 'true'
  run: flutter build ios --release --no-codesign

- name: Build iOS (Fallback)
  if: steps.ios_check.outputs.ios18_available == 'false'
  run: |
    # 使用降级配置构建
    # ...
```

## 📊 成功率对比

| 方法 | 成功率 | 时间成本 | 维护成本 |
|------|--------|----------|----------|
| 手动安装 iOS 18.0 | 低 (~20%) | 高 (5-15分钟) | 高 |
| 降级到 iOS 12.0 | 高 (~95%) | 低 (1-2分钟) | 低 |
| 使用固定 runner 版本 | 中 (~70%) | 中 (2-5分钟) | 中 |

## 🔧 实际实施建议

### 对于生产环境
**推荐使用降级方案**：
- 稳定可靠
- 构建时间短
- 维护成本低
- 兼容性好

### 对于实验环境
可以尝试手动安装：
- 测试最新功能
- 验证兼容性
- 探索新特性

## 🚀 最佳实践

1. **优先级策略**: 先尝试安装，失败则降级
2. **超时控制**: 为安装步骤设置合理的超时时间
3. **缓存利用**: 如果可能，缓存已安装的平台
4. **监控告警**: 监控安装成功率，及时调整策略

## 📝 总结

虽然在 GitHub Actions 中手动安装 iOS 18.0 在技术上是可行的，但由于各种限制，成功率较低。**推荐的最佳实践是使用智能安装 + 降级备用的策略**，这样既尝试了最新平台，又保证了构建的稳定性。
