# 涟漪脚本（RippleScript）设计文档

日期：2026-08-16
目标设备：OPPO Find X9 Pro（PLG110，ColorOS 16 / Android 16）
状态：已与用户确认

## 1. 项目目标

一款运行在 OPPO Find X9 Pro 上的自动化脚本 Android 应用，支持：

- 通过无障碍服务对其他 APP 执行点击、滑动等自动化操作
- 导入 / 导出 JSON 格式脚本并执行
- 手机端可视化制作脚本：录制回放（主）、步骤卡片、逻辑积木三种模式，共享同一数据模型
- 界面遵循 ColorOS 设计规范，简约且功能齐全
- 在本机（Windows + JDK21 + Android SDK）直接构建出 APK

## 2. 目标设备参数（适配依据）

| 项目 | 参数 |
|---|---|
| 屏幕 | 6.78 英寸 AMOLED 柔性直屏，中置挖孔 |
| 分辨率 | 2772×1272（1.5K），450 PPI |
| 布局尺寸 | 约 424×924 dp（系统按 480dpi 计） |
| 刷新率 | 120Hz |
| 系统 | ColorOS 16（Android 16） |

适配策略：全部使用 dp + WindowInsets（含 displayCutout 中置挖孔安全区）自适应布局，不写死像素。

## 3. 技术栈（已确认方案 A）

- 语言：Kotlin
- UI：Jetpack Compose + Material 3，自定义 ColorOS 主题
- 自动化：AccessibilityService + GestureDescription.dispatchGesture
- 异步：Kotlin 协程 + Flow
- 存储：应用私有目录 JSON 文件 + DataStore（运行配置）
- 构建：Gradle Wrapper（AGP 8.x + Kotlin 2.x + Compose BOM），JDK 21，本机 Android SDK
- 最低 SDK：API 26（Android 8.0）；目标/编译 SDK：API 34（本机已装 platform 34；后续可经 sdkmanager 安装 android-36 后升级，Android 16 设备向下兼容）

## 4. 总体架构

```
app/src/main/java/com/ripple/script/
├── ui/                     Compose 界面层
│   ├── theme/              ColorOS 主题（#2660F5 主色、大圆角、水生动效）
│   ├── nav/                底部三 Tab 导航
│   ├── screens/
│   │   ├── ScriptListScreen    Tab1 脚本列表（新建/导入/搜索/长按管理）
│   │   ├── CreateScreen        Tab2 制作（录制 / 卡片 / 积木 三模式）
│   │   ├── RunConfigScreen     运行配置（循环次数、间隔、失败策略）
│   │   └── ProfileScreen       Tab3 我的（权限状态、导出、关于）
│   └── components/         通用组件（步骤卡片、循环块、悬浮控制条视图）
├── service/
│   ├── AutoAccessibilityService  无障碍服务：手势分发 + 触摸事件录制
│   └── FloatingControllerService 悬浮窗控制条：开始/暂停/停止/录制控制
├── engine/
│   ├── ScriptEngine          脚本解释器：JSON → 协程逐步执行，支持暂停/停止
│   ├── StepModel             步骤数据模型 + JSON 序列化/反序列化与校验
│   └── GestureDispatcher     步骤 → GestureDescription（含失败重试 1 次）
└── data/
    └── ScriptRepository      脚本 CRUD（私有目录 .ripple/scripts/*.json）
```

## 5. 脚本数据模型（导入导出格式）

```json
{
  "format": "ripple-script",
  "version": 1,
  "name": "签到脚本",
  "createdAt": 1760000000000,
  "steps": [
    {"type": "launchApp", "packageName": "com.target.app"},
    {"type": "wait", "ms": 1500},
    {"type": "click", "x": 1386, "y": 820},
    {"type": "longPress", "x": 500, "y": 900, "duration": 800},
    {"type": "swipe", "x1": 500, "y1": 1800, "x2": 500, "y2": 600, "duration": 300},
    {"type": "input", "text": "你好"},
    {"type": "back"},
    {"type": "loop", "times": 5, "children": []}
  ]
}
```

步骤类型 8 种：

| type | 参数 | 说明 |
|---|---|---|
| click | x, y | 单击坐标 |
| longPress | x, y, duration | 长按 |
| swipe | x1,y1,x2,y2,duration | 滑动 |
| input | text | 通过无障碍 ACTION_SET_TEXT 输入文本 |
| wait | ms | 固定等待 |
| launchApp | packageName | 启动指定应用 |
| back | — | 返回键（GLOBAL_ACTION_BACK） |
| loop | times, children | 循环包裹子步骤（可嵌套） |

坐标记录为物理像素（2772×1272 设备坐标系），导入其他分辨率设备的脚本时提供「按比例换算」选项。

## 6. 三种编辑模式（同一份数据）

1. 录制回放：开启录制 → 正常使用手机 → 无障碍服务捕获触摸事件（点击/长按/滑动）自动生成步骤；悬浮窗提供暂停/完成；录制结果进入卡片编辑器
2. 步骤卡片：卡片列表视图；点卡片展开参数编辑；长按拖动排序；增删复制单步
3. 逻辑积木：在卡片列表中插入循环块，被包裹步骤自动缩进嵌套显示；循环块可改次数、可删除（子步骤保留）

三模式随时切换，编辑同一 JSON 模型。

## 7. 运行时行为

- 运行入口：脚本列表点击 → 运行配置页（循环次数、步间间隔、失败策略）→ 开始 → 跳转桌面/目标应用，悬浮窗控制条出现
- 悬浮窗：半透明胶囊形（ColorOS 风格），含 暂停/继续、停止 按钮；录制模式显示红色圆点 + 计时
- 执行引擎：协程逐 step 执行；支持暂停（挂起）/停止（取消）；步间默认间隔可配
- 失败策略：单步手势失败重试 1 次 → 仍失败按策略「停止脚本」或「跳过继续」
- 无障碍服务掉线：悬浮窗闪烁提示并暂停执行

## 8. ColorOS 设计规范落地

- 主色 #2660F5（ColorOS 蓝），辅助绿 #046A38；背景浅灰 #F2F3F5，卡片白色，圆角 16-20dp
- 水生设计（Aquatic Design）：波纹按压反馈、列表项弹性动效、页面转场共享元素
- 底部三 Tab（脚本/制作/我的）+ 顶部大标题；支持深色模式
- 适配挖孔屏：statusBar via WindowInsets；120Hz 下动效使用 spring 动画

## 9. 权限与安全

| 权限 | 用途 | 触发时机 |
|---|---|---|
| BIND_ACCESSIBILITY_SERVICE | 手势分发与录制 | 首次使用引导开启 |
| SYSTEM_ALERT_WINDOW | 悬浮控制条 | 运行/录制前检查 |
| SAF 文件选择器 | 脚本导入导出 | 按需，无需存储权限 |

不申请网络、通讯录等任何无关权限；脚本明文 JSON，本地存储，不联网。

## 10. 错误处理

- 无障碍未开启：首页顶部横幅引导跳转系统设置页
- 导入 JSON：格式/字段校验，失败给出具体行级错误提示
- 目标应用不存在（launchApp）：报错并按失败策略处理
- 脚本文件损坏：列表中标记异常，不崩溃

## 11. 测试计划

- 单元测试：StepModel JSON 序列化/校验；ScriptEngine 顺序/循环/暂停/停止逻辑（用假手势执行器）
- 构建验证：`gradlew assembleDebug` 产出 APK
- 真机验收（用户侧）：安装 APK → 开启无障碍 → 录制一段操作 → 回放验证 → 导出再导入验证

## 12. 明确不做（YAGNI）

- 不做 iOS 版（系统不允许）
- 不做云同步/账号体系
- 不做图像识别/找图点击（后续版本可扩展）
- 不做脚本加密与付费机制
