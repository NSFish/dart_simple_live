# macOS 方向键调节音量 —— 交接说明

> 给后续对话的 AI：本文档是**当前实现状态**的唯一交接说明，简洁优先。
> 旧的详细调研文档（`docs_mac_keyboard_volume.md`）已删除，如需考古可在 git 历史中找回（删除前的最后版本见 `538588a`）。

---

## 1. 一句话现状

macOS 直播间已支持 ↑/↓ 调节播放器音量，**已实现、已编译、已实测可用**。
小窗置顶修复已 cherry-pick、构建并全量验证通过（见 §2、§5）。
本机**无 Flutter/Xcode 工具链**，所有构建都走 GitHub Actions。

---

## 2. 代码在哪

**分支**：`feat/macos-keyboard-volume`（基于上游 `dev` @ `bccd2ba`, v1.11.7）
**Fork**：`https://github.com/NSFish/dart_simple_live.git`（当前仓库的 `origin` 即 fork，可直接 push）
**上游**：`https://github.com/xiaoyaocz/dart_simple_live.git`（原作者仓库，当前未配置为 remote，**绝对不要 push 过去**）

### 改动文件（4 个）

| 文件 | 改动 |
|---|---|
| `simple_live_app/lib/modules/live_room/player/player_controller.dart` | `PlayerGestureControlMixin` 内新增 `adjustKeyboardVolume()`（`:667-693`）+ `keyboardVolumeStep`（`:660`）+ `hideKeyboardVolumeTipTimer`（`:657`）；`onVerticalDragStart` 去掉 `isMacOS` 的提示判断（`:548`）；`enterSmallWindow` 置顶改 `false`（`:332`，与 `master` 的 `2c74f22` 对齐） |
| `simple_live_app/lib/modules/live_room/live_room_page.dart` | `buildMediaPlayer()` 外包 `_buildKeyboardVolumeWrapper()`（`:258`, `:298`）；新增 `_isEditableFocused()`（`:327`） |
| `simple_live_app/lib/modules/live_room/live_room_controller.dart` | `onClose()` 加 `hideKeyboardVolumeTipTimer?.cancel()`（`:1059`） |
| `.github/workflows/build-macos.yml` | 新增，macOS-only CI 构建 |

已推送到 fork：`48a3960`（小窗置顶修复）。CI run `35552233442` 构建中。

### 当前分支与 `fork/master` 的关系

| 分支 | 小窗置顶 | 键盘音量 |
|---|---|---|
| `fork/master`（v1.11.4 基线） | `false`（`2c74f22`） | 无 |
| `fork/feat/macos-keyboard-volume`（**当前分支**，v1.11.7 基线） | `false`（`48a3960`，与 `2c74f22` 对齐） | 有 |

两者兼得已完成，无需再 cherry-pick。

---

## 3. 实现要点（改代码前必读）

### 3.1 音量链路必须与 Slider 一致

```dart
player.setVolume(next);                                  // mpv 播放器音量 0-100
AppSettingsController.instance.setPlayerVolume(next);    // Rx + 落盘 kPlayerVolume
```

两者必须**同时调用**。`playerVolume` 是唯一数据源，`PlayerController.onInit` 会用它初始化音量。
`showVolumeSlider()`（`live_room_controller.dart:623`）走的是同一链路，新改动请勿偏离。

### 3.2 必须用 `Focus`，不能用 `KeyboardListener`

macOS 上方向键默认被 Flutter 的**焦点遍历**消费。`KeyboardListener.onKeyEvent` 返回 `void`，
无法声明"已消费"，拦不住；`Focus.onKeyEvent` 返回 `KeyEventResult.handled` 才能真正截获。

**✅ 已实测确认有效**，不需要改成 `HardwareKeyboard` 全局注册。

### 3.3 OSD 提示靠共享状态，无独立组件

复用 `PlayerStateMixin` 的 `showGestureTip`(RxBool) + `gestureTipText`(RxString)，
由 `player_controls.dart` 的 `:380`（全屏）和 `:628`（非全屏）两处渲染。

⚠️ 这两个字段**原本只由手势驱动，没有自动隐藏机制**（手势靠 `onVerticalDragEnd` 关闭）。
键盘没有"结束"事件，所以 `adjustKeyboardVolume` 里自加了 **800ms Timer**。改动时别删。

### 3.4 焦点冲突守卫

`_isEditableFocused()` 检查 `primaryFocus.context?.widget is EditableText`，
避免「关键词屏蔽」弹窗的 `TextField`（`live_room_controller.dart:770`）被抢焦点。

---

## 4. 怎么构建和测试

### 构建（本机无工具链，只能走 CI）

```bash
# 推送改动（当前 origin 即 fork，可直接 push）
git push origin feat/macos-keyboard-volume

# 触发构建
gh workflow run build-macos.yml --repo NSFish/dart_simple_live --ref feat/macos-keyboard-volume

# 等待（arm64 约 5-8 分钟）
gh run watch <run-id> --repo NSFish/dart_simple_live

# 下载产物（约 43MB，可能超时，建议后台执行）
gh run download <run-id> --repo NSFish/dart_simple_live -n SimpleLive-macOS-arm64
unzip -q SimpleLive-macOS-arm64.zip
xattr -dr com.apple.quarantine "Simple Live.app"
open "Simple Live.app"
```

> `gh` 已登录账号 `NSFish`（scope 含 `repo` + `workflow`）。
> 若 `gh` 报 cache 目录权限错误，先 `export XDG_CACHE_HOME=/tmp/ghcache`。

### 构建时的已知现象

| 现象 | 说明 |
|---|---|
| **intel 腿（`macos-15-intel`）必然失败** | `connectivity_plus-7.3.1` 用了 `NWPath.isUltraConstrained`，与 runner 的 Xcode 16.4 SDK 不兼容。**与本次改动无关**，看 arm64 那个 artifact 即可 |
| `flutter-version: "3.47.1"` | 与 `simple_live_app/.fvmrc` 对齐，**已验证可用**。不可降到 3.35.x（`simple_live_core` 要求 Dart >=3.10.0） |
| 打包必须用 `ditto` | 不可用 `zip -r`，否则 framework 符号链接被展开，libmpv 加载两次而崩溃 |

### 运行时的已知问题（不是你的 bug）

关闭软件/切换直播间可能闪退 —— media_kit FFI 回调竞态（[media-kit#1348](https://github.com/media-kit/media-kit/issues/1348)），**上游官方版同样存在**，别试图在功能改动里修它。

---

## 5. 已验证（2026-09-21，真机 v1.11.7 arm64 全量通过）

- ✅ 方向键被 `Focus` 截获，能调音量，不触发焦点遍历或界面滚动
- ✅ OSD 显示「音量 XX%」约 800ms 消失；连续按键 Timer 正确重置
- ✅ 「关键词屏蔽」弹窗内方向键用于移动光标（守卫生效）
- ✅ 音量 0/100 边界 clamp；长按连发不卡顿
- ✅ 音量持久化（退出直播间再进保持）；与小喇叭 Slider 值一致
- ✅ 回归：macOS 竖向拖拽不再弹空的深色卡片
- ✅ 回归：进入小窗不再强制置顶，退出小窗还原正常（`48a3960`）

---

## 6. 可能的后续改动方向

| 需求 | 怎么做 |
|---|---|
| 加设置开关 | 照抄 `app_settings_controller.dart:243` 的 `hardwareDecode` 模式（Rx + 落盘 + `SettingsSwitch`）；`onKeyEvent` 开头判断开关 |
| 改步进值 | `keyboardVolumeStep`（当前 `5.0`） |
| 改提示停留时间 | `adjustKeyboardVolume` 里的 `Duration(milliseconds: 800)` |
| 支持左右键快进 | 直播流通常不支持 seek，需先确认播放器能力 |
| 加静音快捷键 | 方案 A 下用 `player.setVolume(0)` + 记忆原值；不要用 `VolumeController`（那是系统音量，会与 Slider 脱节） |
| 让非全屏也无损生效 | 当前 `Focus` 包在 `buildMediaPlayer()` 外层，非全屏时焦点移走后方向键恢复滚动，这是预期行为 |

---

## 7. 构建产物位置

本机已放置可用测试版：

```
/Users/nsfish/Desktop/dart_simple_live/dist_test/Simple Live.app   (v1.11.7, arm64, 含 48a3960，CI run 35552233442)
```

该目录在仓库之外（避免误提交，`.gitignore` 另有 `dist_test/` 兜底）。直接双击打开即可。

---

## 8. 相关文档

- 上游 `AGENTS.md` —— 仓库通用规范（FVM 固定 Flutter 3.47.1；四个子项目各自 `pubspec.lock`；用 `fvm flutter ...`）。
