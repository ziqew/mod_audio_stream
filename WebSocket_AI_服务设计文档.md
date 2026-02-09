
# WebSocket AI 对话服务完整设计文档

## 文档信息

- **文档版本**: 2.0.0
- **创建日期**: 2026-02-06
- **最后更新**: 2026-02-09
- **作者**: AI Service Team
- **更新内容**:
  - 合并 FreeSWITCH mod_audio_stream 技术文档和 Python WebSocket 服务文档
  - 添加术语表
  - 优化文档结构，消除重复内容
  - 完善用户打断功能文档
  - 添加版本历史记录

---

## 概述

本设计文档描述了一个完整的 WebSocket AI 对话服务系统，该系统集成 FreeSWITCH `mod_audio_stream` 模块，实现端到端的实时语音对话 AI 解决方案。系统包含语音识别(STT)、大语言模型(LLM)和语音合成(TTS)三个核心组件，默认使用通义千问(Qwen)的服务。

### 核心特性

- **实时音频处理**: 支持全双工音频流传输和处理
- **模块化设计**: STT、LLM、TTS 可独立配置和替换
- **异步架构**: 基于 asyncio，高并发支持
- **用户打断功能**: 支持实时检测用户发言并打断 AI 播放
- **VAD 支持**: 智能语音端点检测
- **错误恢复**: 完善的异常处理和重连机制
- **配置灵活**: 支持环境变量和配置文件
- **日志完善**: 详细的调试和监控日志

---

## 目录

1. [系统架构](#1-系统架构)
   - 1.1 [整体架构](#11-整体架构图)
   - 1.2 [核心组件](#12-核心组件)
2. [FreeSWITCH 集成 (mod_audio_stream)](#2-freeswitch-集成-mod_audio_stream)
   - 2.1 [模块概述](#21-模块概述)
   - 2.2 [双向音频流传输](#22-双向音频流传输)
   - 2.3 [音频播放管理](#23-音频播放管理)
   - 2.4 [用户打断功能](#24-用户打断功能)
   - 2.5 [事件机制](#25-事件机制)
   - 2.6 [安全与性能](#26-安全与性能)
   - 2.7 [配置参数](#27-配置参数)
   - 2.8 [FreeSWITCH 拨号计划](#28-freeswitch-拨号计划)
3. [技术规范](#3-技术规范)
   - 3.1 [音频格式规范](#31-音频格式规范)
   - 3.2 [WebSocket 协议](#32-websocket-协议)
   - 3.3 [技术栈](#33-技术栈)
4. [核心组件设计](#4-核心组件设计)
   - 4.1 [WebSocket 服务器](#41-websocket-服务器)
   - 4.2 [会话处理器](#42-会话处理器)
   - 4.3 [STT 适配器](#43-stt-适配器)
   - 4.4 [LLM 适配器](#44-llm-适配器)
   - 4.5 [TTS 适配器](#45-tts-适配器)
   - 4.6 [音频处理模块](#46-音频处理模块)
5. [数据流设计](#5-数据流设计)
   - 5.1 [完整对话流程](#51-完整对话流程)
   - 5.2 [序列图](#52-序列图)
6. [接口设计](#6-接口设计)
   - 6.1 [配置接口](#61-配置接口)
   - 6.2 [API 端点](#62-api-端点)
7. [配置管理](#7-配置管理)
   - 7.1 [环境变量配置](#71-环境变量配置)
   - 7.2 [YAML 配置文件](#72-yaml-配置文件)
8. [实现代码](#8-实现代码)
   - 8.1 [项目结构](#81-项目结构)
   - 8.2 [核心实现代码](#82-核心实现代码)
   - 8.3 [requirements.txt](#83-requirementstxt)
9. [部署指南](#9-部署指南)
   - 9.1 [本地开发部署](#91-本地开发部署)
   - 9.2 [Docker 部署](#92-docker-部署)
   - 9.3 [生产环境部署](#93-生产环境部署)
10. [测试方案](#10-测试方案)
    - 10.1 [单元测试](#101-单元测试)
    - 10.2 [集成测试](#102-集成测试)
    - 10.3 [WebSocket 客户端测试](#103-websocket-客户端测试)
11. [最佳实践](#11-最佳实践)
    - 11.1 [性能优化](#111-性能优化)
    - 11.2 [错误处理](#112-错误处理)
    - 11.3 [监控和日志](#113-监控和日志)
    - 11.4 [安全性](#114-安全性)
    - 11.5 [用户打断功能优化](#115-用户打断功能优化)
12. [故障排查](#12-故障排查)
    - 12.1 [常见问题](#121-常见问题)
    - 12.2 [调试建议](#122-调试建议)
13. [术语表](#13-术语表)
14. [附录A: TTS 音频流传输详解](#附录a-tts-音频流传输详解)
15. [版本历史](#版本历史)
16. [参考资料](#参考资料)

---

## 1. 系统架构

### 1.1 整体架构图

```
┌─────────────────┐
│   FreeSWITCH    │
│ mod_audio_stream│
└────────┬────────┘
         │ WebSocket (L16 PCM Audio)
         │
┌────────▼────────────────────────────────────────┐
│          WebSocket AI 服务                       │
│  ┌──────────────────────────────────────────┐  │
│  │         WebSocket Server                  │  │
│  │    (asyncio + websockets)                │  │
│  └──────────────┬───────────────────────────┘  │
│                 │                               │
│  ┌──────────────▼───────────────────────────┐  │
│  │      音频缓冲与处理模块                    │  │
│  │  - 音频流接收                              │  │
│  │  - VAD (语音活动检测)                      │  │
│  │  - 音频格式转换                            │  │
│  └──────────────┬───────────────────────────┘  │
│                 │                               │
│  ┌──────────────▼───────────────────────────┐  │
│  │      STT 服务适配器                        │  │
│  │  - Qwen STT (默认)                        │  │
│  │  - Google STT                             │  │
│  │  - Azure STT                              │  │
│  └──────────────┬───────────────────────────┘  │
│                 │ Text                          │
│  ┌──────────────▼───────────────────────────┐  │
│  │      LLM 服务适配器                        │  │
│  │  - Qwen LLM (默认)                        │  │
│  │  - OpenAI GPT                             │  │
│  │  - Claude                                 │  │
│  └──────────────┬───────────────────────────┘  │
│                 │ Response Text                 │
│  ┌──────────────▼───────────────────────────┐  │
│  │      TTS 服务适配器                        │  │
│  │  - Qwen TTS (默认)                        │  │
│  │  - Azure TTS                              │  │
│  │  - Google TTS                             │  │
│  └──────────────┬───────────────────────────┘  │
│                 │ Audio Stream                  │
│  ┌──────────────▼───────────────────────────┐  │
│  │      音频编码与发送模块                    │  │
│  │  - Base64 编码                            │  │
│  │  - 格式封装 (streamAudio)                 │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 1.2 核心组件

#### 1.2.1 FreeSWITCH 模块 (mod_audio_stream)

- 负责音频捕获和 WebSocket 连接管理
- 支持全双工音频流传输（呼叫者 ↔ WebSocket 端点）
- 提供音频重采样、缓冲管理和格式转换

#### 1.2.2 WebSocket 客户端

- 基于 libwsc 实现，符合 RFC-6455 标准
- 支持 TLS/WSS 加密连接
- 自动心跳保活机制

#### 1.2.3 音频流处理器 (AudioStreamer)

- 管理音频数据的编解码
- 处理服务器返回的音频播放请求
- 支持多种音频格式（raw, wav, mp3, ogg, pcmu, pcma）

---

## 2. FreeSWITCH 集成 (mod_audio_stream)

### 2.1 模块概述

`mod_audio_stream` 是 FreeSWITCH 的音频流传输模块，它通过 WebSocket 连接将音频流实时传输到远程服务器，并接收服务器返回的音频进行播放。该模块特别适用于实时 AI 语音交互场景。

### 2.2 双向音频流传输

#### 2.2.1 上行音频流（呼叫者 → WebSocket）

- **采样率**: 支持 8kHz、16kHz、24kHz、32kHz、48kHz、64kHz
- **音频通道**:
  - `mono`: 单声道，仅包含呼叫者音频
  - `mixed`: 单声道，混合呼叫者和被叫者音频
  - `stereo`: 立体声，呼叫者和被叫者分别在不同声道
- **数据格式**: L16（线性 PCM）原始音频或 base64 编码
- **缓冲机制**: 可配置音频数据包大小（默认 20ms，可按 20ms 倍数调整）

#### 2.2.2 下行音频流（WebSocket → 呼叫者）

服务器通过 WebSocket 发送 JSON 消息，格式如下：

```json
{
  "type": "streamAudio",
  "data": {
    "audioDataType": "raw",
    "sampleRate": 8000,
    "audioData": "base64_encoded_pcm_data"
  }
}
```

**支持的 audioDataType**:
- `raw` - 原始 PCM 数据（需指定 sampleRate）
- `wav` - WAV 文件格式
- `mp3` - MP3 压缩格式
- `ogg` - OGG Vorbis 格式
- `pcmu` - G.711 μ-law
- `pcma` - G.711 A-law

### 2.3 音频播放管理

#### 2.3.1 播放流程

1. WebSocket 服务器发送包含 base64 编码音频的 JSON 消息
2. `AudioStreamer` 解析消息并验证音频格式
3. 解码 base64 数据并写入临时文件
4. 触发 `mod_audio_stream::play` 事件，携带文件路径
5. FreeSWITCH 播放音频文件给呼叫者
6. 播放完成后，临时文件在会话结束时自动删除

#### 2.3.2 播放控制命令

```bash
# 暂停播放
uuid_audio_stream <uuid> pause

# 恢复播放
uuid_audio_stream <uuid> resume

# 停止流传输
uuid_audio_stream <uuid> stop [metadata]
```

### 2.4 用户打断功能

#### 2.4.1 功能描述

在 AI 语音播放过程中，用户可以通过说话打断服务端的语音播放。系统检测到用户发言后：

1. 立即中断当前正在播放的 AI 语音
2. 开始采集用户的语音输入
3. 将用户语音实时传输到 WebSocket 服务器
4. 服务器端重新生成回复内容
5. 继续播放新的 AI 回复

#### 2.4.2 实现机制

##### 音频活动检测（VAD）

WebSocket 服务器端需要实现语音活动检测（Voice Activity Detection）：
- 实时分析上行音频流，检测用户是否开始说话
- 当检测到语音活动时，发送打断信号

##### 播放中断流程

**服务器端发送中断信号**:

```json
{
  "type": "interruptPlayback",
  "reason": "user_speaking"
}
```

**FreeSWITCH 处理流程**:

1. 接收到 `interruptPlayback` 消息后，调用 `uuid_break <uuid>` 中断当前播放
2. 或使用 `uuid_audio_stream <uuid> pause` 暂停音频流
3. 继续采集用户语音并发送到服务器

##### 重新生成回复流程

**用户说话完毕检测**:

服务器端通过 VAD 检测到用户停止说话后：

```json
{
  "type": "processingComplete",
  "intent": "regenerate_response"
}
```

**发送新的回复音频**:

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

#### 2.4.3 状态管理

系统维护以下状态：

- **IDLE**: 空闲状态，等待用户或服务器输入
- **PLAYING**: 正在播放 AI 回复
- **LISTENING**: 正在采集用户语音
- **INTERRUPTED**: 播放被用户打断
- **PROCESSING**: 服务器正在处理用户输入并生成回复

状态转换流程:

```
IDLE → PLAYING → INTERRUPTED → LISTENING → PROCESSING → PLAYING
                    ↓
                  IDLE (用户未说话)
```

### 2.5 事件机制

系统生成以下 FreeSWITCH 事件：

| 事件类型 | 事件名称 | 说明 |
|---------|---------|------|
| 连接成功 | `mod_audio_stream::connect` | WebSocket 连接建立 |
| 断开连接 | `mod_audio_stream::disconnect` | WebSocket 连接关闭 |
| 连接错误 | `mod_audio_stream::error` | 连接或协议错误 |
| JSON 消息 | `mod_audio_stream::json` | 收到 WebSocket 服务器响应 |
| 播放音频 | `mod_audio_stream::play` | 开始播放音频文件 |

### 2.6 安全与性能

#### 2.6.1 安全特性

- **TLS/WSS 支持**: 加密 WebSocket 连接
- **证书验证**: 支持自定义 CA 证书、客户端证书和密钥
- **主机名验证**: 可配置是否验证服务器证书主机名
- **UTF-8 验证**: 所有文本消息必须是有效的 UTF-8 编码

#### 2.6.2 性能优化

- **压缩**: 支持 per-message-deflate 压缩（默认启用）
- **缓冲管理**: 可配置音频缓冲大小，减少网络传输次数
- **资源管理**:
  - 使用 RAII 模式管理资源
  - 线程安全的音频流处理
  - 临时文件自动清理
- **并发限制**: 社区版支持最多 10 个并发流通道

#### 2.6.3 高并发支持

商业版经过测试，支持：
- 5000+ 并发呼叫
- 正确的会话生命周期管理
- 线程安全的音频注入和关闭
- 有界且可预测的内存使用

### 2.7 配置参数

#### 通道变量

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

### 2.8 FreeSWITCH 拨号计划

#### 基本示例

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

#### 启动音频流传输

```bash
# 启动单声道 16kHz 音频流
uuid_audio_stream <uuid> start wss://ai.example.com/stream mono 16k

# 启动立体声 8kHz 音频流并发送元数据
uuid_audio_stream <uuid> start wss://ai.example.com/stream stereo 8k '{"user":"alice","lang":"zh-CN"}'
```

#### 发送文本消息

```bash
# 向 WebSocket 服务器发送文本消息
uuid_audio_stream <uuid> send_text '{"action":"get_weather","city":"Beijing"}'
```

#### 停止音频流

```bash
# 停止音频流并发送结束消息
uuid_audio_stream <uuid> stop '{"reason":"user_hangup"}'
```

---

## 3. 技术规范

### 3.1 音频格式规范

#### 输入音频格式（从 mod_audio_stream）

| 参数 | 值 |
|------|-----|
| 编码格式 | L16 (16-bit Linear PCM) |
| 字节序 | Big-endian |
| 采样率 | 8000 Hz / 16000 Hz (可配置) |
| 声道 | 1 (mono) / 2 (stereo) |
| 位深度 | 16-bit |
| 数据类型 | Binary |

#### 输出音频格式（发送到 mod_audio_stream）

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

**支持的音频类型**:
- `raw`: 原始 PCM 数据（需指定采样率）
- `wav`: WAV 格式
- `mp3`: MP3 格式
- `ogg`: Ogg Vorbis 格式
- `pcmu`: G.711 μ-law
- `pcma`: G.711 A-law

### 3.2 WebSocket 协议

#### 连接建立

```
ws://hostname:port/stream
wss://hostname:port/stream (SSL)
```

#### 消息类型

**1. 初始元数据（可选）**

```json
{
  "session_id": "unique-session-id",
  "caller": "phone-number",
  "language": "zh-CN"
}
```

**2. 音频数据**

- 类型: Binary
- 格式: L16 PCM
- 大小: 可变（通常 20ms-100ms 音频片段）

**3. 文本消息**

```json
{
  "type": "text",
  "content": "用户输入的文本"
}
```

### 3.3 技术栈

| 组件 | 技术选择 | 版本要求 |
|------|---------|---------|
| Python | CPython | 3.9+ |
| WebSocket Server | websockets | 12.0+ |
| 异步框架 | asyncio | 内置 |
| HTTP 客户端 | aiohttp | 3.9+ |
| 音频处理 | numpy, scipy | latest |
| VAD | webrtcvad | 2.0+ |
| 配置管理 | python-dotenv | 1.0+ |
| 日志 | logging | 内置 |

---

## 4. 核心组件设计

### 4.1 WebSocket 服务器

#### 职责

- 管理 WebSocket 连接
- 接收和解析音频流
- 协调各服务模块
- 发送响应音频

#### 关键类

```python
class AudioStreamServer:
    """WebSocket 音频流服务器"""

    def __init__(self, host: str, port: int, config: dict):
        self.host = host
        self.port = port
        self.config = config
        self.sessions = {}  # session_id -> SessionHandler

    async def start(self):
        """启动服务器"""

    async def handle_connection(self, websocket, path):
        """处理 WebSocket 连接"""

    async def stop(self):
        """停止服务器"""
```

### 4.2 会话处理器

#### 职责

- 管理单个会话的生命周期
- 音频缓冲和 VAD 处理
- 协调 STT → LLM → TTS 流程

#### 关键类

```python
class SessionHandler:
    """会话处理器"""

    def __init__(self, websocket, session_id: str, config: dict):
        self.websocket = websocket
        self.session_id = session_id
        self.config = config

        # 服务适配器
        self.stt_adapter = STTAdapter.create(config)
        self.llm_adapter = LLMAdapter.create(config)
        self.tts_adapter = TTSAdapter.create(config)

        # 音频处理
        self.audio_buffer = AudioBuffer()
        self.vad = VADProcessor()

    async def handle_audio(self, audio_data: bytes):
        """处理接收到的音频数据"""

    async def process_conversation(self, text: str):
        """处理对话流程: LLM -> TTS -> 发送"""
```

### 4.3 STT 适配器

#### 接口设计

```python
class STTAdapter(ABC):
    """STT 服务抽象基类"""

    @abstractmethod
    async def transcribe(self, audio_data: bytes,
                        sample_rate: int,
                        language: str = "zh-CN") -> str:
        """
        转录音频为文本

        参数:
            audio_data: 音频数据 (PCM)
            sample_rate: 采样率
            language: 语言代码

        返回:
            识别的文本
        """
        pass
```

#### Qwen STT 实现

```python
class QwenSTTAdapter(STTAdapter):
    """通义千问 STT 适配器"""

    def __init__(self, api_key: str, endpoint: str):
        self.api_key = api_key
        self.endpoint = endpoint
        self.session = aiohttp.ClientSession()

    async def transcribe(self, audio_data: bytes,
                        sample_rate: int,
                        language: str = "zh-CN") -> str:
        """使用 Qwen API 进行语音识别"""

        # 转换音频格式
        wav_data = self._convert_to_wav(audio_data, sample_rate)

        # 调用 Qwen API
        url = f"{self.endpoint}/audio/transcriptions"

        form = aiohttp.FormData()
        form.add_field('file', wav_data,
                      filename='audio.wav',
                      content_type='audio/wav')
        form.add_field('model', 'qwen-audio-turbo')
        form.add_field('language', language)

        headers = {
            'Authorization': f'Bearer {self.api_key}'
        }

        async with self.session.post(url, data=form, headers=headers) as resp:
            result = await resp.json()
            return result.get('text', '')
```

### 4.4 LLM 适配器

#### 接口设计

```python
class LLMAdapter(ABC):
    """LLM 服务抽象基类"""

    @abstractmethod
    async def chat(self, messages: List[dict],
                   system_prompt: str = None) -> str:
        """
        对话生成

        参数:
            messages: 对话历史 [{"role": "user", "content": "..."}]
            system_prompt: 系统提示词

        返回:
            AI 回复
        """
        pass
```

#### Qwen LLM 实现

```python
class QwenLLMAdapter(LLMAdapter):
    """通义千问 LLM 适配器"""

    def __init__(self, api_key: str, model: str = "qwen-turbo"):
        self.api_key = api_key
        self.model = model
        self.session = aiohttp.ClientSession()
        self.endpoint = "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation"

    async def chat(self, messages: List[dict],
                   system_prompt: str = None) -> str:
        """使用 Qwen API 进行对话"""

        # 构建请求
        payload = {
            "model": self.model,
            "input": {
                "messages": messages
            },
            "parameters": {
                "result_format": "message"
            }
        }

        if system_prompt:
            payload["input"]["messages"].insert(0, {
                "role": "system",
                "content": system_prompt
            })

        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

        async with self.session.post(self.endpoint,
                                    json=payload,
                                    headers=headers) as resp:
            result = await resp.json()
            return result['output']['choices'][0]['message']['content']
```

### 4.5 TTS 适配器

#### 接口设计

```python
class TTSAdapter(ABC):
    """TTS 服务抽象基类"""

    @abstractmethod
    async def synthesize(self, text: str,
                        voice: str = "default",
                        sample_rate: int = 8000) -> bytes:
        """
        文本转语音

        参数:
            text: 要合成的文本
            voice: 语音名称
            sample_rate: 采样率

        返回:
            音频数据 (PCM)
        """
        pass
```

#### Qwen TTS 实现

```python
class QwenTTSAdapter(TTSAdapter):
    """通义千问 TTS 适配器"""

    def __init__(self, api_key: str, model: str = "cosyvoice-v1"):
        self.api_key = api_key
        self.model = model
        self.session = aiohttp.ClientSession()
        self.endpoint = "https://dashscope.aliyuncs.com/api/v1/services/audio/tts/synthesis"

    async def synthesize(self, text: str,
                        voice: str = "longxiaochun",
                        sample_rate: int = 8000) -> bytes:
        """使用 Qwen API 进行语音合成"""

        payload = {
            "model": self.model,
            "input": {
                "text": text
            },
            "parameters": {
                "voice": voice,
                "format": "pcm",
                "sample_rate": sample_rate
            }
        }

        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

        async with self.session.post(self.endpoint,
                                    json=payload,
                                    headers=headers) as resp:
            audio_data = await resp.read()
            return audio_data
```

### 4.6 音频处理模块

#### 音频缓冲器

```python
class AudioBuffer:
    """音频缓冲器"""

    def __init__(self, sample_rate: int = 8000):
        self.sample_rate = sample_rate
        self.buffer = bytearray()
        self.min_duration = 0.5  # 最小缓冲时长（秒）

    def append(self, data: bytes):
        """添加音频数据"""
        self.buffer.extend(data)

    def get_duration(self) -> float:
        """获取缓冲区音频时长"""
        # L16 PCM: 2 bytes per sample
        samples = len(self.buffer) // 2
        return samples / self.sample_rate

    def is_ready(self) -> bool:
        """是否有足够的音频可处理"""
        return self.get_duration() >= self.min_duration

    def get_and_clear(self) -> bytes:
        """获取并清空缓冲区"""
        data = bytes(self.buffer)
        self.buffer.clear()
        return data
```

#### VAD 处理器

```python
import webrtcvad

class VADProcessor:
    """语音活动检测"""

    def __init__(self, sample_rate: int = 8000, aggressiveness: int = 2):
        self.vad = webrtcvad.Vad(aggressiveness)
        self.sample_rate = sample_rate
        self.frame_duration = 30  # ms
        self.frame_size = int(sample_rate * self.frame_duration / 1000) * 2

        # 状态管理
        self.is_speaking = False
        self.silence_frames = 0
        self.speech_frames = 0
        self.max_silence_frames = 20  # 600ms 静音后结束

    def process_frame(self, frame: bytes) -> tuple[bool, bool]:
        """
        处理音频帧

        返回:
            (is_speech, speech_ended)
        """
        # 检测是否为语音
        is_speech = self.vad.is_speech(frame, self.sample_rate)

        if is_speech:
            self.speech_frames += 1
            self.silence_frames = 0

            if not self.is_speaking and self.speech_frames >= 3:
                # 开始说话
                self.is_speaking = True
        else:
            self.silence_frames += 1

            if self.is_speaking and self.silence_frames >= self.max_silence_frames:
                # 结束说话
                self.is_speaking = False
                self.speech_frames = 0
                return (False, True)  # 语音结束

        return (is_speech, False)
```

---

## 5. 数据流设计

### 5.1 完整对话流程

```
1. 连接建立
   FreeSWITCH → WebSocket 连接 → AI 服务

2. 音频接收
   mod_audio_stream → Binary Audio (L16 PCM) → AudioBuffer

3. VAD 检测
   AudioBuffer → VAD → 检测到语音结束

4. 语音识别
   音频数据 → STT Adapter → 文本

5. LLM 处理
   文本 → LLM Adapter → AI 回复

6. 语音合成
   AI 回复 → TTS Adapter → 音频数据

7. 音频发送
   音频数据 → Base64 编码 → streamAudio JSON → mod_audio_stream

8. 播放
   mod_audio_stream → FreeSWITCH → 播放给呼叫者
```

### 5.2 序列图

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────┐  ┌─────┐  ┌─────┐
│FreeSWITCH│  │WebSocket │  │ Session  │  │ STT │  │ LLM │  │ TTS │
│          │  │  Server  │  │ Handler  │  │     │  │     │  │     │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └──┬──┘  └──┬──┘  └──┬──┘
     │             │               │           │        │        │
     │  Connect    │               │           │        │        │
     ├────────────>│               │           │        │        │
     │             │ Create        │           │        │        │
     │             ├──────────────>│           │        │        │
     │             │               │           │        │        │
     │ Audio Data  │               │           │        │        │
     ├────────────>│ Binary        │           │        │        │
     │             ├──────────────>│           │        │        │
     │             │               │ Buffer    │        │        │
     │             │               ├───────┐   │        │        │
     │             │               │       │   │        │        │
     │             │               │<──────┘   │        │        │
     │ More Audio  │               │           │        │        │
     ├────────────>│               │           │        │        │
     │             ├──────────────>│           │        │        │
     │             │               │ VAD: End  │        │        │
     │             │               │           │        │        │
     │             │               │ Transcribe│        │        │
     │             │               ├──────────>│        │        │
     │             │               │           │        │        │
     │             │               │<──────────┤        │        │
     │             │               │   Text    │        │        │
     │             │               │           │        │        │
     │             │               │ Chat      │        │        │
     │             │               ├───────────┼───────>│        │
     │             │               │           │        │        │
     │             │               │<──────────┼────────┤        │
     │             │               │         Response   │        │
     │             │               │           │        │        │
     │             │               │ Synthesize│        │        │
     │             │               ├───────────┼────────┼───────>│
     │             │               │           │        │        │
     │             │               │<──────────┼────────┼────────┤
     │             │               │         Audio      │        │
     │             │               │           │        │        │
     │             │  streamAudio  │           │        │        │
     │             │<──────────────┤           │        │        │
     │<────────────┤               │           │        │        │
     │  JSON+Audio │               │           │        │        │
     │             │               │           │        │        │
```

---

## 6. 接口设计

### 6.1 配置接口

```python
@dataclass
class ServiceConfig:
    """服务配置"""

    # 服务器配置
    host: str = "0.0.0.0"
    port: int = 8080
    ssl_cert: Optional[str] = None
    ssl_key: Optional[str] = None

    # STT 配置
    stt_provider: str = "qwen"  # qwen, google, azure
    stt_api_key: str = ""
    stt_endpoint: str = ""
    stt_model: str = "qwen-audio-turbo"
    stt_language: str = "zh-CN"

    # LLM 配置
    llm_provider: str = "qwen"  # qwen, openai, claude
    llm_api_key: str = ""
    llm_model: str = "qwen-turbo"
    llm_system_prompt: str = "你是一个友好的AI助手。"
    llm_temperature: float = 0.7
    llm_max_tokens: int = 2000

    # TTS 配置
    tts_provider: str = "qwen"  # qwen, azure, google
    tts_api_key: str = ""
    tts_endpoint: str = ""
    tts_model: str = "cosyvoice-v1"
    tts_voice: str = "longxiaochun"
    tts_sample_rate: int = 8000

    # 音频配置
    audio_sample_rate: int = 8000
    audio_channels: int = 1
    audio_format: str = "pcm"

    # VAD 配置
    vad_enabled: bool = True
    vad_aggressiveness: int = 2
    vad_min_speech_duration: float = 0.5
    vad_max_silence_duration: float = 0.6

    # 会话配置
    session_timeout: int = 300  # 秒
    max_conversation_turns: int = 50

    # 日志配置
    log_level: str = "INFO"
    log_file: Optional[str] = None
```

### 6.2 API 端点

虽然主要是 WebSocket，但可以提供 HTTP 端点用于健康检查和配置：

```python
# GET /health
# 健康检查
{
  "status": "healthy",
  "uptime": 12345,
  "active_sessions": 5
}

# GET /metrics
# 服务指标
{
  "total_sessions": 100,
  "active_sessions": 5,
  "total_audio_minutes": 245.5,
  "stt_requests": 150,
  "llm_requests": 150,
  "tts_requests": 150
}

# POST /config/reload
# 重新加载配置（热更新）
{
  "success": true,
  "message": "Configuration reloaded"
}
```

---

## 7. 配置管理

### 7.1 环境变量配置

创建 `.env` 文件：

```bash
# 服务器配置
SERVER_HOST=0.0.0.0
SERVER_PORT=8080

# Qwen API 密钥
QWEN_API_KEY=your_qwen_api_key_here

# STT 配置
STT_PROVIDER=qwen
STT_MODEL=qwen-audio-turbo
STT_LANGUAGE=zh-CN

# LLM 配置
LLM_PROVIDER=qwen
LLM_MODEL=qwen-turbo
LLM_SYSTEM_PROMPT=你是一个友好、专业的AI助手。请用简洁明了的语言回答问题。
LLM_TEMPERATURE=0.7
LLM_MAX_TOKENS=2000

# TTS 配置
TTS_PROVIDER=qwen
TTS_MODEL=cosyvoice-v1
TTS_VOICE=longxiaochun
TTS_SAMPLE_RATE=8000

# 音频配置
AUDIO_SAMPLE_RATE=8000
AUDIO_CHANNELS=1

# VAD 配置
VAD_ENABLED=true
VAD_AGGRESSIVENESS=2

# 日志配置
LOG_LEVEL=INFO
LOG_FILE=/var/log/websocket_ai_service.log
```

### 7.2 YAML 配置文件

创建 `config.yaml`:

```yaml
server:
  host: "0.0.0.0"
  port: 8080
  ssl:
    enabled: false
    cert_file: null
    key_file: null

stt:
  provider: "qwen"
  api_key: "${QWEN_API_KEY}"
  endpoint: "https://dashscope.aliyuncs.com/api/v1/services/audio/asr"
  model: "qwen-audio-turbo"
  language: "zh-CN"

llm:
  provider: "qwen"
  api_key: "${QWEN_API_KEY}"
  model: "qwen-turbo"
  system_prompt: "你是一个友好、专业的AI助手。"
  parameters:
    temperature: 0.7
    max_tokens: 2000
    top_p: 0.8

tts:
  provider: "qwen"
  api_key: "${QWEN_API_KEY}"
  endpoint: "https://dashscope.aliyuncs.com/api/v1/services/audio/tts"
  model: "cosyvoice-v1"
  voice: "longxiaochun"
  sample_rate: 8000

audio:
  sample_rate: 8000
  channels: 1
  format: "pcm"
  buffer_duration: 0.5

vad:
  enabled: true
  aggressiveness: 2
  min_speech_duration: 0.5
  max_silence_duration: 0.6

session:
  timeout: 300
  max_conversation_turns: 50

logging:
  level: "INFO"
  format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  file: "/var/log/websocket_ai_service.log"
```

---

## 8. 实现代码

### 8.1 项目结构

```
websocket_ai_service/
├── README.md
├── requirements.txt
├── setup.py
├── .env.example
├── config.yaml
├── main.py
├── src/
│   ├── __init__.py
│   ├── server.py              # WebSocket 服务器
│   ├── session.py             # 会话处理器
│   ├── config.py              # 配置管理
│   ├── adapters/
│   │   ├── __init__.py
│   │   ├── base.py            # 抽象基类
│   │   ├── stt/
│   │   │   ├── __init__.py
│   │   │   ├── qwen.py        # Qwen STT
│   │   │   ├── google.py      # Google STT
│   │   │   └── azure.py       # Azure STT
│   │   ├── llm/
│   │   │   ├── __init__.py
│   │   │   ├── qwen.py        # Qwen LLM
│   │   │   ├── openai.py      # OpenAI GPT
│   │   │   └── claude.py      # Claude
│   │   └── tts/
│   │       ├── __init__.py
│   │       ├── qwen.py        # Qwen TTS
│   │       ├── azure.py       # Azure TTS
│   │       └── google.py      # Google TTS
│   ├── audio/
│   │   ├── __init__.py
│   │   ├── buffer.py          # 音频缓冲
│   │   ├── vad.py             # VAD 处理
│   │   └── converter.py       # 格式转换
│   └── utils/
│       ├── __init__.py
│       ├── logger.py          # 日志工具
│       └── metrics.py         # 指标统计
├── tests/
│   ├── test_server.py
│   ├── test_session.py
│   ├── test_adapters.py
│   └── test_audio.py
└── docker/
    ├── Dockerfile
    └── docker-compose.yml
```

### 8.2 核心实现代码

由于篇幅限制，这里仅展示关键文件的框架。完整实现代码请参考项目源代码。

#### main.py

```python
#!/usr/bin/env python3
"""
WebSocket AI 对话服务主入口
"""
import asyncio
import signal
import sys
from pathlib import Path

from src.config import load_config
from src.server import AudioStreamServer
from src.utils.logger import setup_logger

logger = setup_logger(__name__)


async def main():
    """主函数"""
    # 加载配置
    config = load_config()

    # 创建服务器
    server = AudioStreamServer(
        host=config.host,
        port=config.port,
        config=config
    )

    # 信号处理
    def signal_handler(sig, frame):
        logger.info("收到停止信号，正在关闭服务器...")
        asyncio.create_task(server.stop())

    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    # 启动服务器
    try:
        logger.info(f"启动 WebSocket AI 服务: ws://{config.host}:{config.port}")
        await server.start()
    except Exception as e:
        logger.error(f"服务器错误: {e}", exc_info=True)
        sys.exit(1)


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("服务器已停止")
```

详细的实现代码包括：

- `src/server.py`: WebSocket 服务器实现
- `src/session.py`: 会话处理器实现
- `src/config.py`: 配置管理
- `src/adapters/stt/qwen.py`: Qwen STT 适配器
- `src/adapters/llm/qwen.py`: Qwen LLM 适配器
- `src/adapters/tts/qwen.py`: Qwen TTS 适配器
- `src/audio/buffer.py`: 音频缓冲器
- `src/audio/vad.py`: VAD 处理器
- `src/utils/logger.py`: 日志工具

完整代码实现请参考第 4 章核心组件设计中的示例代码。

### 8.3 requirements.txt

```txt
# WebSocket 服务器
websockets>=12.0
aiohttp>=3.9.0

# 音频处理
numpy>=1.24.0
scipy>=1.11.0
webrtcvad>=2.0.10

# 配置管理
python-dotenv>=1.0.0
PyYAML>=6.0

# 测试
pytest>=7.4.0
pytest-asyncio>=0.21.0
pytest-cov>=4.1.0
```

---

## 9. 部署指南

### 9.1 本地开发部署

#### 步骤 1: 安装依赖

```bash
# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# 或 venv\Scripts\activate  # Windows

# 安装依赖
pip install -r requirements.txt
```

#### 步骤 2: 配置环境变量

创建 `.env` 文件：

```bash
cp .env.example .env
nano .env
```

配置 Qwen API 密钥：

```bash
QWEN_API_KEY=your_api_key_here
```

#### 步骤 3: 运行服务

```bash
python main.py
```

### 9.2 Docker 部署

#### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8080

# 运行服务
CMD ["python", "main.py"]
```

#### docker-compose.yml

```yaml
version: '3.8'

services:
  websocket-ai:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SERVER_HOST=0.0.0.0
      - SERVER_PORT=8080
      - QWEN_API_KEY=${QWEN_API_KEY}
      - LOG_LEVEL=INFO
    volumes:
      - ./logs:/var/log
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python", "-c", "import socket; s=socket.socket(); s.connect(('localhost',8080)); s.close()"]
      interval: 30s
      timeout: 10s
      retries: 3
```

#### 构建和运行

```bash
# 构建镜像
docker-compose build

# 运行服务
docker-compose up -d

# 查看日志
docker-compose logs -f
```

### 9.3 生产环境部署

#### 使用 Systemd 服务

创建 `/etc/systemd/system/websocket-ai.service`:

```ini
[Unit]
Description=WebSocket AI Service
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/websocket-ai
Environment="PATH=/opt/websocket-ai/venv/bin"
ExecStart=/opt/websocket-ai/venv/bin/python /opt/websocket-ai/main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable websocket-ai
sudo systemctl start websocket-ai
sudo systemctl status websocket-ai
```

#### 使用 Nginx 反向代理

```nginx
upstream websocket_backend {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name your-domain.com;

    location /stream {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 超时设置
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```

---

## 10. 测试方案

### 10.1 单元测试

详细的单元测试示例请参考项目中的 `tests/` 目录。

### 10.2 集成测试

完整的集成测试代码请参考项目源代码。

### 10.3 WebSocket 客户端测试

```python
"""
WebSocket 客户端测试脚本
"""
import asyncio
import websockets
import json
import base64


async def test_client():
    """测试 WebSocket 连接"""

    uri = "ws://localhost:8080/stream"

    async with websockets.connect(uri) as websocket:
        print("已连接到服务器")

        # 发送文本消息测试
        message = {
            "type": "text",
            "content": "你好，请介绍一下你自己"
        }

        await websocket.send(json.dumps(message))
        print("已发送文本消息")

        # 接收响应
        response = await websocket.recv()
        print(f"收到响应: {len(response)} bytes")

        # 解析响应
        data = json.loads(response)

        if data.get("type") == "streamAudio":
            audio_data = base64.b64decode(data["data"]["audioData"])
            print(f"收到音频: {len(audio_data)} bytes")

            # 保存音频
            with open("response.pcm", "wb") as f:
                f.write(audio_data)
            print("音频已保存到 response.pcm")


if __name__ == "__main__":
    asyncio.run(test_client())
```

---

## 11. 最佳实践

### 11.1 性能优化

#### 1. 音频缓冲优化

```python
# 使用适当的缓冲大小
# 太小：频繁处理，增加开销
# 太大：延迟增加，用户体验差
OPTIMAL_BUFFER_DURATION = 0.5  # 500ms
```

#### 2. 并发处理

```python
# 使用 asyncio.gather 并发处理
async def process_parallel():
    stt_task = asyncio.create_task(stt_adapter.transcribe(...))
    # 可以同时执行其他任务

    result = await stt_task
```

#### 3. 连接池

```python
# 复用 HTTP 会话
session = aiohttp.ClientSession(
    connector=aiohttp.TCPConnector(limit=100)
)
```

### 11.2 错误处理

#### 1. 重试机制

```python
async def retry_on_error(func, max_retries=3):
    """带重试的函数调用"""
    for i in range(max_retries):
        try:
            return await func()
        except Exception as e:
            if i == max_retries - 1:
                raise
            await asyncio.sleep(2 ** i)  # 指数退避
```

#### 2. 超时控制

```python
# 为每个服务调用设置超时
try:
    result = await asyncio.wait_for(
        stt_adapter.transcribe(...),
        timeout=10.0
    )
except asyncio.TimeoutError:
    logger.error("STT 超时")
```

### 11.3 监控和日志

#### 1. 结构化日志

```python
logger.info("处理音频", extra={
    "session_id": session_id,
    "audio_length": len(audio_data),
    "duration": duration
})
```

#### 2. 指标收集

```python
# 记录关键指标
metrics = {
    "stt_latency": stt_end - stt_start,
    "llm_latency": llm_end - llm_start,
    "tts_latency": tts_end - tts_start,
    "total_latency": total_end - total_start
}
```

### 11.4 安全性

#### 1. API 密钥管理

```python
# 永远不要在代码中硬编码密钥
# 使用环境变量或密钥管理服务
api_key = os.getenv("QWEN_API_KEY")
```

#### 2. 输入验证

```python
# 验证音频数据大小
MAX_AUDIO_SIZE = 10 * 1024 * 1024  # 10MB

if len(audio_data) > MAX_AUDIO_SIZE:
    raise ValueError("音频数据过大")
```

### 11.5 用户打断功能优化

#### 1. VAD 参数调优

- 降低 VAD 灵敏度阈值，避免误触发
- 设置合理的静音检测时长（建议 300-500ms）

#### 2. 延迟优化

- 使用较小的音频缓冲（20-40ms）以减少延迟
- 选择低延迟的音频编解码器（如 raw PCM）

#### 3. 用户体验

- 在 AI 回复的自然停顿点更容易被打断
- 提供音频反馈提示用户打断成功（如"叮"声）

#### 4. 错误处理

- 处理网络抖动导致的误打断
- 实现打断次数限制，防止循环打断

---

## 12. 故障排查

### 12.1 常见问题

**问题**: 打断功能不工作

- **检查**: 确认 WebSocket 服务器实现了 VAD 和打断逻辑
- **检查**: 验证上行音频流是否正常（检查 `mono`/`stereo` 配置）
- **检查**: 查看 FreeSWITCH 日志中的 `mod_audio_stream::json` 事件

**问题**: 音频播放延迟高

- **调整**: 减小 `STREAM_BUFFER_SIZE` 值（如 20ms 或 40ms）
- **调整**: 使用 `raw` 音频格式代替 `mp3`/`ogg`
- **检查**: 网络延迟和带宽

**问题**: 频繁断开连接

- **启用**: `STREAM_HEART_BEAT`（建议 30 秒）
- **检查**: 负载均衡器的空闲超时设置
- **检查**: TLS 证书配置

### 12.2 调试建议

1. 禁用日志抑制：不设置 `STREAM_SUPPRESS_LOG`
2. 查看 FreeSWITCH 日志：`fs_cli -x "console loglevel debug"`
3. 使用 Wireshark 抓包分析 WebSocket 流量
4. 监控事件：`fs_cli -x "event plain mod_audio_stream::*"`

---

## 13. 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| **STT** | Speech-to-Text | 语音识别技术，将语音转换为文本 |
| **TTS** | Text-to-Speech | 语音合成技术，将文本转换为语音 |
| **LLM** | Large Language Model | 大语言模型，用于自然语言理解和生成 |
| **VAD** | Voice Activity Detection | 语音活动检测，用于识别语音和静音 |
| **WebSocket** | - | 一种在单个TCP连接上进行全双工通讯的协议 |
| **PCM** | Pulse Code Modulation | 脉冲编码调制，一种音频编码格式 |
| **L16** | Linear 16-bit PCM | 16位线性PCM音频格式 |
| **WSS** | WebSocket Secure | 基于TLS/SSL的安全WebSocket连接 |
| **Qwen** | 通义千问 | 阿里云提供的AI服务平台 |
| **ESL** | Event Socket Library | FreeSWITCH事件套接字库 |
| **RAII** | Resource Acquisition Is Initialization | 资源获取即初始化，C++编程技术 |
| **Base64** | - | 一种将二进制数据编码为ASCII字符的编码方式 |
| **asyncio** | - | Python异步I/O框架 |
| **Docker** | - | 容器化平台 |
| **Nginx** | - | 高性能的HTTP和反向代理服务器 |

---

## 附录A: TTS 音频流传输详解

本节专门说明 TTS 语音流如何传输到 mod_audio_stream 的完整流程。

### A.1 传输流程概述

```
TTS 服务生成音频
        ↓
返回 PCM 音频数据 (bytes)
        ↓
Base64 编码
        ↓
封装为 streamAudio JSON
        ↓
通过 WebSocket 发送
        ↓
mod_audio_stream 接收
        ↓
Base64 解码
        ↓
保存为临时文件
        ↓
触发 mod_audio_stream::play 事件
        ↓
FreeSWITCH 播放给呼叫者
```

### A.2 详细实现步骤

#### 步骤 1: TTS 生成音频

```python
# 调用 TTS 适配器
audio_response = await self.tts_adapter.synthesize(
    text=response,              # AI 回复文本
    voice=self.config.tts_voice,    # 语音类型
    sample_rate=self.config.tts_sample_rate  # 采样率
)

# audio_response 是原始 PCM 音频数据 (bytes)
# 格式: L16 PCM, Big-endian, 16-bit
# 采样率: 8000 Hz 或 16000 Hz
```

#### 步骤 2: 音频编码

```python
# Base64 编码音频数据
audio_base64 = base64.b64encode(audio_response).decode('utf-8')

# 示例输出:
# "AAABAAEAAAAAAAAAAAEAAAABAAAAAQAAAAEAAAABAAAAAQAAAAEAAAABAAAAAgAAAA..."
```

#### 步骤 3: 封装 streamAudio 消息

```python
message = {
    "type": "streamAudio",
    "data": {
        "audioDataType": "raw",  # 或 "wav", "mp3", "ogg"
        "sampleRate": 8000,       # 必须匹配实际采样率
        "audioData": audio_base64
    }
}
```

**消息字段说明**:

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | 是 | 固定为 "streamAudio" |
| `data.audioDataType` | string | 是 | 音频格式：raw/wav/mp3/ogg |
| `data.sampleRate` | integer | 是* | 采样率（仅 raw 格式需要）|
| `data.audioData` | string | 是 | Base64 编码的音频数据 |

#### 步骤 4: WebSocket 发送

```python
# 转换为 JSON 字符串
json_message = json.dumps(message)

# 通过 WebSocket 发送（文本消息）
await self.websocket.send(json_message)

logger.info(f"音频已发送: {len(audio_response)} bytes")
```

#### 步骤 5: mod_audio_stream 接收处理

mod_audio_stream 模块会自动：

1. **接收 JSON 消息**
2. **解析 streamAudio 类型**
3. **Base64 解码音频数据**
4. **根据 audioDataType 创建临时文件**:
   - `raw`: `/tmp/freeswitch/audio_xxxxx.r8` (8kHz) 或 `.r16` (16kHz)
   - `wav`: `/tmp/freeswitch/audio_xxxxx.wav`
   - `mp3`: `/tmp/freeswitch/audio_xxxxx.mp3`
5. **触发 `mod_audio_stream::play` 事件**，包含文件路径
6. **应用程序监听事件并使用 `uuid_broadcast` 播放**

### A.3 完整代码示例

详细的代码实现请参考第 4 章核心组件设计。

### A.4 音频格式转换

#### WAV 格式

```python
import io
import wave

def pcm_to_wav(pcm_data: bytes, sample_rate: int) -> bytes:
    """将 PCM 转换为 WAV 格式"""

    wav_buffer = io.BytesIO()

    with wave.open(wav_buffer, 'wb') as wav_file:
        wav_file.setnchannels(1)      # 单声道
        wav_file.setsampwidth(2)      # 16-bit
        wav_file.setframerate(sample_rate)
        wav_file.writeframes(pcm_data)

    return wav_buffer.getvalue()
```

#### MP3 格式（需要 pydub）

```python
from pydub import AudioSegment
import io

def pcm_to_mp3(pcm_data: bytes, sample_rate: int) -> bytes:
    """将 PCM 转换为 MP3 格式"""

    # 创建 AudioSegment
    audio = AudioSegment(
        data=pcm_data,
        sample_width=2,
        frame_rate=sample_rate,
        channels=1
    )

    # 导出为 MP3
    mp3_buffer = io.BytesIO()
    audio.export(mp3_buffer, format="mp3", bitrate="32k")

    return mp3_buffer.getvalue()
```

### A.5 性能优化

#### 1. 流式传输（大音频）

对于长音频，可以分块发送：

```python
async def send_audio_chunked(self, audio_data: bytes, chunk_size: int = 32000):
    """分块发送大音频"""

    total_size = len(audio_data)
    chunks = [audio_data[i:i+chunk_size] for i in range(0, total_size, chunk_size)]

    logger.info(f"分块发送音频: {len(chunks)} 块, 总大小: {total_size} bytes")

    for i, chunk in enumerate(chunks):
        await self.send_audio(chunk)

        # 控制发送速率，避免缓冲区溢出
        if i < len(chunks) - 1:
            await asyncio.sleep(0.1)
```

#### 2. 压缩传输

使用 MP3 格式可以显著减少传输数据量：

```python
# Raw PCM: ~128 KB/s (8kHz, 16-bit)
# MP3 (32kbps): ~4 KB/s

# 对于 10 秒音频:
# PCM: 1.28 MB
# MP3: 40 KB

# 节省带宽: ~97%
```

---

## 版本历史

| 版本 | 日期 | 更新内容 |
|------|------|----------|
| 2.0.0 | 2026-02-09 | 合并 FreeSWITCH 和 Python 服务文档，消除重复，添加术语表 |
| 1.1.0 | 2026-02-06 | 添加附录A - TTS音频流传输详解 |
| 1.0.0 | 2026-02-06 | 初始版本，完整的 Python WebSocket 服务设计 |

---

## 参考资料

- [RFC 6455 - WebSocket 协议](https://tools.ietf.org/html/rfc6455)
- [FreeSWITCH 官方文档](https://freeswitch.org/confluence/)
- [libwsc - WebSocket 客户端库](https://github.com/amigniter/libwsc)
- [通义千问 API 文档](https://help.aliyun.com/zh/dashscope/)
- [Python asyncio 文档](https://docs.python.org/3/library/asyncio.html)
- [WebRTC VAD 文档](https://webrtc.org/)

---

## 版本信息

- **社区版**: 免费使用，限制 10 个并发通道
- **商业版**: 无并发限制，支持 5000+ 并发呼叫，提供源代码访问

更多信息请联系：[amsoftswitch@gmail.com](mailto:amsoftswitch@gmail.com)

---

**版权声明**: 本文档遵循 MIT 许可证

**最后更新**: 2026-02-09 (v2.0.0 - 文档整合与优化)
