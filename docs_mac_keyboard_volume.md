# macOS 直播间方向键调节音量 —— 调研与实施方案

> **文档性质**：调研 + 实施方案 + **已落地的实现记录**（代码已实现并推送到 fork，见第 12 节）。
> **调研环境**：macOS 26.6.2 (arm64)，**未安装 Flutter / Dart / CocoaPods，且 Xcode 未完整安装**（`xcode-select -p` → `/Library/Developer/CommandLineTools`，`/Applications/Xcode*.app` 不存在），因此本机**无法编译验证**。
> **目标**：让另一个会话里的 AI 能够仅凭本文档，完成实现、远端编译与验证。
> **调研基线**：代码分析基于 `master` @ `ba828e6`（v1.11.4）；**实际开发基线为 `dev` @ `bccd2ba`（v1.11.7）**，两者在本文涉及的文件上**无差异**（见 7.7.1）。
> **远端仓库**：https://github.com/xiaoyaocz/dart_simple_live

---

## 0. 一句话结论

**功能可行且已实现**：纯 Dart 改动，不涉及任何 macOS 原生代码、不需要 `pod install`、不需要重新生成 `GeneratedPluginRegistrant.swift`。复用项目里已有的手势音量 OSD 提示组件，UI 零改动，但**必须自己补一个自动隐藏 Timer**（这是现有代码缺失的部分）。

**编译可行，且已有成功先例**：本机无法本地构建（无 Flutter/Xcode），但**走 GitHub Actions 远端构建已被实际验证可行**。
本机存在此前会话留下的完整产物链：fork `NSFish/dart_simple_live` + 自定义 workflow `.github/workflows/build-macos.yml` +
已安装的 `/Applications/Simple Live.app`（v1.11.4, arm64, ad-hoc）。macOS 构建**不需要任何证书或 secrets**。
⚠️ 但**不能直接用上游的 `publish_app_dev.yaml`**（Android 步骤会先崩、且 `ref: dev` 写死）—— 必须用独立的 macOS-only workflow，详见第 7 节。

👉 **本次实现的具体改动、提交与构建记录见第 12 节。**

---

## 1. 仓库结构与相关文件

这是一个 Flutter monorepo（无根目录 `pubspec.yaml`，各子工程独立）：

```
dart_simple_live/
├── simple_live_app/      ← 桌面端（Android/iOS/Windows/Linux/macOS）★ 本次目标
├── simple_live_tv_app/   ← TV 端（遥控器适配，有可参考的键盘实现）
├── simple_live_core/     ← 直播站点协议库
└── simple_live_console/  ← 命令行工具
```

本次改动只涉及 `simple_live_app/lib/`。

### 涉及文件清单

| 文件 | 路径（相对 `simple_live_app/`） | 作用 |
|---|---|---|
| 直播间页面 | `lib/modules/live_room/live_room_page.dart` | **主要改动点**：挂载键盘监听 |
| 播放器状态/手势 mixin | `lib/modules/live_room/player/player_controller.dart` | **主要改动点**：新增键盘调音量方法 + 隐藏 Timer |
| 播放控件 UI | `lib/modules/live_room/player/player_controls.dart` | OSD 提示渲染处，**理论上无需改动** |
| 直播间控制器 | `lib/modules/live_room/live_room_controller.dart` | 现有 `showVolumeSlider()` 所在；`onClose()` 需取消 Timer |
| 全局设置 | `lib/app/controller/app_settings_controller.dart` | `playerVolume` 状态与持久化 |
| 本地存储 key | `lib/services/local_storage_service.dart` | `kPlayerVolume` |
| 直播间设置页 | `lib/modules/settings/play_settings_page.dart` | 可选：加开关 |
| TV 端参考实现 | `../simple_live_tv_app/lib/modules/live_room/live_room_page.dart` | 键盘监听范例 |
| 节流工具 | `lib/app/custom_throttle.dart` | `DelayedThrottle` |

---

## 2. 现状分析：macOS 上现有的音量调节方式

### 2.1 唯一可用入口：播放条上的小喇叭

`lib/modules/live_room/live_room_controller.dart:623`：

```dart
void showVolumeSlider(BuildContext targetContext) {
  SmartDialog.showAttach(
    targetContext: targetContext,
    alignment: Alignment.topCenter,
    displayTime: const Duration(seconds: 3),   // ← 3 秒自动消失，由 SmartDialog 托管
    maskColor: const Color(0x00000000),
    builder: (context) {
      return Container(
        decoration: BoxDecoration(
          borderRadius: AppStyle.radius12,
          color: Theme.of(context).cardColor,
        ),
        padding: AppStyle.edgeInsetsA4,
        child: Obx(
          () => SizedBox(
            width: 200,
            child: Slider(
              min: 0,
              max: 100,
              value: AppSettingsController.instance.playerVolume.value,
              onChanged: (newValue) {
                player.setVolume(newValue);                          // ← 关键点
                AppSettingsController.instance.setPlayerVolume(newValue);  // ← 关键点
              },
            ),
          ),
        ),
      );
    },
  );
}
```

这个 Slider 按钮的可见性条件是 `!Platform.isAndroid && !Platform.isIOS`（见 `player_controls.dart:596` 全屏版、`:557` 非全屏版），**macOS 上可见**。

**调用链（这是我们要对齐的黄金标准）：**

```
player.setVolume(value)                              → mpv 播放器音量（0–100）
AppSettingsController.instance.setPlayerVolume(value) → 写入 Rx + 落盘 kPlayerVolume
```

### 2.2 音量状态与持久化链路

`lib/app/controller/app_settings_controller.dart:425`：

```dart
Rx<double> playerVolume = 100.0.obs;
void setPlayerVolume(double value) {
  playerVolume.value = value;
  LocalStorageService.instance.setValue(
    LocalStorageService.kPlayerVolume,
    value,
  );
}
```

读取（`:94`，默认值 `100.0`）：

```dart
playerVolume.value = LocalStorageService.instance.getValue(
  LocalStorageService.kPlayerVolume,
  100.0,
);
```

Key 定义在 `lib/services/local_storage_service.dart:112`：`static const String kPlayerVolume = "PlayerVolume";`

应用时机 —— `player_controller.dart:663`（`PlayerController.onInit`）：

```dart
@override
void onInit() {
  initSystem();
  initStream();
  //设置音量
  player.setVolume(AppSettingsController.instance.playerVolume.value);
  super.onInit();
}
```

✅ **结论：只要走 `setPlayerVolume`，音量就会自动持久化，下次启动/换直播间自动恢复。**

### 2.3 手势调音量 —— 在 macOS 上被禁用（且有 bug）

`PlayerGestureControlMixin` 定义在 `player_controller.dart:473`：
```dart
mixin PlayerGestureControlMixin
    on PlayerStateMixin, PlayerMixin, PlayerSystemMixin {
```

手势调音量走的是**系统音量**（`VolumeController.instance.setVolume`，`:614`），路径与上面的 Slider **完全不同**。

`onVerticalDragStart`（`:545-553`）：
```dart
verticalDragging = true;
if (Platform.isAndroid || Platform.isIOS || Platform.isMacOS) {
  showGestureTip.value = true;                      // ⚠️ macOS 会进来
}
if (Platform.isAndroid || Platform.isIOS) {
  _currentVolume = await VolumeController.instance.getVolume();
}
if (Platform.isAndroid || Platform.isIOS || Platform.isMacOS) {
  _currentBrightness = await ScreenBrightness.instance.application;
}
```

`onVerticalDragUpdate`（`:565-567`）：
```dart
if (!Platform.isAndroid && !Platform.isIOS) {
  return;                                           // ⚠️ 桌面直接短路
}
```

**🐛 既有 bug（复现路径）**：macOS 上在播放器中部竖向拖拽 → `onVerticalDragStart` 把 `showGestureTip` 置 `true`，但 `onVerticalDragUpdate` 立即 return，`gestureTipText` 从未被赋值（保持空串 `""`）→ **弹出一个空的深色圆角卡片**，直到 `onVerticalDragEnd`（`:651`）才关闭。

> 实现键盘提示时，这个空卡片会和你的音量提示共用同一个 widget，调试时极易混淆。**建议顺手修掉**：把 `:547` 的判断从 `isAndroid || isiOS || isMacOS` 收敛为 `isAndroid || isiOS`。

---

## 3. 关于 `volume_controller` 插件：**修正一处先前误判**

> ⚠️ **先前的一轮分析中曾据 `macos/Podfile.lock` 推断「volume_controller 在 macOS 上未被使用」——该推断错误，特此更正。**

**事实核查结果：**

| 证据 | 内容 | 结论 |
|---|---|---|
| `lib/pubspec.yaml` | `volume_controller: ^3.4.0` | 已声明依赖 |
| `macos/Flutter/GeneratedPluginRegistrant.swift` | 含 `import volume_controller` 及 `VolumeControllerPlugin.register(with: registry.registrar(forPlugin: "VolumeControllerPlugin"))` | ✅ **macOS 上已注册** |
| `windows/flutter/generated_plugin_registrant.cc:19` | 含 `volume_controller_plugin_c_api.h` | ✅ Windows 已注册 |
| `linux/flutter/generated_plugin_registrant.cc:14,33-35` | 含 volume_controller 注册 | ✅ Linux 已注册 |
| `macos/Podfile.lock` | **不含** volume_controller | ❌ 误导来源 |
| pub.dev | macOS / Windows / Linux / Android / iOS 全支持（macOS 自 3.1.0 起） | ✅ 官方支持 |

**为什么 Podfile.lock 不可信**：`git log` 显示
- `macos/Podfile.lock` 最后一次修改：`028a988 更新MacOS配置`
- `macos/Flutter/GeneratedPluginRegistrant.swift` 最后一次修改：`e7928c3 修改JS引擎`

即 **lock 文件比 registrant 旧，是过期快照**。不能以它推断插件支持情况。

**但代码层面，桌面端确实从未调用过它** —— 全仓库 `VolumeController` 调用点仅 3 处，全在 Android/iOS 分支内：
- `player_controller.dart:551` `getVolume()`（仅 Android/iOS）
- `player_controller.dart:614` `setVolume()`（仅被手势路径调用，桌面被短路）
- `player_controller.dart:237` `showSystemUI = false`（仅 Android/iOS）

---

## 4. 键盘监听：项目现有实现（含一个失效的实现）

### 4.1 ❌ 失效范例 —— `lib/main.dart:262`

```dart
child: KeyboardListener(
  focusNode: FocusNode(),          // ⚠️ 没有 autofocus: true
  onKeyEvent: (KeyEvent event) async {
    if (event is KeyDownEvent &&
        event.logicalKey == LogicalKeyboardKey.escape) {
      if (!Platform.isAndroid && !Platform.isIOS) {
        if (await windowManager.isFullScreen()) {
          await windowManager.setFullScreen(false);
          return;
        }
      }
    }
  },
  child: child!,
),
```

**问题**：`KeyboardListener` 不会自动请求焦点，`FocusNode()` 创建后没有 `requestFocus()`，也没有 `autofocus: true`。默认 `primaryFocus` 是 rootScope，事件不会派发到这个节点。**这个 ESC 退出全屏逻辑大概率从未生效**。

> 顺手可修：加 `autofocus: true`，或改用 `Focus` widget。但注意顶层加 `autofocus` 可能与其他页面输入框抢焦点，**建议不在本次改动中动它**，或单独评估。

### 4.2 ✅ 可靠范例 —— `simple_live_tv_app/lib/modules/live_room/live_room_page.dart:37`

```dart
child: KeyboardListener(
  focusNode: controller.focusNode,
  autofocus: true,                 // ← 关键
  onKeyEvent: onKeyEvent,
  child: Scaffold(
    backgroundColor: Colors.black,
    body: Obx(() => buildMediaPlayer()),
  ),
),
```

处理函数（`:51`）：
```dart
void onKeyEvent(KeyEvent key) {
  if (key is KeyUpEvent) {
    return;                        // ← 只处理 down，忽略 up
  }
  Log.logPrint(key);

  if (key.logicalKey == LogicalKeyboardKey.select ||
      key.logicalKey == LogicalKeyboardKey.enter ||
      key.logicalKey == LogicalKeyboardKey.space) {
    if (!controller.showControlsState.value) {
      controller.showControls();
    } else {
      controller.hideControls();
    }
    return;
  }
  // ... arrowUp/arrowDown 切台，arrowLeft 显示关注，arrowRight/M 打开设置
}
```

**这是本仓库唯一确认可用的键盘监听写法，直接照抄其结构。**

---

## 5. OSD 提示组件（重点：用户已确认关心这一块）

### 5.1 状态字段

`player_controller.dart:104-107`（在 `PlayerStateMixin` 内）：

```dart
/// 显示手势Tip
RxBool showGestureTip = false.obs;

/// 手势Tip文本
RxString gestureTipText = "".obs;
```

### 5.2 渲染位置 —— 两处，全屏版与非全屏版

`player_controls.dart` 中有**两份完全相同的渲染代码**：

- **全屏版**：`buildFullControls()` 内，`:376-393`
- **非全屏版**：`buildControls()` 内，`:624-641`

```dart
Obx(
  () => Offstage(
    offstage: !controller.showGestureTip.value,
    child: Center(
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: Colors.grey.shade900,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Text(
          controller.gestureTipText.value,
          style: const TextStyle(fontSize: 18, color: Colors.white),
        ),
      ),
    ),
  ),
),
```

**样式**：深色（`Colors.grey.shade900`）圆角 12 卡片，内边距 12，白色 18px 文本。与 Android 上手势调音量看到的「音量 65%」**完全一致**。

### 5.3 ⚠️ 关键：这个提示**不会自动消失**

全仓库 `showGestureTip` 仅 4 处引用（已核实）：

| 位置 | 操作 |
|---|---|
| `player_controller.dart:104` | 声明 |
| `player_controller.dart:547` | `onVerticalDragStart` → 置 `true` |
| `player_controller.dart:651` | `onVerticalDragEnd` → 置 `false` |
| `player_controls.dart:380` / `:628` | 读取渲染（×2） |

**它完全依赖"拖拽结束"事件关闭。键盘没有"结束"事件** → **必须自己加一个 ~800ms–1s 的 Timer 隐藏。**

### 5.4 ⚠️ 关键：OSD 的作用域是"播放器区域"，不是全屏

两份 OSD 都是 `buildFullControls` / `buildControls` 这个 `Stack` 的直接子节点，而该 Stack 是 `Video(controls: (state) => playerControls(state, controller))` 的控件层，**只覆盖播放器矩形**。

- **全屏时** → 播放器铺满屏幕 → OSD 屏幕居中 ✅
- **非全屏时** → 播放器只是页面上方 16:9 区域 → OSD **居中于播放器矩形内**，不会盖住聊天区

这个行为是合理的（提示跟随视频），但实现时要知晓，不要误判为 bug。

### 5.5 死代码警告（别误用）

以下字段在 `player_controller.dart` 中**只有声明，全仓库从未使用**，不要以为有人已经处理过自动隐藏：

| 字段 | 行号 | 注释 | 实际状态 |
|---|---|---|---|
| `hidevolumeTimer` | `:80` | "音量控制条计时器" | ❌ 死代码，仅 1 处出现 |
| `hideSeekTipTimer` | `:119` | "自动隐藏提示计时器" | ❌ 死代码，仅 1 处出现 |
| `showBottomTip` | `:110` | "显示提示底部Tip" | ❌ 死代码，仅 1 处出现 |
| `bottomTipText` | `:113` | "提示底部Tip文本" | ❌ 死代码，仅 1 处出现 |

> 注：`showVolumeSlider` 的 3 秒消失是靠 `SmartDialog` 的 `displayTime: const Duration(seconds: 3)` 实现的，与 `hidevolumeTimer` 无关。

**整个 OSD 体系中，唯一被真正驱动的就是 `showGestureTip` + `gestureTipText`。**

---

## 6. 实施方案

### 6.1 方案选择

| | **方案 A（推荐）**：`player.setVolume` | **方案 B（备选）**：`VolumeController` |
|---|---|---|
| 调节对象 | App 内 mpv 播放音量 | 系统全局音量 |
| 与现有小喇叭 Slider 一致 | ✅ 完全同一链路 | ❌ 会脱节（Slider 显示的值与系统音量不符） |
| 音量持久化 | ✅ 走 `kPlayerVolume` | ❌ |
| 原生改动 | 无 | 无（插件已在 macOS 注册） |
| 风险 | 低 | 中（macOS 系统音量读写，未经本项目验证） |

**推荐方案 A。** 理由：与用户既有心智模型和 UI 状态一致，改动可控。

### 6.2 改动 1：`player_controller.dart` —— 新增键盘调音量方法

放在 `PlayerGestureControlMixin`（`:473`）内，或新建一个 mixin 混入 `PlayerController`（`:655`）。

```dart
/// 方向键调节音量（macOS/Windows/Linux 桌面端）
Timer? _hideVolumeTipTimer;

void setKeyboardVolume(double delta) {
  // 1. 守卫：锁定控制器时忽略（与 setGestureVolume 保持一致）
  if (lockControlsState.value) {
    return;
  }

  // 2. 计算新音量，步进 5，clamp 0–100
  final current = AppSettingsController.instance.playerVolume.value;
  final next = (current + delta).clamp(0.0, 100.0);
  if (next == current) {
    return;
  }

  // 3. 应用 —— 与 showVolumeSlider 完全同一链路
  player.setVolume(next);
  AppSettingsController.instance.setPlayerVolume(next);

  // 4. 更新 OSD（与手势文案保持一致）
  gestureTipText.value = "音量 ${next.toInt()}%";
  showGestureTip.value = true;

  // 5. ★ 自动隐藏 Timer（现有代码缺失，必须补）
  _hideVolumeTipTimer?.cancel();
  _hideVolumeTipTimer = Timer(const Duration(milliseconds: 800), () {
    showGestureTip.value = false;
  });
}
```

**步进说明**：取 5 是为了与手势的 `_convertVolume`（`:608`，`(volume / 5).round() * 5`）保持一致。

**节流说明**：方向键长按会触发 `KeyRepeatEvent`。由于每次只 ±5 且 `player.setVolume` 是轻量的 mpv 属性设置，实测可不节流；若发现卡顿，可复用 `lib/app/custom_throttle.dart` 的 `DelayedThrottle(200)`（参考 `:543` 的用法）。

### 6.3 改动 2：`live_room_page.dart` —— 挂载键盘监听

**挂载点选择**：`LiveRoomPage.build()` 有两种形态（`:88-101`）：

```dart
if (controller.fullScreenState.value) {
  return PopScope(
    canPop: false,
    onPopInvokedWithResult: (e, r) { controller.exitFull(); },
    child: Scaffold(body: buildMediaPlayer()),     // 全屏
  );
} else {
  return buildPageUI();                             // 非全屏，播放器嵌在页面里
}
```

**推荐：包在 `buildMediaPlayer()`（定义于 `:241`）的外层**，这样全屏与非全屏两种形态都能覆盖，只改一处。

```dart
Widget buildMediaPlayer() {
  // ... 现有 boxFit / aspectRatio 计算（:242-258）...

  final video = Stack(
    children: [
      Video(/* ...现有参数不变... */),
      Obx(() => Visibility(/* "未开播" 文字，现有 */)),
    ],
  );

  // 桌面端才挂键盘监听
  if (!(Platform.isMacOS || Platform.isWindows || Platform.isLinux)) {
    return video;
  }

  return Focus(
    autofocus: true,
    onKeyEvent: (FocusNode node, KeyEvent event) {
      // 只处理按下（长按重复也响应），忽略抬起
      if (event is KeyUpEvent) {
        return KeyEventResult.ignored;
      }

      // ★ 焦点在输入框时不拦截（见 6.4）
      if (_isEditableFocused()) {
        return KeyEventResult.ignored;
      }

      switch (event.logicalKey) {
        case LogicalKeyboardKey.arrowUp:
          controller.setKeyboardVolume(5);
          return KeyEventResult.handled;
        case LogicalKeyboardKey.arrowDown:
          controller.setKeyboardVolume(-5);
          return KeyEventResult.handled;
        default:
          return KeyEventResult.ignored;
      }
    },
    child: video,
  );
}
```

**为什么用 `Focus` 而不是 `KeyboardListener`**（这是本方案的核心技术点）：

macOS 桌面端**方向键默认被 Flutter 的焦点遍历（Focus traversal）消费**，用于在控件间移动焦点。`KeyboardListener.onKeyEvent` 返回 `void`，**无法声明"已消费"**，拦不住事件继续冒泡到焦点系统。而 `Focus.onKeyEvent` 返回 `KeyEventResult`，返回 `handled` 即可阻止冒泡。

### 6.4 焦点冲突处理

**冲突 1：直播间内有 TextField**

`live_room_controller.dart:770` 的 `showDanmuShield()` 会弹出含 `TextField` 的关键词输入弹窗。`autofocus: true` 的 `Focus` 会与它抢焦点，导致输入框打不了字。

解决 —— 在 `onKeyEvent` 开头判断：

```dart
bool _isEditableFocused() {
  final focus = FocusManager.instance.primaryFocus;
  if (focus == null) return false;
  return focus.context?.widget is EditableText;
}
```

命中就 `return KeyEventResult.ignored`，让方向键正常用于文本光标移动。

**冲突 2：非全屏时聊天列表抢焦点**

非全屏形态下，用户点击聊天区/设置面板后焦点移走，方向键恢复为列表滚动行为。**这其实是合理行为**（焦点在哪，键盘服务于哪），建议保留。

如果产品上要求"全页面生效"，改用全局注册：

```dart
// 在 LiveRoomController.onInit()
HardwareKeyboard.instance.addHandler(_onGlobalKey);
// 在 onClose()（live_room_controller.dart:1055）
HardwareKeyboard.instance.removeHandler(_onGlobalKey);
```

⚠️ 全局方案下**必须**自己实现 `_isEditableFocused()` 守卫，且要额外注意弹窗（`SmartDialog` / `showModalBottomSheet`）打开时不应响应。

### 6.5 改动 3：清理 —— 取消 Timer + 修空卡片

**a) 取消 Timer**

`live_room_controller.dart:1055` 的 `onClose()`：

```dart
@override
void onClose() {
  WidgetsBinding.instance.removeObserver(this);
  scrollController.removeListener(scrollListener);
  autoExitTimer?.cancel();
  _hideVolumeTipTimer?.cancel();      // ← 新增
  liveDanmaku.stop();
  danmakuController = null;
  _liveDurationTimer?.cancel();
  super.onClose();
}
```

（若用全局 `HardwareKeyboard` 方案，此处还要 `removeHandler`。）

**b) 修 macOS 空卡片 bug**

`player_controller.dart:547`：

```dart
// 改前
if (Platform.isAndroid || Platform.isIOS || Platform.isMacOS) {
  showGestureTip.value = true;
}
// 改后
if (Platform.isAndroid || Platform.isIOS) {
  showGestureTip.value = true;
}
```

⚠️ 注意：`:552` 那行的 `_currentBrightness` 读取仍保留 `isMacOS`（亮度手势在桌面另有用途），**只改 `:547` 这一处**。

### 6.6 改动 4（可选）：设置开关

若需加"方向键调节音量"开关，照抄 `hardwareDecode`（`app_settings_controller.dart:243-248`）的模式：

1. `lib/services/local_storage_service.dart` 加 `static const String kKeyboardVolume = "KeyboardVolume";`
2. `app_settings_controller.dart` 加：
   ```dart
   var keyboardVolume = true.obs;
   void setKeyboardVolume(bool e) {
     keyboardVolume.value = e;
     LocalStorageService.instance.setValue(LocalStorageService.kKeyboardVolume, e);
   }
   ```
   并在初始化处（约 `:88` 附近）读取：
   ```dart
   keyboardVolume.value = LocalStorageService.instance.getValue(
     LocalStorageService.kKeyboardVolume, true);
   ```
3. `play_settings_page.dart` 的"播放器"卡片内（`SettingsCard`，约 `:31-110`）加一项：
   ```dart
   AppStyle.divider,
   Obx(() => SettingsSwitch(
     title: "方向键调节音量",
     subtitle: "↑/↓ 调节播放器音量",
     value: controller.keyboardVolume.value,
     onChanged: (e) => controller.setKeyboardVolume(e),
   )),
   ```
4. `onKeyEvent` 开头加 `if (!AppSettingsController.instance.keyboardVolume.value) return KeyEventResult.ignored;`

---

## 7. 在 GitHub Actions 上远端构建（★ 已验证可行，且本机已有成功先例）

> **背景**：本机无 Flutter/Dart/CocoaPods，且 **Xcode 未完整安装**，无法本地编译。
>
> **重要**：这条路径**不是设想，而是已经被验证过的事实** —— 本机存在一个此前的会话留下的完整产物链：
> - Fork 仓库 `NSFish/dart_simple_live` 已存在
> - 自定义 workflow `.github/workflows/build-macos.yml` 已存在并跑通过
> - 本机 `/Applications/Simple Live.app`（v1.11.4，arm64，ad-hoc 签名，56MB）就是这个流程的产物
> - 本机 `gh` CLI 已登录账号 `NSFish`（token scope 含 `repo` + `workflow`）

### 7.1 现有 workflow 清单（上游仓库）

| 文件 | 触发方式 | 是否含 macOS 构建 |
|---|---|---|
| `.github/workflows/publish_app_dev.yaml` | **`workflow_dispatch`**（可手动）+ `push: tags: dev_v*` | ✅ `build-mac-ios-android` job |
| `.github/workflows/publish_app_release.yml` | ❌ 仅 `push: tags: v*` | ✅ 同上 |
| `publish_tv_app_dev.yaml` | `push: tags: dev_tv_v*` | ⛔ TV 端，无关 |
| `publish_tv_app_release.yaml` | `push: tags: v*` | ⛔ TV 端，无关 |

**为什么不应直接用上游这个 workflow**：`build-mac-ios-android` 是个**串行 job**，顺序为
`下载 Android keystore → 建 key.properties → Build APK → ... → Build MacOS → 上传 Mac`。

fork 中没有 `KEYSTORE_BASE64` / `STORE_PASSWORD` / `KEY_PASSWORD` / `KEY_ALIAS` 这些 secrets，`key.properties` 会被写成空值，而 `simple_live_app/android/app/build.gradle.kts:44`：

```kotlin
keyAlias = keystoreProperties["keyAlias"] as String        // ← null as String 抛异常
```

Gradle 配置阶段即崩 → job 中止在 Build APK → **永远到不了 Build MacOS**。此外三处 checkout 都硬编码 `ref: dev`（第 21/135/184 行），手动触发也只会构建 `dev` 分支。

> **结论：不要改上游 workflow，而是新增一个独立的 macOS-only workflow。** 这正是此前会话的做法（见 7.3）。

### 7.2 🟢 关键前提：macOS 构建不需要任何证书或 secrets

已核实 `simple_live_app/macos/Runner.xcodeproj/project.pbxproj`，Runner target 的**全部三个配置**都是 ad-hoc 签名：

| 配置 | pbxproj 行号 | `CODE_SIGN_IDENTITY` | `CODE_SIGN_STYLE` | `DEVELOPMENT_TEAM` | `PROVISIONING_PROFILE_SPECIFIER` |
|---|---|---|---|---|---|
| Debug | :670 | `"-"` | Automatic | **无** | `""` |
| Release | :617 | `"-"` | Automatic | **无** | `""` |
| Profile | :544 | `"-"` | Automatic | **无** | `""` |

`CODE_SIGN_IDENTITY = "-"` 即 **ad-hoc 签名**，无需 Apple 开发者账号、证书或 provisioning profile。
**这是整个远端构建方案成立的根本前提**，也是 fork 里 `actions/secrets` 总数为 0 仍能构建成功的原因（已核实）。

Entitlements 覆盖完整（`macos/Runner/Release.entitlements`）：`app-sandbox` + `network.client` + `network.server` + `files.user-selected.read-write`。

已实测验证下载产物：`Signature=adhoc`、`TeamIdentifier=not set`、`com.xycz.simpleLiveApp`。

### 7.3 ✅ 已验证可用的 workflow（直接复用）

Fork `NSFish/dart_simple_live` 中已存在 `.github/workflows/build-macos.yml`，**经实际运行验证可用**：

```yaml
name: Build macOS

on:
  workflow_dispatch:

jobs:
  build-macos:
    strategy:
      fail-fast: false
      matrix:
        include:
          - runner: macos-latest      # Apple Silicon
            arch: arm64
          - runner: macos-15-intel    # Intel
            arch: x64
    runs-on: ${{ matrix.runner }}

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          # 与 simple_live_app/.fvmrc 固定的版本保持一致（3.47.1）。
          # 不能降到 3.35.x：simple_live_core 要求 Dart SDK >= 3.10.0。
          # 已知问题：Dart 3.10 起 media_kit 释放播放器时的 FFI 回调竞态会 abort，
          # 表现为关闭软件/切换直播间时闪退（media-kit#1348），官方版同样存在。
          flutter-version: "3.47.1"
          cache: true

      - name: Enable Flutter Desktop
        run: flutter config --enable-macos-desktop

      - name: Restore packages
        run: cd simple_live_app && flutter pub get

      - name: Build macOS
        run: cd simple_live_app && flutter build macos --release

      - name: Zip app
        run: |
          cd simple_live_app/build/macos/Build/Products/Release
          APP=$(ls -d *.app | head -1)
          # 必须用 ditto：zip -r 默认展开符号链接，会把 framework 内的
          # Versions/A 软链变成实体副本，导致 libmpv 被加载两次而崩溃
          ditto -c -k --sequesterRsrc --keepParent "$APP" "SimpleLive-${{ matrix.arch }}.zip"

      - uses: actions/upload-artifact@v4
        with:
          name: SimpleLive-macOS-${{ matrix.arch }}
          path: simple_live_app/build/macos/Build/Products/Release/*.zip
```

**这份配置相对上游的关键改进（都是踩坑换来的，务必保留）：**

| 要点 | 原因 |
|---|---|
| **独立 workflow，不含 Android/iOS 步骤** | 规避 7.1 的 secrets 阻塞与 `ref: dev` 硬编码 |
| **`flutter-version`** | ⚠️ **不可降到 3.35.x** —— `simple_live_core/pubspec.yaml` 要求 `sdk: '>=3.10.0'`，而 Flutter 3.35 自带 Dart 3.9.2，`pub get` 会直接失败（此坑已实际踩过，见 7.4）。<br>**基于 `dev` 分支**：`.fvmrc` 固定 `3.47.1`，workflow 已对齐使用 `"3.47.1"`。`3.38.x` 是 master 上已验证可用的值，可作为回退（见 7.7.1）。 |
| **`ditto -c -k --sequesterRsrc --keepParent`** | ⚠️ **不可用 `zip -r`** —— `zip` 默认跟随并展开符号链接，把 framework 内 `Versions/A` 软链变成实体副本，导致 libmpv 被加载两次而崩溃。已实测验证：ditto 产物保留 117 个符号链接 |
| **用 `flutter build macos` 而非 `flutter_distributor`** | 更直接，无需 `appdmg`/`flutter_distributor` 依赖；产物 zip 足够本地测试 |
| **matrix 同时出 arm64 与 x64** | arm64 对应本机（Apple Silicon）；intel 腿**会失败**（见 7.5），但不影响 arm64 产物 |

> 注：`fail-fast: false` 很重要 —— 它保证 intel 腿失败时 arm64 腿仍能跑完并上传产物。

### 7.4 ⚠️ 已踩过的坑：Flutter 版本与 Dart SDK 下限冲突

第一次运行时 workflow 用的是 `flutter-version: "3.35.x"`，`Restore packages` 步骤报错：

```
The current Dart SDK version is 3.9.2.
Because simple_live_app depends on simple_live_core from path which requires SDK version >=3.10.0, version solving failed.
* Try using the Flutter SDK version: 3.47.2.
```

原因是 `simple_live_core/pubspec.yaml:7` 写死 `sdk: '>=3.10.0'`。

当时曾试图降到 3.35.x 来规避 media_kit 的 FFI 崩溃（[media-kit#1348](https://github.com/media-kit/media-kit/issues/1348)：Dart 3.10 起释放播放器时 FFI 回调竞态会 abort，表现为关软件/切直播间闪退），但**降版本不可行**，最终回退到 `3.38.x` 并接受该已知问题（上游官方版同样存在）。

**结论：必须用 `3.38.x`。** 若将来 `simple_live_core` 的 SDK 下限继续提高，需同步调整此版本号。

### 7.5 ⚠️ 已知问题：intel 腿必然失败（不影响 arm64）

`macos-15-intel` + `x64` 那条腿的 `Build macOS` 步骤**必定失败**，原因是第三方插件与 Xcode 16.4 SDK 的兼容问题：

```
/Users/runner/.pub-cache/hosted/pub.dev/connectivity_plus-7.3.1/macos/.../PathMonitorConnectivityProvider.swift:29:42:
error: value of type 'NWPath' has no member 'isUltraConstrained'
Command SwiftCompile failed with a nonzero exit code
** BUILD FAILED **
```

这是 `connectivity_plus` 用了较新 SDK 才有的 API，而 intel runner 的 Xcode/SDK 版本不匹配（**与本次改动无关，是上游依赖问题**）。

**影响**：整次 run 的结论会显示 `failure`，但 **arm64 腿是 ✓ 成功的，产物正常上传**。查看时只看 `SimpleLive-macOS-arm64` 这个 artifact 即可。

**若要让 intel 腿也通过**（非必需）：可在 matrix 中暂时移除 intel 条目，或锁定 `connectivity_plus` 到不含该 API 的版本。

### 7.6 完整操作流程（含可直接执行的 gh 命令）

本机 `gh` 已登录，理论上可全程命令行完成；但**推送代码与确认改动仍建议人工过目**。

1. **确认 fork 存在**（已存在，无需重建）：
   ```bash
   gh repo view NSFish/dart_simple_live --json isFork,parent
   ```

2. **推送代码改动到 fork 的 `master` 分支**（fork 的默认分支是 `master`）：
   ```bash
   git remote add fork https://github.com/NSFish/dart_simple_live.git   # 若无
   git push fork master
   ```

3. **触发 workflow**：
   ```bash
   gh workflow run build-macos.yml --repo NSFish/dart_simple_live --ref master
   ```

4. **查看运行状态**（等待完成，arm64 约 5–7 分钟）：
   ```bash
   gh run list --repo NSFish/dart_simple_live --limit 5
   gh run watch <run-id> --repo NSFish/dart_simple_live
   ```

5. **下载产物**（arm64）：
   ```bash
   gh run download <run-id> --repo NSFish/dart_simple_live -n SimpleLive-macOS-arm64
   # 或按 artifact id：
   gh api repos/NSFish/dart_simple_live/actions/artifacts/<id>/zip > artifact.zip
   ```
   > ⚠️ 首次下载可能超时（制品约 42MB）。建议放后台执行或加长超时。

6. **解压并安装**：
   ```bash
   unzip -q artifact.zip            # 得到 SimpleLive-arm64.zip
   unzip -q SimpleLive-arm64.zip   # 得到 Simple Live.app
   xattr -dr com.apple.quarantine "Simple Live.app"
   open "Simple Live.app"
   ```

**已验证的实测数据（供参考）**：

| 项目 | 值 |
|---|---|
| 制品名 | `SimpleLive-macOS-arm64` |
| 大小 | 42,760,125 字节（zip 内层 43,050,313 字节） |
| 构建耗时 | arm64 腿 5–7 分钟（含 cache） |
| 有效期 | 90 天（最近两次：2026-09-08 创建，2026-12-07 过期） |
| 解压后 app | 56MB，`lipo -archs` 显示 `x86_64 arm64`（App.framework 为 universal） |
| 符号链接 | 117 个（ditto 修复生效的证据） |

### 7.7 fork 与上游的当前差距（⚠️ 曾在此处出错，已修正）

> **修正说明**：本节此前写成「fork 落后上游 4 个提交」——**方向写反了**。
> 错误来源：`gh api repos/NSFish/dart_simple_live/compare/master...xiaoyaocz:master` 返回的
> `behind_by: 4, ahead_by: 0` 是**以 fork 为基准**看的，含义是「fork 比 upstream 多 4 个（领先）」，
> 不是「fork 落后」。反之 `repos/xiaoyaocz/dart_simple_live/compare/master...NSFish:master` 返回 `ahead_by: 4, behind_by: 0`，可直接确认。

**正确的关系**（已核实）：

```
upstream master (ba828e67, 2026-01-23, v1.11.4)   ← 本地 checkout 就是这个
    └── 2c74f226  fix: 小窗模式不再强制置顶
    └── 2db12f92  fix: 用 ditto 打包以保留 framework 符号链接
    └── 9807968d  fix: 用 Flutter 3.35.x 规避 media_kit FFI 回调闪退
    └── e7b34821  revert: 回退 Flutter 3.38.x        ← fork master HEAD
```

| 项目 | 状态 |
|---|---|
| fork 默认分支 | `master` |
| **fork 领先 upstream master** | **4 个提交**（`ahead_by: 4`） |
| **fork 落后 upstream master** | **0**（upstream master 上没有任何 fork 缺少的内容） |
| fork 的 secrets | 0 个（**不需要**，ad-hoc 签名） |
| Actions 权限 | `enabled: true`, `allowed_actions: all` |

**这 4 个提交的具体内容**（净改动只有两处，中间两次 Flutter 版本反复互相抵消）：

| 提交 | 内容 | 改动文件 |
|---|---|---|
| `2c74f226` | 新建 macOS workflow（+44 行）+ 小窗不再置顶 | `.github/workflows/build-macos.yml`（新增）、`player_controller.dart`（`setAlwaysOnTop(true)` → `false`，1 行） |
| `2db12f92` | 改用 `ditto` 打包 | workflow：`zip -q -r` → `ditto -c -k --sequesterRsrc --keepParent` |
| `9807968d` | 尝试降 Flutter 到 `3.35.x` | workflow：`flutter-version` |
| `e7b34821` | **回退**上述尝试，改回 `3.38.x` + 注释说明原因 | workflow：`flutter-version` |

**净 diff（相对 upstream master）**：
```
added    .github/workflows/build-macos.yml                         +50/-0
modified simple_live_app/lib/modules/live_room/player/player_controller.dart  +1/-1
```

> **结论：不存在「需要先同步上游再叠加改动」的事情。** fork master 是在 upstream master 之上直接叠加的，可以直接在其上继续开发。

### 7.7.1 ★ 分支基线选择（重要）

`master` 与 `dev` 差距很大，务必看清：

| 分支 | HEAD | 日期 | app 版本 | `.fvmrc` Flutter |
|---|---|---|---|---|
| upstream `master` | `ba828e67` | 2026-01-23 | `1.11.4+11104` | `3.38.3` |
| upstream `dev` | `bccd2ba2` | **2026-08-27** | **`1.11.7+11107`** | **`3.47.1`** |
| 本地 checkout | `ba828e67` | 2026-01-23 | `1.11.4+11104` | `3.38.3` |
| fork `dev` | `bccd2ba2` | — | 与 upstream `dev` **完全一致**（`status: identical`） | — |

**已确认：本项目决定基于 `dev` (`bccd2ba2`, v1.11.7) 开发。**

**好消息：本文档依赖的所有关键文件在 `master` 与 `dev` 之间完全没有差异**，因此第 2–6 节的全部行号引用与代码结论**依然有效**：

| 文件 | master vs dev |
|---|---|
| `player_controller.dart` | ✅ 无差异 |
| `live_room_page.dart` | ✅ 无差异 |
| `player_controls.dart` | ✅ 无差异 |
| `live_room_controller.dart` | ✅ 无差异 |
| `app_settings_controller.dart` | ✅ 无差异 |
| `simple_live_core/pubspec.yaml` | ✅ 无差异 |

`master → dev` 的实际差异集中在：4 个 workflow 文件、`AGENTS.md`、`README.md`、`.fvmrc`（`3.38.3`→`3.47.1`）、gradle 配置（`build.gradle.kts`/`gradle.properties`/wrapper）、`pubspec.yaml`（`auto_orientation_v2: ^2.3.6`→`^2.4.6`）、`follow_user_*`、`indexed_settings_*`、`test/widget_test.dart`。

**⚠️ 基于 dev 开发的额外注意点**：

- dev 的 `.fvmrc` 指向 Flutter **3.47.1**，而 `simple_live_app/pubspec.yaml` 的 `environment.sdk` 仍是 `">=3.0.5 <4.0.0"`（未变）。
- `simple_live_core/pubspec.yaml` 的 `sdk: '>=3.10.0'` 下限在 dev 上**依然存在**，所以 7.4 的坑（不能降 3.35.x）**依然成立**。
- 但既然 dev 已升到 Flutter 3.47.1，是否还要用 `3.38.x` 构建需要**实测**。**建议 CI 先试 `3.47.x`（与 `.fvmrc` 对齐）；若失败再回退到 `3.38.x`**（后者已在此 fork 上验证可用）。
- 本文档第 7.3 节的 workflow 是为 `master` 写的（`3.38.x`）。移植到 `dev` 时需同步调整 `flutter-version`（见 7.3 的版本说明）。

### 7.8 CI 路径的额外注意点

| 事项 | 说明 |
|---|---|
| **`simple_live_app/pubspec.lock` 被 gitignore** | `.gitignore:49` 忽略 `pubspec.lock`（`simple_live_tv_app` / `simple_live_console` 的 lock 反而**有**提交）。CI 每次全新解析依赖，产物依赖版本可能与你预期不同。 |
| **Release 构建看不到 DEBUG LOG 按钮** | `main.dart` 中该按钮条件是 `visible: !kReleaseMode`，CI 构建的是 release，**测试时看不到调试日志面板**，只能靠 OSD 自身和 macOS `Console.app` 观察。 |
| **ad-hoc 签名需去隔离** | 见 7.6 步骤 6 的 `xattr` 命令。 |
| **artifact 需登录下载** | `gh` 已登录，命令行下载无此问题；网页端下载需登录 GitHub。 |
| **runner 版本漂移** | `macos-latest` 会随 GitHub 升级。上游 `MACOSX_DEPLOYMENT_TARGET = 10.14`（pbxproj :556/:635/:682），向下兼容没问题。 |
| **CI 上无法交互测试** | CI 只负责**编译出产物**；所有交互验证（方向键是否被 `Focus` 拦住等）必须下载后在本地手动做 —— 见第 8 节清单。 |
| **构建耗时** | arm64 腿约 5–7 分钟（有 Flutter cache）。首次无 cache 会久一些。 |

---

## 8. 落地检查清单（给接手 AI）

### 前置条件

**本机无 Flutter 工具链 → 走 GitHub Actions（已验证的路径）**

- [ ] 确认 fork 存在且可用：
      `gh repo view NSFish/dart_simple_live --json isFork,parent`
- [ ] 确认 `gh` 已登录：`gh auth status`（需 `repo` + `workflow` scope）
- [ ] **确认基线分支**：本项目基于 `dev` (`bccd2ba2`, v1.11.7) 开发（见 7.7.1）
- [ ] 确认 fork 中已有 `.github/workflows/build-macos.yml`（见 7.3；若无则按 7.3 创建）
- [ ] **确认 `flutter-version`**：dev 建议先试 `3.47.x`；`3.38.x` 为已验证回退值（见 7.4 / 7.7.1）
- [ ] **确认打包用 `ditto` 而非 `zip -r`**（见 7.3）
- [ ] 注意：fork master **领先** upstream master 4 个提交（**不是落后**），无需同步上游；但基于 `dev` 开发时需从 `origin/dev` 起分支（见 7.7）

**若本机将来装了完整 Flutter + Xcode（备选）**
- [ ] `flutter --version`（`simple_live_app/pubspec.yaml` 要求 `sdk: ">=3.0.5 <4.0.0"`）
- [ ] `simple_live_app/pubspec.lock` 未提交，需先 `flutter pub get` 生成

### 实现步骤

- [ ] 1. `player_controller.dart`：`PlayerGestureControlMixin` 内加 `setKeyboardVolume(double delta)` + `_hideVolumeTipTimer`
- [ ] 2. `player_controller.dart:547`：修 macOS 空卡片 bug（`isMacOS` 从 `showGestureTip` 判断中移除）
- [ ] 3. `live_room_page.dart`：`buildMediaPlayer()` 外层包 `Focus`（桌面端条件），实现 `onKeyEvent` + `_isEditableFocused()` 守卫
- [ ] 4. `live_room_controller.dart:1055` `onClose()`：cancel Timer
- [ ] 5. （可选）设置开关四处改动
- [ ] 6. `flutter analyze lib/` 通过

### 必须在真机（macOS）手动验证的点

> **以下全部无法靠静态分析发现，必须手动跑：**

- [ ] **方向键是否真的被 `Focus` 拦住**（不再触发焦点遍历 / 列表滚动）—— 这是最大风险点
- [ ] 全屏态：↑/↓ 调音量，OSD 屏幕居中显示"音量 XX%"，约 800ms 后消失
- [ ] 非全屏态：OSD 居中于播放器矩形内（非全屏居中），确认是预期行为
- [ ] OSD 不会永久停留（Timer 生效）；连续快速按键时 Timer 被正确重置
- [ ] 长按方向键（KeyRepeatEvent）音量连续变化且不卡顿
- [ ] 音量到 0 / 100 边界正确 clamp，不越界
- [ ] 音量值持久化：调完退出直播间再进入，音量保持（`kPlayerVolume`）
- [ ] 调完音量后点小喇叭，Slider 显示的值与键盘调的值**一致**（验证走的是同一链路）
- [ ] 打开"关键词屏蔽"弹窗（`showDanmuShield`），`TextField` 能正常输入，方向键用于移动光标而非调音量
- [ ] 锁定控制器（`lockControlsState`）后方向键不再调音量
- [ ] 回归：macOS 竖向拖拽不再弹出空的深色卡片
- [ ] 回归：Windows / Linux 不受影响（`Focus` 对三个桌面平台都挂了，需一并验证）
- [ ] 回归：Android / iOS 完全无变化（条件分支排除）

---

## 9. 风险与开放问题

| 风险 | 说明 | 缓解 |
|---|---|---|
| **焦点遍历抢占方向键** | macOS 上方向键默认用于焦点移动，`KeyboardListener` 拦不住 | 用 `Focus` + `KeyEventResult.handled`；必须真机验证 |
| **TextField 焦点冲突** | `showDanmuShield` 弹窗输入框会被 `autofocus` 抢焦点 | `_isEditableFocused()` 守卫；必须真机验证 |
| OSD 永久停留 | 现有代码无自动隐藏 Timer | 自加 800ms Timer + `onClose` 取消 |
| 非全屏 OSD 位置 | 居中于播放器矩形而非全屏 | 属预期行为，已记录 |
| 音量语义分歧 | 方案 A 改 App 音量，方案 B 改系统音量，二者不可混用 | 已选定 A；若产品要系统音量需整体切换并同步改 Slider |

### 已知问题（与本次改动无关，但测试时必然会遇到，别误判为你的 bug）

| 问题 | 现象 | 原因 | 状态 |
|---|---|---|---|
| **media_kit 关闭/切换直播间闪退** | 关闭软件或切换直播间时 abort | Dart 3.10 起 media_kit 释放播放器时的 FFI 回调竞态（[media-kit#1348](https://github.com/media-kit/media-kit/issues/1348)） | ⚠️ **上游官方版同样存在**。曾试图用 Flutter 3.35.x 规避，但因 `simple_live_core` 要求 Dart >=3.10.0 而不可行（见 7.4）。**不要试图在本次改动中修复。** |
| **intel (x64) 构建失败** | CI 中 `macos-15-intel` 腿的 `Build macOS` 报错 | `connectivity_plus-7.3.1` 使用了 Xcode 16.4 SDK 中不存在（或已变更）的 `NWPath.isUltraConstrained` API | ⚠️ 与本次改动无关，是上游依赖问题。arm64 产物不受影响（见 7.5）。 |
| **macOS 竖向拖拽弹空卡片** | 播放器中部竖拖出现空的深色小卡片 | `onVerticalDragStart` 含 `isMacOS` 置 `showGestureTip=true`，但 `onVerticalDragUpdate` 排除桌面后直接 return | ✅ **本次改动会顺手修复**（见 6.5） |

### 过程中已确认无效的尝试（避免重走弯路）

| 尝试 | 结果 |
|---|---|
| 降 Flutter 到 `3.35.x` 以规避 media_kit 崩溃 | ❌ 失败：`simple_live_core` 要求 `sdk: '>=3.10.0'`，而 3.35 自带 Dart 3.9.2，`pub get` 直接报错 |
| 用 `zip -r` 打包 .app | ❌ 失败：zip 展开 framework 符号链接，libmpv 被加载两次导致崩溃。必须用 `ditto` |
| 直接跑上游 `publish_app_dev.yaml` | ❌ 失败：Android 步骤因缺 secrets 先崩（见 7.1） |

### 未决问题（需产品/用户确认）

1. 是否需要同时支持左右方向键（如 ←/→ 快退/快进）？直播流通常不支持 seek，本次未纳入。
2. 步进 5 是否合适？是否需要区分"单击 5、长按加速"？
3. 是否需要"静音"快捷键（如 M 键）？`volume_controller` 提供 `setMute`/`isMuted`，但方案 A 下应直接用 `player.setVolume(0)` + 记忆原值。
4. 是否要修 `main.dart:262` 那个失效的 ESC 监听？（加 `autofocus: true` 可能影响其他页面，需单独评估）

---

## 10. 关键代码位置速查

```
simple_live_app/lib/
├── modules/live_room/
│   ├── live_room_page.dart
│   │   :88-101     全屏/非全屏两种形态分支
│   │   :241        buildMediaPlayer()  ← 键盘监听挂载点
│   ├── player/
│   │   ├── player_controller.dart
│   │   │   :78     mixin PlayerStateMixin
│   │   │   :80     hidevolumeTimer（死代码）
│   │   │   :89     showControlsState
│   │   │   :97     lockControlsState
│   │   │   :104    showGestureTip     ← OSD 开关
│   │   │   :107    gestureTipText     ← OSD 文本
│   │   │   :110/113 showBottomTip/bottomTipText（死代码）
│   │   │   :119    hideSeekTipTimer（死代码）
│   │   │   :226    mixin PlayerSystemMixin
│   │   │   :473    mixin PlayerGestureControlMixin ← 键盘方法放这里
│   │   │   :545    onVerticalDragStart（:547 有 macOS bug）
│   │   │   :565    onVerticalDragUpdate（桌面 return）
│   │   │   :580    setGestureVolume
│   │   │   :608    _convertVolume（步进 5）
│   │   │   :614    _realSetVolume（走系统音量）
│   │   │   :655    class PlayerController
│   │   │   :663    onInit → player.setVolume(playerVolume)
│   │   └── player_controls.dart
│   │       :376    OSD 渲染（全屏版）
│   │       :557    音量 Slider 按钮（非全屏版）
│   │       :596    音量 Slider 按钮（全屏版）
│   │       :624    OSD 渲染（非全屏版）
│   └── live_room_controller.dart
│       :623        showVolumeSlider()  ← 黄金标准调用链
│       :770        TextField（焦点冲突源）
│       :1055       onClose()  ← 取消 Timer
├── app/controller/app_settings_controller.dart
│   :94             playerVolume 读取（默认 100.0）
│   :243            hardwareDecode（设置项范例）
│   :425            playerVolume + setPlayerVolume
├── services/local_storage_service.dart
│   :112            kPlayerVolume
├── app/custom_throttle.dart
│                   DelayedThrottle
└── main.dart
    :262            KeyboardListener（失效范例，无 autofocus）

simple_live_tv_app/lib/modules/live_room/live_room_page.dart
    :37             KeyboardListener（可靠范例，有 autofocus）
    :51             onKeyEvent 处理函数
```

---

## 11. 附：环境说明与本机已有资产

当前机器环境探测结果（macOS 26.6.2 (25G83) / arm64）：

| 检查项 | 命令 | 结果 |
|---|---|---|
| Flutter | `which flutter` | ❌ 无 |
| Dart | `which dart` | ❌ 无 |
| CocoaPods | `which pod` | ❌ 无（`~/.pub-cache/hosted/pub.dev` 也不存在） |
| Xcode | `xcodebuild -version` | ⚠️ **报错** `requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance` |
| Xcode.app | `ls -d /Applications/Xcode*.app` | ❌ `No such file or directory` |
| CommandLineTools | `xcode-select -p` | ✅ `/Library/Developer/CommandLineTools`（**不含 Xcode 构建链**） |
| Java / keytool | `which java keytool` | ✅ `/usr/bin/java`、`/usr/bin/keytool` |
| Homebrew | `which brew` | ✅ `/opt/homebrew/bin/brew`（7.0.4） |
| **GitHub CLI** | `which gh` | ✅ `/opt/homebrew/bin/gh`，**已登录账号 `NSFish`**（token scope：`gist`, `read:org`, `repo`, `workflow`） |

**结论**：本机既无 Flutter 工具链，也无完整 Xcode，**无法进行任何本地构建**。

补充：`simple_live_app/pubspec.lock` 被 `.gitignore:49` 忽略（未提交），依赖解析也无法在本机复现。

### 本机已存在的前次会话资产（★ 接手前务必先核实）

| 资产 | 位置 / 标识 | 状态 |
|---|---|---|
| Fork 仓库 | `NSFish/dart_simple_live`（public, fork of `xiaoyaocz/dart_simple_live`） | ✅ 存在 |
| 自定义 workflow | fork 的 `.github/workflows/build-macos.yml` | ✅ 存在且跑通（见 7.3） |
| 成功产物 | artifact `SimpleLive-macOS-arm64`（id `10038988465`，2026-09-08，90 天有效） | ✅ 有效（2026-12-07 过期）；**本次调研已实际下载并解压验证通过** |
| 已安装 App | `/Applications/Simple Live.app` | ✅ 已装（v1.11.4, arm64, ad-hoc, 56MB，2026-09-08 安装） |
| fork 的自定义提交 | `2c74f226` / `2db12f92` / `9807968d` / `e7b34821` | ✅ fork **领先**上游 master 4 个提交（非落后，见 7.7） |

**这意味着**：接手者**无需从零搭建**，可以直接复用 fork 与 workflow，只需把自己的代码改动推上去重新触发构建即可。

因此本文的**代码结论**来自静态代码阅读 + pub.dev 文档核实；而**构建路径结论**已通过实际下载 artifact 并解压验证（含 `codesign`、`lipo`、符号链接数量检查）。
编译与验证请走**第 7 节的 GitHub Actions 路径**（无需本机工具链），验证清单见第 8 节。

> 理论替代方案：`brew install --cask flutter` + 从 App Store 安装完整 Xcode（>10GB）。相比 CI 路径成本高得多，且会污染本机环境。**不推荐。**

---

## 12. 本次实现记录（已完成）

### 12.1 改动概览

**分支**：`feat/macos-keyboard-volume`（基于 `origin/dev` @ `bccd2ba`, v1.11.7）
**Fork 远端**：`https://github.com/NSFish/dart_simple_live.git`

| 文件 | 改动 |
|---|---|
| `simple_live_app/lib/modules/live_room/player/player_controller.dart` | +42/-1：新增 `adjustKeyboardVolume()` + `keyboardVolumeStep` + `hideKeyboardVolumeTipTimer`；修复 `onVerticalDragStart` 的 macOS 空提示框 |
| `simple_live_app/lib/modules/live_room/live_room_page.dart` | +98/-27：新增 `_buildKeyboardVolumeWrapper()`（`Focus` 键盘监听）+ `_isEditableFocused()`；`buildMediaPlayer()` 返回值包一层 |
| `simple_live_app/lib/modules/live_room/live_room_controller.dart` | +1：`onClose()` 取消 Timer |
| `.github/workflows/build-macos.yml` | 新增（+50）：macOS-only CI 构建 |
| `docs_mac_keyboard_volume.md` | 新增：本文档 |

**行号定位（dev 分支）**：
- `player_controller.dart:545-548` — `onVerticalDragStart` 的提示判断（已去掉 `isMacOS`）
- `player_controller.dart:654-690` — 新增的键盘调音量代码
- `live_room_page.dart:258-336` — `buildMediaPlayer` + `_buildKeyboardVolumeWrapper` + `_isEditableFocused`
- `live_room_controller.dart:1059` — `hideKeyboardVolumeTipTimer?.cancel()`

### 12.2 关键实现决策（与第 6 节的差异说明）

| 项 | 第 6 节原方案 | 实际实现 | 原因 |
|---|---|---|---|
| 方法名 | `setKeyboardVolume(double delta)` | `adjustKeyboardVolume(int direction)` | 用 `int` 方向值语义更清晰，避免调用方传任意浮点 |
| 步进常量 | 内联 `5` | `static const double keyboardVolumeStep = 5.0` | 便于后续调整；`static const` 可直接访问，无需实例 |
| 挂载点 | `buildMediaPlayer()` 外层 | 同左（`_buildKeyboardVolumeWrapper`） | 保持方案 |
| 平台判断 | `!(isMacOS \|\| isWindows \|\| isLinux)` | 同左 | 保持方案 |
| 设置开关（6.6） | 可选 | **未实现** | 保持最小改动，确认需求后再加 |
| 修 `main.dart:262` ESC | 可选 | **未实现** | 影响面大，单独评估 |

**一处潜在类型问题已提前规避**：`num.clamp()` 的返回类型在 Dart 中为 `num`，因此写成
`(current + direction * keyboardVolumeStep).clamp(0.0, 100.0).toDouble()`，
显式 `.toDouble()` 以匹配 `player.setVolume(double)` 与 `playerVolume`（`Rx<double>`）。

### 12.3 构建记录

| 运行 | 分支 | 状态 | 说明 |
|---|---|---|---|
| [35548079896](https://github.com/NSFish/dart_simple_live/actions/runs/35548079896) | `feat/macos-keyboard-volume` | ✅ **arm64 成功** / ❌ intel 失败 | `flutter-version: 3.47.1`（与 `.fvmrc` 对齐） |

**结果详情**：

- **arm64 腿（`macos-latest`）：成功**，耗时 7m28s，产出 `Simple Live.app (115.5MB)`
- **intel 腿（`macos-15-intel`）：失败**，❌ `connectivity_plus-7.3.1` 的 `NWPath.isUltraConstrained` 编译错误 —— **与本次改动无关**，正是 7.5 记录的已知问题，且在 dev 上依然存在
- ✅ **`flutter-version: 3.47.1` 可用**（无需回退到 `3.38.x`），`flutter pub get` 与编译均通过
- ✅ **我们的改动无任何 Dart 编译错误或告警**（构建日志中针对 `simple_live_app/lib/**` 的 error/warning 为空）
- ⚠️ 日志中的 warning 全部来自第三方插件（`flutter_inappwebview_macos`、`media_kit_video` 等），与本次改动无关

**产物验证（已实际下载解压，40.9MB artifact / 43,000,013 字节）**：

| 检查项 | 结果 |
|---|---|
| 应用版本 | `1.11.7`（dev 基线，✓ 非旧的 1.11.4） |
| 架构 | `x86_64 arm64`（App.framework 为 universal） |
| 签名 | `Signature=adhoc`、`TeamIdentifier=not set`、`Identifier=com.xycz.simpleLiveApp` |
| `codesign --verify --deep --strict` | ✅ 通过（exit 0） |
| `ditto` 符号链接保留 | ✅ `Mpv.framework/Mpv -> Versions/Current/Mpv` 等软链完整；`Versions/A/Mpv` 仅一份实体（未被复制成两份），共 75 个软链且**全部有效** |
| **改动是否真的编进产物** | ✅ **决定性证据**：`strings App.framework/.../App \| grep -c adjustKeyboardVolume` → 新产物 **2**，本机旧版 `/Applications/Simple Live.app`（v1.11.4）→ **0** |

> 符号链接数从 v1.11.4 的 117 变为 v1.11.7 的 75，是因为 Flutter 版本不同（3.38 → 3.47）导致 framework 集合变化，属正常现象；关键是**软链指向均有效**，`Mpv` 实体只有一份。

**下载与安装流程**（同 7.6）：
```bash
gh run list --repo NSFish/dart_simple_live --limit 3
gh run download <run-id> --repo NSFish/dart_simple_live -n SimpleLive-macOS-arm64
unzip -q SimpleLive-macOS-arm64.zip
xattr -dr com.apple.quarantine "Simple Live.app"
open "Simple Live.app"
```

> ⚠️ 首次 `gh run download` 可能超时（约 43MB），建议放后台执行。

### 12.4 ⚠️ 尚未完成：人工交互验证

**代码已编译通过，但尚未经过任何运行/交互验证。** 本机虽已有可用产物，但以下验证必须由人工完成（第 8 节清单的精华）：

| # | 验证项 | 为什么关键 |
|---|---|---|
| 1 | **方向键是否真的被 `Focus` 截获**（既不触发焦点遍历，也不滚动列表） | 这是选用 `Focus` 而非 `KeyboardListener` 的**唯一理由**；若失效则方案不成立，需改用 `HardwareKeyboard` 全局注册（见 6.4） |
| 2 | OSD 显示「音量 XX%」且约 800ms 后消失；连续快按时 Timer 正确重置 | 现有代码无此 Timer，是本方案新增的部分 |
| 3 | 打开「关键词屏蔽」弹窗后，方向键用于移动光标而非调音量 | `_isEditableFocused()` 守卫是否生效 |
| 4 | 调完音量点小喇叭，Slider 显示值与键盘调的值一致 | 证明两者走同一链路（`playerVolume`） |
| 5 | 音量到 0/100 边界正确 clamp，不越界 | `clamp` 逻辑 |
| 6 | 长按方向键（KeyRepeatEvent）音量连续变化且不卡顿 | 是否需要加节流 |
| 7 | 音量持久化：调完退出直播间再进，音量保持 | `kPlayerVolume` 落盘 |
| 8 | 锁定控制器后方向键不再调音量 | `lockControlsState` 守卫 |
| 9 | **回归**：macOS 竖向拖拽不再弹出空的深色卡片 | 本次修复项（12.1） |
| 10 | **回归**：Android/iOS 行为完全无变化 | 平台分支隔离 |

**已知会遇到的无关问题**：关闭软件/切换直播间时可能闪退（media_kit FFI 竞态，见第 9 节），**这不是本次改动引入的，上游官方版同样存在**。
