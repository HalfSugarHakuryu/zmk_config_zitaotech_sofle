# ZitaoTech Sofle 项目探索说明

更新时间：2026-03-24 15:48:39 +08:00

## 1. 这是什么项目

这是一个基于 ZMK 的自定义分体键盘固件仓库，不只是常见的用户层 `keymap` 配置，还包含了完整的自定义 `board`、`shield`、`dts/dtsi`、`Kconfig`、`CMakeLists.txt` 和多份自定义驱动代码。

从仓库内容看，它服务的是一把名为 `ZitaoTech Sofle` 的分体键盘，左右手的硬件能力并不完全对称：

- 左手侧：`lpm_view + left_bbtrackball`
- 右手侧：`lpm_view + right_trackpoint`
- 两侧都带显示、分体通信、背光/灯效相关能力
- 两侧都接入编码器

这意味着它本质上是一个“自定义硬件适配仓”，而不是“只改按键映射的配置仓”。

## 2. 当前编译入口

### 2.1 GitHub Actions

仓库的 CI 很简单，直接复用 ZMK 官方的用户配置构建工作流：

- `.github/workflows/main.yml`
- `build.yaml`
- `config/west.yml`

其中：

- `config/west.yml` 将 ZMK 固定在 `v0.3.0`
- `.github/workflows/main.yml` 复用 `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3.0`
- `build.yaml` 定义了 4 个构建目标

### 2.2 build.yaml 的构建矩阵

当前构建目标如下：

1. 左手正常固件：`zitaotech_sofle_left + lpm_view;left_bbtrackball`
2. 右手正常固件：`zitaotech_sofle_right + lpm_view;right_trackpoint`
3. 左手设置重置固件：`zitaotech_sofle_left + lpm_view;left_bbtrackball;settings_reset`
4. 右手设置重置固件：`zitaotech_sofle_right + lpm_view;right_trackpoint;settings_reset`

如果后续你只是补文档、图片、说明类文件，默认不会影响这个构建矩阵。

## 3. 目录职责梳理

### 3.1 根目录

- `README.md`
  现在内容非常少，只保留了项目标题和一句英文描述。
- `build.yaml`
  GitHub Actions 的构建入口。
- `config/zitaotech_sofle.json`
  物理布局定义，包含按键坐标、拇指区几何信息，以及左右编码器传感器声明。
- `config/zitaotech_sofle.keymap`
  一份完整的按键层定义。
- `config/west.yml`
  ZMK 依赖清单和版本锁定文件。

### 3.2 `config/boards/arm/zitaotech_sofle`

这一层是自定义板级定义，属于真正决定硬件能力的核心目录。

- `zitaotech_sofle.dtsi`
  公共底板定义，包含：
  - `nrf52840_qiaa` 芯片基底
  - 矩阵变换
  - 左右编码器
  - 分体输入和 USB CDC
  - Flash 分区
- `zitaotech_sofle_left.dts`
  左手侧实际设备树，定义：
  - 左手矩阵引脚
  - 键盘背光、轨迹球灯、底光
  - 电池检测
  - 左编码器启用
  - `lpm_view_spi`
- `zitaotech_sofle_right.dts`
  右手侧实际设备树，定义：
  - 右手矩阵引脚
  - 键盘背光、底光、蓝色状态灯
  - 电池检测
  - 右编码器启用
  - `lpm_view_spi`
- `Kconfig*`
  板级开关、分体角色、PWM/背光能力默认值。
- `CMakeLists.txt`
  左右半边分别挂入各自的 `keyboard_backlight.c`。
- `board.cmake`
  烧录/调试工具链设置，包含 `uf2`、`nrfjprog`、`openocd-nrf5`。

### 3.3 `config/boards/shields/left_bbtrackball`

左手的 Blackberry Trackball 盾层：

- `left_bbtrackball.overlay`
  定义轨迹球设备节点、输入监听器、轨迹球灯 PWM。
- `custom_driver_left/bbtrackball_input_handler.c`
  轨迹球核心输入逻辑：
  - 通过 GPIO 中断采集方向变化
  - 通过定时任务输出鼠标移动
  - 在特定层键按住时切换为鼠标移动模式
  - 否则走滚轮/滚动模式
- `custom_driver_left/trackball_led.c`
  轨迹球灯效逻辑：
  - 跟随轨迹球活动亮灭
  - 支持渐亮渐暗
  - 与 CapsLock / RGB 亮度联动

### 3.4 `config/boards/shields/right_trackpoint`

右手的 TrackPoint 盾层：

- `right_trackpoint.overlay`
  定义 I2C TrackPoint 设备、输入 split、监听器、自定义 LED PWM。
- `custom_driver_right/trackpoint_0x15.c`
  TrackPoint 输入驱动：
  - 以 I2C 地址 `0x15` 读取数据包
  - 正常情况下输出鼠标移动
  - 按住 Space 时切换为滚轮输出
  - 轮询周期为 5ms
- `custom_driver_right/custom_led.c`
  指点杆灯光逻辑：
  - 跟随背光亮度变化
  - 亮度变化后延迟熄灭
  - 保留最近一次有效亮度，供 TrackPoint 速度缩放使用

### 3.5 `config/boards/shields/lpm_view`

显示盾层：

- `lpm_view.overlay`
  把 `lpm_view_spi` 绑定到 `jdi,lpm009m360a` 显示设备。
- `lpm_view.conf`
  开启显示、自定义状态控件、关闭空闲熄屏。
- `custom_status_screen.c`
  状态页入口。
- `widgets/`
  自定义状态组件、外围状态页、动画/静态图资源。
- `display_driver/lpm009m360a.c`
  自定义显示驱动实现。

### 3.6 `config/boards/shields/settings_reset`

这是用于清空持久化设置的辅助盾层，通常用于蓝牙配对、保存配置异常后的恢复固件。

## 4. 当前键位和输入行为

从两份 `keymap` 来看，项目至少包含以下输入能力：

- 3 层键位：
  - `default_layer`
  - `func_layer`
  - `RES_layer`
- 双编码器：
  - 默认层控制音量和左右方向
  - 功能层控制蓝牙设备切换和上下方向
- 鼠标按键集成：
  - 左右键
  - 中键或其他鼠标点击行为
- 蓝牙控制：
  - `BT_CLR`
  - `BT_NXT`
  - `BT_PRV`
- 显示、背光、底光、输出切换也都已经接入

整体上，这套固件不是纯键盘方案，而是“键盘 + 显示 + 指点设备 + 灯效”的综合交互固件。

## 5. 维护时最值得记住的入口

如果以后你要改功能，可以按下面的思路找入口：

### 5.1 改键位

优先检查：

- `config/zitaotech_sofle.keymap`
- `config/boards/arm/zitaotech_sofle/zitaotech_sofle.keymap`

### 5.2 改物理布局或矩阵映射

优先检查：

- `config/zitaotech_sofle.json`
- `config/boards/arm/zitaotech_sofle/zitaotech_sofle.dtsi`
- `config/boards/arm/zitaotech_sofle/zitaotech_sofle_left.dts`
- `config/boards/arm/zitaotech_sofle/zitaotech_sofle_right.dts`

### 5.3 改轨迹球

优先检查：

- `config/boards/shields/left_bbtrackball/left_bbtrackball.overlay`
- `config/boards/shields/left_bbtrackball/custom_driver_left/bbtrackball_input_handler.c`
- `config/boards/shields/left_bbtrackball/custom_driver_left/trackball_led.c`

### 5.4 改 TrackPoint

优先检查：

- `config/boards/shields/right_trackpoint/right_trackpoint.overlay`
- `config/boards/shields/right_trackpoint/custom_driver_right/trackpoint_0x15.c`
- `config/boards/shields/right_trackpoint/custom_driver_right/custom_led.c`

### 5.5 改显示页

优先检查：

- `config/boards/shields/lpm_view/custom_status_screen.c`
- `config/boards/shields/lpm_view/widgets/`
- `config/boards/shields/lpm_view/display_driver/lpm009m360a.c`

### 5.6 改编译和目标组合

优先检查：

- `build.yaml`
- `config/west.yml`
- `.github/workflows/main.yml`

## 6. 我在探索中看到的几个注意点

### 6.1 有两份功能不完全相同的 keymap

仓库里同时存在：

- `config/zitaotech_sofle.keymap`
- `config/boards/arm/zitaotech_sofle/zitaotech_sofle.keymap`

而且这两份文件并不只是排版不同，实际绑定也有差异，例如：

- 某些鼠标键位定义不同
- `func_layer` 的若干按键内容不同
- 退格、方括号、数字键盘区等映射存在差异

这会给后续维护带来一个现实问题：

> 修改键位之前，最好先确认“实际编译时生效的是哪一份 keymap”，否则很容易出现“改了但没生效”或“两处配置漂移”的情况。

### 6.2 左右半边的键盘背光驱动基本重复

以下两个文件内容几乎一致：

- `config/boards/arm/zitaotech_sofle/custom_driver_left/keyboard_backlight.c`
- `config/boards/arm/zitaotech_sofle/custom_driver_right/keyboard_backlight.c`

这说明它们现在是“按半边复制一份”的维护方式。短期没问题，但后续如果要修背光逻辑，需要同步改两边。

### 6.3 这不是纯配置仓，修改时要区分层次

这个仓库同时包含：

- 用户层配置
- 板级硬件定义
- 盾层设备定义
- 自定义输入驱动
- 自定义显示驱动

所以改动前最好先判断你改的是：

- 行为层
- 设备层
- 引脚层
- 编译层

否则很容易在错误层级上排查问题。

## 7. 手动编译命令参考

仓库里已有两条手动编译示例：

```bash
west build -p -b zitaotech_sofle_left -- -DSHIELD="lpm_view;left_bbtrackball"
west build -p -b zitaotech_sofle_right -- -DSHIELD="lpm_view;right_trackpoint"
```

如果要编译设置重置固件，可以在 `SHIELD` 后面追加 `settings_reset`。

## 8. 这份文档会不会影响编译

不会。

原因很简单：

- 这次只新增了一个根目录 `md` 文件
- 没有改 `build.yaml`
- 没有改 `config/`
- 没有改 `.github/workflows/`
- 没有改任何 `CMakeLists.txt`、`Kconfig`、`.conf`、`.dts`、`.keymap`、`.c`

因此它不会进入当前 ZMK 构建图，也不会改变 GitHub Actions 的构建结果。

## 9. 一句话总结

这个仓库已经是一个比较完整的 ZMK 自定义硬件固件工程，核心价值不只在键位，而在于“自定义分体板 + 左轨迹球 + 右 TrackPoint + 自定义低功耗显示 + 自定义灯效/背光逻辑”的整体集成。
