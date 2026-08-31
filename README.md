# Ripple Rewards Script · ColorOS 16 光场设计版

> 「微软必应积分」Android 无障碍自动化脚本，**ColorOS 16 光场设计**原生体验。

[![Release](https://img.shields.io/github/v/release/zhq380/RippleScript?include_prereleases&sort=date)](https://github.com/zhq380/RippleScript/releases)
[![Android](https://img.shields.io/badge/Android-8.0%2B-blue)](https://developer.android.com/)
[![ColorOS](https://img.shields.io/badge/ColorOS-16-orange)](https://www.coloros.com/)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.0-blue)](https://kotlinlang.org/)

## 📥 快速下载

👉 **[点击下载 ColorOS 版 APK](https://github.com/zhq380/RippleScript/releases/latest)**

## 🔀 版本对比

| 特性 | 🎨 ColorOS 版 (本仓库) | 📱 自适应版 |
|------|----------------------|------------|
| 屏幕适配 | 全宽无限制，ColorOS 原生全屏 | 600dp 限宽居中，平板/横屏友好 |
| 主题定位 | OPPO ColorOS 14+ 用户 | 通用 Android 8.0+ |
| 品牌元素 | ColorOS 16 光场标签、极光引擎 | 无品牌，纯响应式 |
| Repo | [zhq380/RippleScript](https://github.com/zhq380/RippleScript) | [zhq380/bing-rewards-assistant](https://github.com/zhq380/bing-rewards-assistant) |
| Release | [v1.10.11-coloros](https://github.com/zhq380/RippleScript/releases) | [v1.10.11](https://github.com/zhq380/bing-rewards-assistant/releases) |

## ✨ 功能特性

| 功能 | 说明 |
|------|------|
| 🎯 **每日自动签到** | 智能识别「金色硬币」位置，自动随日期后移点击 |
| 🔍 **智能搜索** | 内置 1000 词中英文词库，自动防重复 |
| 📰 **连续阅读** | 缺口预计算 + 连续阅读模式，减少来回跳转 |
| 📅 **每日活动** | OCR 兜底识别活动卡片，15s 计分等待 |
| 🧩 **双实例支持** | 主应用 / 分身独立配置、独立运行 |
| 🛡️ **看门狗自愈** | 超时自动恢复，弹窗不卡死 |
| 💾 **断点续跑** | 进程被杀后跳过已完成任务 |
| 🎨 **ColorOS 16 光场** | 温润通透极光引擎动效、潮汐引擎帧追踪 |

## 📱 环境要求

- **Android**: API 26+ (Android 8.0)
- **推荐**: ColorOS 14+ 以匹配光场设计
- **权限**: 无障碍服务、WRITE_SECURE_SETTINGS（可选）、通知、电池白名单

## 🔧 安装

1. 从 [Releases](https://github.com/zhq380/RippleScript/releases/latest) 下载 APK
2. 系统设置 → 无障碍 → 开启 Ripple Rewards Script
3. (可选) adb shell pm grant com.ripple.script android.permission.WRITE_SECURE_SETTINGS
4. 打开 APP → 配置脚本 → 点击运行

## 📄 许可证

[MIT License](LICENSE)