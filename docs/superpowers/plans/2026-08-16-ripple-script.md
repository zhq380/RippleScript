# 涟漪脚本（RippleScript）实施计划

> **For agentic workers:** 本计划按任务逐步执行。步骤用 checkbox（`- [ ]`）跟踪。
> 本机无 git，所有"提交"检查点以构建/测试验证代替。

**Goal:** 构建 Find X9 Pro 自动化脚本 Android 应用（录制回放 + 卡片/积木编辑 + JSON 导入导出，ColorOS 风格），产出可安装 APK。

**Architecture:** Kotlin + Jetpack Compose（Material 3 自定义 ColorOS 主题）单 Activity 多页面；AccessibilityService 负责手势分发与触摸录制（API 34+ setMotionEventSources）；ScriptEngine 为纯 Kotlin 协程解释器（可单元测试）；悬浮控制条用 WindowManager + 传统 View。

**Tech Stack:** Gradle 8.10.2 / AGP 8.7.3 / Kotlin 2.0.21 / compileSdk 35 (target 34, min 26) / Compose BOM 2024.12.01 / kotlinx-serialization / coroutines / DataStore。Maven 走 Aliyun 镜像，Gradle 走腾讯镜像。

---

## 文件结构

```
xm/
├── settings.gradle.kts / build.gradle.kts / gradle.properties / .gitignore
├── app/
│   ├── build.gradle.kts / proguard-rules.pro
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── res/ values(strings,colors,themes) values-night xml(accessibility) drawable ic_launcher + 控制条图标 mipmap-anydpi-v26
│       └── java/com/ripple/script/
│           ├── App.kt MainActivity.kt
│           ├── engine/ StepModel.kt StepTree.kt ScriptEngine.kt RunConfig.kt StepExecutor.kt GestureClassifier.kt
│           ├── data/ ScriptRepository.kt RunConfigStore.kt
│           ├── service/ AutoAccessibilityService.kt FloatingControllerService.kt Recorder.kt
│           ├── util/ Permissions.kt
│           └── ui/ theme/(Color,Theme) AppRoot.kt components/(StepCard,StepEditSheet) screens/(ScriptList,Create,RunConfig,Profile)
└── app/src/test/java/com/ripple/script/ StepModelTest StepTreeTest ScriptEngineTest GestureClassifierTest ScriptRepositoryTest
```

核心依赖接口（全项目只依赖这几个抽象）：
- `StepExecutor`（suspend click/longPress/swipe/input/launchApp/back）——引擎与 Android 解耦的关键
- `EngineHub`（paused/stopRequested + checkpoint()）——暂停/停止控制
- `StepTree`（insert/remove/move，按路径索引嵌套列表）——三编辑模式共享数据操作

---

### Task 0: 环境准备

- [ ] 下载 Gradle 8.10.2（腾讯镜像）解压到 `tools/`
- [ ] `sdkmanager` 安装 `platforms;android-35` `build-tools;35.0.0`
- [ ] 项目文件就绪后 `gradle.bat wrapper --gradle-version 8.10.2`，distributionUrl 改为 `https://mirrors.cloud.tencent.com/gradle/gradle-8.10.2-bin.zip`
- 验证：`.\gradlew.bat --version` 输出 8.10.2

### Task 1: 项目骨架

- [ ] 写入 gradle 三件套 + app/build.gradle.kts + Manifest + res（主题、图标、无障碍配置）
- [ ] MainActivity + Theme 占位
- 验证：`gradlew :app:assembleDebug` BUILD SUCCESSFUL（空壳 APK）

### Task 2: StepModel + JSON（TDD）

- [ ] 测试：嵌套 loop 往返序列化；`"type":"click"` 判别；未知字段容忍；非法 type 报错
- [ ] 实现：`@Serializable sealed interface Step` + 8 个 @SerialName 子类 + `Script(format,version,name,createdAt,steps)` + `ScriptJson`（encodeDefaults/ignoreUnknownKeys）
- 验证：`gradlew :app:testDebugUnitTest --tests "*StepModel*"` PASS

### Task 3: StepTree（TDD）

- [ ] 测试：根级/嵌套 insert；remove 任意深度；move ±1 越界钳制；at() 取值
- [ ] 实现：递归不可变列表操作（LoopStep.children 下降）
- 验证：`--tests "*StepTree*"` PASS

### Task 4: ScriptEngine（TDD）

- [ ] FakeExecutor 记录调用序列、可注入失败次数
- [ ] 测试：执行顺序；loop×config.loopTimes 笛卡尔；SKIP 策略失败后继续；STOP 策略失败即停且成功=false；手动停止；暂停门（虚拟时间 advanceTimeBy 证明阻塞后放行）；步间隔虚拟时间验证
- [ ] 实现：`run(script, config, hub)`；checkpoint 暂停门；单步失败重试 1 次后按策略；StopRunException → Result(false, executed, msg)
- 验证：`--tests "*ScriptEngine*"` PASS

### Task 5: GestureClassifier（TDD）

- [ ] 测试：短按→Click；≥500ms 静止→LongPress(dur)；移动>60px→Swipe(起止,dur)；多指忽略；250~500ms 静止→无输出；cancel 复位
- [ ] 实现：onDown/onMove/onUp(t)/onPointerAdded/cancel，纯 Kotlin 无 Android 依赖
- 验证：`--tests "*GestureClassifier*"` PASS

### Task 6: ScriptRepository（TDD）

- [ ] 测试（TemporaryFolder）：save/list 排序/delete；importJson 合法入库分配新 id；非法 JSON/空 steps 返回 failure
- [ ] 实现：目录 `files/scripts/$createdAt.json`；Entry(id, script?) 损坏标记 null
- 验证：`--tests "*ScriptRepository*"` PASS

### Task 7: 主题 + 导航骨架

- [ ] Color.kt（ColorOS 蓝 #2660F5 / 容器色 / 深色全套）Theme.kt（圆角 10-28dp，disable dynamicColor）
- [ ] AppRoot：底部 NavigationBar 三 Tab（脚本/制作/我的）+ NavHost（scripts/create/profile/run/{id}/edit/{id}）
- 验证：assembleDebug PASS

### Task 8: AutoAccessibilityService

- [ ] companion `@Volatile instance`；onServiceConnected 里 API≥34 `setMotionEventSources(SOURCE_TOUCHSCREEN)`（try-catch）
- [ ] `awaitGesture(path,duration)`：suspendCancellableCoroutine + GestureResultCallback，取消→StepFailure
- [ ] AccessibilityStepExecutor：click/longPress/swipe→手势；input→findFocus(FOCUS_INPUT_METHOD).ACTION_SET_TEXT；launchApp→getLaunchIntentForPackage(+NEW_TASK)；back→performGlobalAction
- [ ] onMotionEvent→Recorder 激活时分发给 GestureClassifier（忽略悬浮条矩形内事件、多指标记）
- 验证：编译通过；真机开权限后 dispatchGesture 生效

### Task 9: Recorder + FloatingControllerService

- [ ] Recorder：active/steps:MutableStateFlow/excludedRect
- [ ] 悬浮条：TYPE_APPLICATION_OVERLY 胶囊（半透明黑渐变圆角 28dp），可拖动；run 模式（暂停⇄继续、停止，轮询图标状态，结束时展示结果 1.5s）；record 模式（REC 计数 + 停止）；addView 后回写 excludedRect
- [ ] run 模式：解析 intent extras（script/config JSON）→ EngineHub + ScriptEngine(AccessibilityStepExecutor) 协程执行
- 验证：编译通过；真机悬浮条出现、按钮生效

### Task 10: ScriptListScreen

- [ ] 卡片列表（名称/步骤数/日期/损坏标记），点击→run/{id}，长按→底部菜单（编辑/导出/删除）
- [ ] 顶部导入（OpenDocument application/json → importJson → Snackbar 错误详情）；空态引导
- [ ] 导出：CreateDocument 建议名 `名称.json`
- 验证：手动 + assembleDebug

### Task 11: CreateScreen（制作页）

- [ ] 编辑器状态：steps/name/editingId；Recorder.active 横幅（N 步 + 完成录制并入）；开始录制（检查双权限→启动 record 模式悬浮服务）
- [ ] StepCard 列表：徽章色 + 标题 + 参数摘要；点击→StepEditSheet（类型化字段编辑）；上移/下移/删除/复制；LoopStep 卡片嵌套缩进渲染 + 循环内添加按钮
- [ ] 添加步骤底部面板：8 类型网格；保存对话框（新脚本命名 / 编辑覆盖）
- 验证：手动 + assembleDebug

### Task 12: RunConfigScreen + ProfileScreen

- [ ] 运行配置：循环次数 stepper、间隔 slider、失败策略单选；开始→权限校验（缺→跳设置对话框）→ startService(extras) → moveTaskToBack
- [ ] 我的：无障碍/悬浮窗权限状态卡（跳转）、运行参数默认值（DataStore 读写）、关于（版本/适配说明/后台保活提示）
- 验证：手动 + assembleDebug

### Task 13: 收尾

- [ ] `gradlew :app:testDebugUnitTest` 全绿
- [ ] `gradlew :app:assembleDebug` → `app/build/outputs/apk/debug/app-debug.apk`
- [ ] 输出安装/使用指引（开无障碍、开悬浮窗、录制→回放流程）

---

## 关键代码（已在实现中定稿的契约）

### 引擎循环（ScriptEngine）
```
run: repeat(config.loopTimes){ executeAll(steps) }
executeAll: for step → hub.checkpoint() → when(step){
  Loop → repeat(times){ executeAll(children) }
  Wait → delay(ms)
  else → executeSingle(重试1次→SKIP/STOP策略)
} → delay(config.stepIntervalMs)
checkpoint: stop→throw；while(paused&&!stop) delay(100)
```

### 手势判定阈值（GestureClassifier）
位移<60px 且 <250ms→Click；<60px 且 ≥500ms→LongPress；位移≥60px→Swipe(≥50ms)；250~500ms 静止→丢弃；多指→丢弃。

### 权限判定（Permissions）
`ENABLED_ACCESSIBILITY_SERVICES` 含本服务 ComponentName（flattenToString 比对）；`Settings.canDrawOverlays`。

### 录制排除区
悬浮条 addView 后 `getLocationOnScreen`+宽高→`Recorder.excludedRect`，onMotionEvent 落在矩形内直接忽略。

## Self-Review 结论
- 规格覆盖：设计文档 §5-§11 均有对应任务（导入导出→T10，三模式→T9/T11，运行配置→T4/T12，ColorOS→T7，权限→T8/T12，错误处理→T2/T4/T6）✓
- 无占位符；类型契约（Step/StepExecutor/EngineHub/StepTree/Recorder）跨任务一致 ✓
- 范围：单一交付物（APK）✓
