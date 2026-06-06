# Stackchan-HtSz: M5Stack Core S3 Stack-chan 个性化增强固件

基于 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) 的 M5Stack Core S3 Stack-chan 增强固件。在原版语音交互基础上增加了触摸、体感、情绪灯、舵机等交互能力，侧重**陪伴感**体验。

> 搓了五天，来回烧录测试了小几百次。分享版抹掉了个性化设置（人设、文案池等），大家实际使用时自定义内容依然很多。不至于像我一样花五天，但建议预留至少半天加载个性化信息并测试。

## 更新日志

### 2026-06-06
- **唤醒词灵敏度重构**：`CONFIG_CUSTOM_WAKE_WORD_THRESHOLD` 改为 `CONFIG_WAKE_WORD_SENSITIVITY` Low/Medium/High 三档，AFE/ESP/自定义三种唤醒方式统一采用
- **文本打断优化**：`SendUserText` 在 Speaking 状态下打断说话并重新唤醒，Listening 状态下关闭音频通道后重新唤醒
- **M5Stack Core S3 休眠背光**：进入休眠时背光亮度改为 0，彻底熄灭
- **省电定时器**：绕过 NVS `sleep_mode` 设置和 `CanEnterSleepMode` 检查，始终启用
- **SPIRAM 模式**：esp32s3 从 OCT 改回 QUAD

## 功能一览

### 头顶触摸 (SI12T)
- 3区电容触摸，摸头触发对话
- 7条随机触摸回应文本（可自定义）
- 5秒触发冷却
- 抗EMI：0xCC灵敏度 + 12秒FTC校准等待（基于 M5Stack BSP 参考实现和 TS12 datasheet）

### 体感检测 (BMI270)
- **摇晃检测**：摇一摇触发互动
- **抱起检测**：拿起来触发互动
- 5分钟全局冷却，armed/disarmed 状态机
- 自定义 BMI270 驱动（地址 0x69，绕过 SDK 默认 0x68）

### 屏幕触摸 (FT6336)
- 双击、上下左右滑动、长按 6 种手势
- 60 条随机动作文本（可自定义）

### WS2812 情绪灯环
- 12颗 LED，通过 PY32 IO Expander (0x6F) 控制
- 21种情绪对应颜色映射，跟表情同步变化
- MCP 工具支持：`self.led.set_color`、`self.led.turn_off`、`self.led.auto`

### 舵机 (SCS 总线)
- 摄像头人脸追踪 (GC0308)
- 空闲扫视动画
- 对话时暂停/恢复

### 文本打断
- LLM/用户输入文本消息可在 Speaking/Listening 状态下打断当前对话并重新唤醒
- 避免对话中发文本被静默丢弃

### 早安问候 (定时任务)
- 工作日早上定时问候 + 天气查询
- SNTP 延迟启动（避免 tcpip panic）

### 自定义唤醒词
- Multinet6 自定义唤醒词支持
- 通过 `sdkconfig.defaults` 配置

### 其他
- I2C 错误容错处理（防止偶发超时导致整机重启）
- 摄像头拍照（MCP 工具）
- 电池监测 + 低电量提醒
- 休眠时自动进入省电模式，背光彻底熄灭
- 省电定时器始终启用，不再受 NVS 设置影响

## 已知问题和注意事项

1. **负载过大时可能无法开机**：电池供电 + 灯全亮时偶尔出现。已改为对话才亮灯，但不排除仍可能发生。解决办法：插 USB 再开机。

2. **LLM 控灯偶尔卡住**：让 LLM 调灯时有概率卡住。可以加看门狗，目前未实现。

3. **LLM 口头说调灯但实际不动**：prompt 里加了强制要求但不保证每次生效。

4. **自定义灯光后需要物理重启**：烧录后需要拔 USB + 关机，等 30 秒后再插 USB 重启。尝试过用软件方式免除但未成功。换色不频繁的话影响不大。

5. **唤醒词灵敏度需自行调整**：默认阈值可能在易误触发和需要大声喊之间摇摆，请在 `sdkconfig.defaults` 里调整 `CONFIG_WAKE_WORD_SENSITIVITY`（Low/Medium/High 三档）找到适合自己环境的值。另外灵敏度和唤醒词本身关系很大——生僻词/自定义人名的识别率天然低于"你好小智"这类常见词组，需要设成 High（更灵敏）才能触发，但相应误触发概率也会增加。

6. **自部署小智服务端的朋友注意安全配置**：
① 服务端的 websocket 和 http 端口不要直接暴露公网，用防火墙限制访问来源
② config 里的 auth.enabled 一定要改成 true，加上你设备的 MAC 白名单
③ TTS、LLM 这些第三方 API Key 不要写在配置文件里暴露着，端口一旦被扫到配置就全泄露了
④ 小智有摄像头调用能力，端口裸奔等于把摄像头开放给任何人
⑤ 公网端口扫描是常态，部署完记得 ss -tlnp 检查一下自己开了哪些端口。安全无小事。

## 编译

需要 [ESP-IDF v5.5.x](https://github.com/espressif/esp-idf)。

```bash
# 克隆
git clone https://github.com/mo-hantang/Stackchan-HtSz.git
cd Stackchan-HtSz

# 编译（低配机器用 -j1 避免内存不足）
idf.py build -- -j1
```

## 烧录

```bash
# 全量烧录（首次或 OTA 出问题时使用）
idf.py flash

# 或手动指定：
python -m esptool --chip esp32s3 -b 460800 \
  --before default_reset --after hard_reset \
  write_flash --flash_mode dio --flash_size 16MB --flash_freq 80m \
  0x0 build/bootloader/bootloader.bin \
  0x8000 build/partition_table/partition-table.bin \
  0xd000 build/ota_data_initial.bin \
  0x410000 build/xiaozhi.bin
```

> **注意**：如果之前用过 OTA 升级，务必同时刷入 `ota_data_initial.bin`（地址 0xd000），否则设备可能从旧分区启动。

## 配置

编译前修改 `sdkconfig.defaults`：

```
CONFIG_BOARD_TYPE_M5STACK_CORE_S3=y
CONFIG_USE_CUSTOM_WAKE_WORD=y
CONFIG_CUSTOM_WAKE_WORD="ni hao xiao zhi"
CONFIG_CUSTOM_WAKE_WORD_DISPLAY="你好小智"
CONFIG_WAKE_WORD_SENSITIVITY_MEDIUM=y
CONFIG_OTA_URL="http://你的服务器IP:8003/xiaozhi/ota/"
```

## 服务端

本固件配合 [xiaozhi-esp32-server](https://github.com/78/xiaozhi-esp32-server) 使用。需要在服务端配置 LLM 和 TTS。

## 自定义

`m5stack_core_s3.cc` 中的触摸文本、手势动作、早安问候等使用通用占位符（主人/小智）。替换成你自己的人设文本即可。

主要自定义点：
- **触摸回应**：搜索 `msgs[]` 数组
- **手势动作池**：搜索 `DoubleClickPool`、`UpSwipePool` 等函数
- **早安问候**：搜索 `MorningLoop`
- **情绪灯颜色**：搜索 `UpdateLedsFromEmotion`

## 致谢

- [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) — 基础固件
- [M5Stack StackChan-BSP](https://github.com/m5stack/StackChan-BSP) — SI12T 驱动参考
- [TS12 Datasheet](http://file2.dzsc.com/product/18/09/06/1114361_130715119.pdf) — 触摸传感器寄存器文档

## License

同上游 [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32)。
