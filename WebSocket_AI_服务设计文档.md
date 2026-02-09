# WebSocket AI 服务设计文档

## 概述

本文档描述了基于 FreeSWITCH `mod_audio_stream` 模块的 WebSocket AI 服务架构设计，该服务支持实时音频流传输、AI 语音交互以及用户侧打断功能。

## 系统架构

### 1. 核心组件

#### 1.1 FreeSWITCH 模块 (mod_audio_stream)
- 负责音频捕获和 WebSocket 连接管理
- 支持全双工音频流传输（呼叫者 ↔ WebSocket 端点）
- 提供音频重采样、缓冲管理和格式转换

#### 1.2 WebSocket 客户端
- 基于 libwsc 实现，符合 RFC-6455 标准
- 支持 TLS/WSS 加密连接
- 自动心跳保活机制

#### 1.3 音频流处理器 (AudioStreamer)
- 管理音频数据的编解码
- 处理服务器返回的音频播放请求
- 支持多种音频格式（raw, wav, mp3, ogg, pcmu, pcma）

## 核心功能

### 2. 双向音频流传输

#### 2.1 上行音频流（呼叫者 → WebSocket）
- **采样率**：支持 8kHz、16kHz、24kHz、32kHz、48kHz、64kHz
- **音频通道**：
  - `mono`：单声道，仅包含呼叫者音频
  - `mixed`：单声道，混合呼叫者和被叫者音频
  - `stereo`：立体声，呼叫者和被叫者分别在不同声道
- **数据格式**：L16（线性 PCM）原始音频或 base64 编码
- **缓冲机制**：可配置音频数据包大小（默认 20ms，可按 20ms 倍数调整）

#### 2.2 下行音频流（WebSocket → 呼叫者）
服务器通过 WebSocket 发送 JSON 消息，格式如下：

```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 8000,
    "audioData": "base64编码的音频数据"
  }
}
```

**支持的音频类型**：
- `raw`：原始 PCM 数据（需指定采样率）
- `wav`：WAV 格式
- `mp3`：MP3 格式
- `ogg`：Ogg Vorbis 格式
- `pcmu`：G.711 μ-law
- `pcma`：G.711 A-law

### 3. 音频播放管理

#### 3.1 播放流程
1. WebSocket 服务器发送包含 base64 编码音频的 JSON 消息
2. `AudioStreamer` 解析消息并验证音频格式
3. 解码 base64 数据并写入临时文件
4. 触发 `mod_audio_stream::play` 事件，携带文件路径
5. FreeSWITCH 播放音频文件给呼叫者
6. 播放完成后，临时文件在会话结束时自动删除

#### 3.2 播放控制命令
- **暂停播放**：`uuid_audio_stream <uuid> pause`
- **恢复播放**：`uuid_audio_stream <uuid> resume`
- **停止流传输**：`uuid_audio_stream <uuid> stop [metadata]`

### 4. 用户侧打断功能

#### 4.1 功能描述
在 AI 语音播放过程中，用户可以通过说话打断服务端的语音播放。系统检测到用户发言后：
1. 立即中断当前正在播放的 AI 语音
2. 开始采集用户的语音输入
3. 将用户语音实时传输到 WebSocket 服务器
4. 服务器端重新生成回复内容
5. 继续播放新的 AI 回复

#### 4.2 实现机制

##### 4.2.1 音频活动检测（VAD）
WebSocket 服务器端需要实现语音活动检测（Voice Activity Detection）：
- 实时分析上行音频流，检测用户是否开始说话
- 当检测到语音活动时，发送打断信号

##### 4.2.2 播放中断流程

**服务器端发送中断信号**：
```json
{
  "type": "interruptPlayback",
  "reason": "user_speaking"
}
```

**FreeSWITCH 处理流程**：
1. 接收到 `interruptPlayback` 消息后，调用 `uuid_break <uuid>` 中断当前播放
2. 或使用 `uuid_audio_stream <uuid> pause` 暂停音频流
3. 继续采集用户语音并发送到服务器

##### 4.2.3 重新生成回复流程

**用户说话完毕检测**：
服务器端通过 VAD 检测到用户停止说话后：

```json
{
  "type": "processingComplete",
  "intent": "regenerate_response"
}
```

**发送新的回复音频**：
```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 16000,
    "audioData": "新生成的回复音频的base64编码",
    "sequence": 1,
    "isInterrupted": true
  }
}
```

#### 4.3 状态管理

系统维护以下状态：
- **IDLE**：空闲状态，等待用户或服务器输入
- **PLAYING**：正在播放 AI 回复
- **LISTENING**：正在采集用户语音
- **INTERRUPTED**：播放被用户打断
- **PROCESSING**：服务器正在处理用户输入并生成回复

状态转换流程：
```
IDLE → PLAYING → INTERRUPTED → LISTENING → PROCESSING → PLAYING
                    ↓
                  IDLE (用户未说话)
```

### 5. 事件机制

系统生成以下 FreeSWITCH 事件：

| 事件类型 | 事件名称 | 说明 |
|---------|---------|------|
| 连接成功 | `mod_audio_stream::connect` | WebSocket 连接建立 |
| 断开连接 | `mod_audio_stream::disconnect` | WebSocket 连接关闭 |
| 连接错误 | `mod_audio_stream::error` | 连接或协议错误 |
| JSON 消息 | `mod_audio_stream::json` | 收到 WebSocket 服务器响应 |
| 播放音频 | `mod_audio_stream::play` | 开始播放音频文件 |

### 6. 安全与性能

#### 6.1 安全特性
- **TLS/WSS 支持**：加密 WebSocket 连接
- **证书验证**：支持自定义 CA 证书、客户端证书和密钥
- **主机名验证**：可配置是否验证服务器证书主机名
- **UTF-8 验证**：所有文本消息必须是有效的 UTF-8 编码

#### 6.2 性能优化
- **压缩**：支持 per-message-deflate 压缩（默认启用）
- **缓冲管理**：可配置音频缓冲大小，减少网络传输次数
- **资源管理**：
  - 使用 RAII 模式管理资源
  - 线程安全的音频流处理
  - 临时文件自动清理
- **并发限制**：社区版支持最多 10 个并发流通道

#### 6.3 高并发支持
商业版经过测试，支持：
- 5000+ 并发呼叫
- 正确的会话生命周期管理
- 线程安全的音频注入和关闭
- 有界且可预测的内存使用

## 使用示例

### 7.1 启动音频流传输

```bash
# 启动单声道 16kHz 音频流
uuid_audio_stream <uuid> start wss://ai.example.com/stream mono 16k

# 启动立体声 8kHz 音频流并发送元数据
uuid_audio_stream <uuid> start wss://ai.example.com/stream stereo 8k '{"user":"alice","lang":"zh-CN"}'
```

### 7.2 实现打断功能的示例流程

#### FreeSWITCH 拨号计划示例：
```xml
<extension name="ai_assistant">
  <condition field="destination_number" expression="^9999$">
    <action application="answer"/>
    <action application="set" data="STREAM_BUFFER_SIZE=60"/>
    <action application="uuid_audio_stream"
            data="${uuid} start wss://ai.example.com/assistant mono 16k"/>
    <action application="park"/>
  </condition>
</extension>
```

#### WebSocket 服务器伪代码：
```python
async def handle_audio_stream(websocket):
    vad = VoiceActivityDetector()
    current_state = "IDLE"

    async for message in websocket:
        if message.type == "binary":  # 音频数据
            audio_data = message.data

            # VAD 检测
            if current_state == "PLAYING":
                if vad.detect_speech(audio_data):
                    # 检测到用户说话，发送打断信号
                    await websocket.send(json.dumps({
                        "type": "interruptPlayback",
                        "reason": "user_speaking"
                    }))
                    current_state = "INTERRUPTED"

            # 处理用户语音
            if current_state in ["LISTENING", "INTERRUPTED"]:
                process_user_audio(audio_data)

                if vad.is_speech_ended(audio_data):
                    # 用户说话结束，生成回复
                    current_state = "PROCESSING"
                    response_audio = generate_ai_response()

                    # 发送新的回复
                    await websocket.send(json.dumps({
                        "type": "streamAudio",
                        "data": {
                            "audioDataType": "raw",
                            "sampleRate": 16000,
                            "audioData": base64.b64encode(response_audio).decode()
                        }
                    }))
                    current_state = "PLAYING"
```

### 7.3 发送文本消息

```bash
# 向 WebSocket 服务器发送文本消息
uuid_audio_stream <uuid> send_text '{"action":"get_weather","city":"Beijing"}'
```

### 7.4 停止音频流

```bash
# 停止音频流并发送结束消息
uuid_audio_stream <uuid> stop '{"reason":"user_hangup"}'
```

## 配置参数

### 8. 通道变量

| 变量名 | 说明 | 默认值 |
|-------|------|--------|
| `STREAM_MESSAGE_DEFLATE` | 禁用 per-message-deflate 压缩 | off |
| `STREAM_HEART_BEAT` | 心跳间隔（秒） | off |
| `STREAM_SUPPRESS_LOG` | 抑制日志输出 | off |
| `STREAM_BUFFER_SIZE` | 缓冲时长（毫秒，必须是 20 的倍数） | 20 |
| `STREAM_EXTRA_HEADERS` | 额外的 HTTP 头（JSON 格式） | none |
| `STREAM_TLS_CA_FILE` | CA 证书文件路径 | SYSTEM |
| `STREAM_TLS_KEY_FILE` | 客户端密钥文件 | none |
| `STREAM_TLS_CERT_FILE` | 客户端证书文件 | none |
| `STREAM_TLS_DISABLE_HOSTNAME_VALIDATION` | 禁用主机名验证 | false |

## 最佳实践

### 9.1 打断功能优化建议

1. **VAD 参数调优**：
   - 降低 VAD 灵敏度阈值，避免误触发
   - 设置合理的静音检测时长（建议 300-500ms）

2. **延迟优化**：
   - 使用较小的音频缓冲（20-40ms）以减少延迟
   - 选择低延迟的音频编解码器（如 raw PCM）

3. **用户体验**：
   - 在 AI 回复的自然停顿点更容易被打断
   - 提供音频反馈提示用户打断成功（如"叮"声）

4. **错误处理**：
   - 处理网络抖动导致的误打断
   - 实现打断次数限制，防止循环打断

### 9.2 资源管理

- 设置合理的并发通道限制
- 监控临时文件磁盘使用
- 实现会话超时自动清理机制

## 技术细节

### 10.1 音频文件管理

- 临时文件存储在 `SWITCH_GLOBAL_dirs.temp_dir` 目录
- 文件命名格式：`<session_id>_<index>.tmp.<extension>`
- 会话结束时自动删除所有临时文件

### 10.2 线程安全

- 使用 `std::mutex` 保护共享状态
- 使用 `std::atomic` 标记清理状态
- 使用 `std::weak_ptr` 避免循环引用

### 10.3 生命周期管理

- WebSocket 连接与 FreeSWITCH 会话绑定
- 会话关闭时自动清理 WebSocket 连接和临时文件
- 支持优雅关闭（发送最终消息后断开）

## 故障排查

### 11.1 常见问题

**问题**：打断功能不工作
- **检查**：确认 WebSocket 服务器实现了 VAD 和打断逻辑
- **检查**：验证上行音频流是否正常（检查 `mono`/`stereo` 配置）
- **检查**：查看 FreeSWITCH 日志中的 `mod_audio_stream::json` 事件

**问题**：音频播放延迟高
- **调整**：减小 `STREAM_BUFFER_SIZE` 值（如 20ms 或 40ms）
- **调整**：使用 `raw` 音频格式代替 `mp3`/`ogg`
- **检查**：网络延迟和带宽

**问题**：频繁断开连接
- **启用**：`STREAM_HEART_BEAT`（建议 30 秒）
- **检查**：负载均衡器的空闲超时设置
- **检查**：TLS 证书配置

### 11.2 调试建议

1. 禁用日志抑制：不设置 `STREAM_SUPPRESS_LOG`
2. 查看 FreeSWITCH 日志：`fs_cli -x "console loglevel debug"`
3. 使用 Wireshark 抓包分析 WebSocket 流量
4. 监控事件：`fs_cli -x "event plain mod_audio_stream::*"`

## 版本信息

- **社区版**：免费使用，限制 10 个并发通道
- **商业版**：无并发限制，支持 5000+ 并发呼叫，提供源代码访问

更多信息请联系：[amsoftswitch@gmail.com](mailto:amsoftswitch@gmail.com)

## 参考资料

- [RFC 6455 - WebSocket 协议](https://tools.ietf.org/html/rfc6455)
- [FreeSWITCH 官方文档](https://freeswitch.org/confluence/)
- [libwsc - WebSocket 客户端库](https://github.com/amigniter/libwsc)
