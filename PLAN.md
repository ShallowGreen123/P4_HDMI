# T5-P4 板级替换方案（保持当前调试态行为）

## 摘要
- 以 `Git HEAD` 为实现基线，彻底移除 `espressif/esp32_p4_function_ev_board` 依赖，让工程只使用本地 T5-P4 板级层。
- 保持当前行为不变：`800x600` HDMI、`RGB888/BGR`、外部分配解码缓冲、`video-only` 默认值、现有 LT8912B 诊断流程不回退到早期 `1280x720 + audio`。
- 目标不是重写应用层，而是让现有 `bsp/esp-bsp.h` / `bsp_*` 调用继续工作，只把底层提供者切到 T5-P4。

## 关键变更
- 构建与依赖清理：
  - 从 [main/idf_component.yml](d:/dgx/code/0_lilygo/AI_HDMI/hdmi_video_renderer/main/idf_component.yml) 移除 `espressif/esp32_p4_function_ev_board`。
  - 从 [CMakeLists.txt](d:/dgx/code/0_lilygo/AI_HDMI/hdmi_video_renderer/CMakeLists.txt) 删除仅对旧 BSP 生效的 `idf_component_get_property(...esp32_p4_function_ev_board...)` 抑警逻辑。
  - 重新解析依赖并提交新的 `dependencies.lock`，验收标准是依赖图里不再出现旧 BSP，也不再下载它的转接组件。
- T5-P4 BSP 自洽化：
  - 保留 `bsp/esp-bsp.h` 兼容入口，不改应用层 include 和 `bsp_display_new` / `bsp_sdcard_mount` / `bsp_audio_codec_speaker_init` 等调用面。
  - 在 [t5_p4_board.c](d:/dgx/code/0_lilygo/AI_HDMI/hdmi_video_renderer/components/t5_p4_board/t5_p4_board.c) 为 `CONFIG_BSP_LCD_LT8912B_TEST_PATTERN` 增加和 `PN_SWAP/LANE_SWAP` 一样的兜底定义，避免未显式配置时编译失败。
  - 保持本地 `BSP_*` Kconfig 符号名不变，但确保只剩一套定义，消除当前 duplicate symbol / duplicate choice 告警。
- 当前行为对齐：
  - 让项目配置文件与当前代码行为一致，显式落到 `HDMI 800x600 + RGB888 + HDMI enabled`，不再保留 `sdkconfig` 仍写 `1280x720` 的错位状态。
  - 保持 `ENABLE_AUDIO_PLAYBACK=0` 和 `app_stream_adapter_set_file(..., false)` 这套 video-only 默认路径，不恢复默认音频播放。
  - 更新 README 中仍然写着“默认 1280x720 / audio support”的表述，改成与当前 T5-P4 调试态一致；如果保留音频相关说明，应明确为“代码能力存在，但默认关闭”。

## 公共接口与类型
- 应用层公共接口不改名：
  - 继续使用 `bsp/esp-bsp.h`
  - 继续使用现有 `bsp_*` 函数和 `BSP_*` 宏
- 板级实现来源改为本地 T5-P4 组件，不再混用 managed EV Board BSP。
- 不引入新的应用层 API、类型名或调用顺序变化。

## 测试与验收
- `idf.py reconfigure` / `idf.py build` 不再出现 `BSP_*` 重复定义、choice 冲突、旧 BSP Kconfig 混入。
- 构建产物的依赖求解结果里不再包含 `espressif/esp32_p4_function_ev_board`。
- T5-P4 BSP 单独可编译：`CONFIG_BSP_LCD_LT8912B_TEST_PATTERN` 未配置时也能过编译，显式开启时也能正常编译并打印诊断值。
- 上板启动日志保持当前目标行为：
  - 使用 `t5_p4_board` 初始化
  - I2C / PCA9535 / LT8912B 走 T5-P4 引脚
  - 显示输出为 `800x600`
  - stream adapter 仍以 `video-only` 模式启动
  - SD 卡挂载和循环播放流程不变

## 假设
- 实施时以 `Git HEAD` 内容恢复为干净基线后再改；当前工作区里缺失/漂移的已跟踪文件不作为产品变更的一部分。
- 本次任务以“替换板级实现并保持当前调试态”为准，不顺带恢复早期 `1280x720 + audio` 默认行为。
