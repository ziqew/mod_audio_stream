# WebSocket AI 对话服务设计文档

## 文档概述

本设计文档描述了一个基于 Python 的 WebSocket 服务，该服务集成 mod_audio_stream，实现完整的语音对话 AI 系统。系统包含语音识别(STT)、大语言模型(LLM)和语音合成(TTS)三个核心组件，默认使用通义千问(Qwen)的服务。

### 版本信息

- **文档版本**: 1.1.0
- **创建日期**: 2026-02-06
- **最后更新**: 2026-02-06
- **作者**: AI Service Team
- **更新内容**: 添加附录A - TTS音频流传输详解

---

## 目录

1. [系统架构](#系统架构)
2. [技术规范](#技术规范)
3. [核心组件设计](#核心组件设计)
4. [数据流设计](#数据流设计)
5. [接口设计](#接口设计)
6. [配置管理](#配置管理)
7. [实现代码](#实现代码)
8. [部署指南](#部署指南)
9. [测试方案](#测试方案)
10. [最佳实践](#最佳实践)

---

## 系统架构

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

### 1.2 核心特性

- ✅ **实时音频处理**: 支持流式音频接收和发送
- ✅ **模块化设计**: STT、LLM、TTS 可独立配置和替换
- ✅ **异步架构**: 基于 asyncio，高并发支持
- ✅ **VAD 支持**: 智能语音端点检测
- ✅ **错误恢复**: 完善的异常处理和重连机制
- ✅ **配置灵活**: 支持环境变量和配置文件
- ✅ **日志完善**: 详细的调试和监控日志

---

## 技术规范

### 2.1 音频格式规范

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
    "audioData": "base64_encoded_pcm_data"
  }
}
```

**支持的 audioDataType**:
- `raw` - 原始 PCM 数据（需指定 sampleRate）
- `wav` - WAV 文件格式
- `mp3` - MP3 压缩格式
- `ogg` - OGG Vorbis 格式

### 2.2 WebSocket 协议

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

### 2.3 技术栈

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

## 核心组件设计

### 3.1 WebSocket 服务器

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

### 3.2 会话处理器

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

### 3.3 STT 适配器

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

### 3.4 LLM 适配器

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

### 3.5 TTS 适配器

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

### 3.6 音频处理模块

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

## 数据流设计

### 4.1 完整对话流程

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

### 4.2 序列图

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

## 接口设计

### 5.1 配置接口

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

### 5.2 API 端点

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

## 配置管理

### 6.1 环境变量配置

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

### 6.2 YAML 配置文件

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

## 实现代码

### 7.1 项目结构

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


### 7.2 核心实现代码

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

#### src/server.py

```python
"""
WebSocket 服务器实现
"""
import asyncio
import logging
import websockets
from websockets.server import WebSocketServerProtocol
from typing import Dict

from .session import SessionHandler
from .config import ServiceConfig

logger = logging.getLogger(__name__)


class AudioStreamServer:
    """WebSocket 音频流服务器"""
    
    def __init__(self, host: str, port: int, config: ServiceConfig):
        self.host = host
        self.port = port
        self.config = config
        self.sessions: Dict[str, SessionHandler] = {}
        self.server = None
        
    async def start(self):
        """启动服务器"""
        logger.info(f"启动服务器: {self.host}:{self.port}")
        
        async with websockets.serve(
            self.handle_connection,
            self.host,
            self.port,
            ping_interval=20,
            ping_timeout=10
        ):
            await asyncio.Future()  # 永久运行
            
    async def handle_connection(self, websocket: WebSocketServerProtocol, path: str):
        """处理 WebSocket 连接"""
        session_id = None
        
        try:
            # 获取远程地址
            remote_addr = websocket.remote_address
            logger.info(f"新连接来自: {remote_addr}")
            
            # 生成会话 ID
            session_id = f"{remote_addr[0]}_{remote_addr[1]}_{id(websocket)}"
            
            # 创建会话处理器
            session = SessionHandler(
                websocket=websocket,
                session_id=session_id,
                config=self.config
            )
            
            self.sessions[session_id] = session
            logger.info(f"会话创建: {session_id}, 活跃会话数: {len(self.sessions)}")
            
            # 运行会话
            await session.run()
            
        except websockets.exceptions.ConnectionClosed:
            logger.info(f"连接关闭: {session_id}")
        except Exception as e:
            logger.error(f"会话错误 {session_id}: {e}", exc_info=True)
        finally:
            # 清理会话
            if session_id and session_id in self.sessions:
                await self.sessions[session_id].cleanup()
                del self.sessions[session_id]
                logger.info(f"会话清理: {session_id}, 剩余会话数: {len(self.sessions)}")
                
    async def stop(self):
        """停止服务器"""
        logger.info("停止所有会话...")
        
        # 清理所有会话
        for session_id in list(self.sessions.keys()):
            await self.sessions[session_id].cleanup()
            
        self.sessions.clear()
        logger.info("服务器已停止")
```

#### src/session.py

```python
"""
会话处理器实现
"""
import asyncio
import base64
import json
import logging
from typing import Optional
from websockets.server import WebSocketServerProtocol

from .config import ServiceConfig
from .adapters.stt.qwen import QwenSTTAdapter
from .adapters.llm.qwen import QwenLLMAdapter
from .adapters.tts.qwen import QwenTTSAdapter
from .audio.buffer import AudioBuffer
from .audio.vad import VADProcessor

logger = logging.getLogger(__name__)


class SessionHandler:
    """会话处理器"""
    
    def __init__(self, websocket: WebSocketServerProtocol, session_id: str, config: ServiceConfig):
        self.websocket = websocket
        self.session_id = session_id
        self.config = config
        
        # 初始化适配器
        self.stt_adapter = QwenSTTAdapter(
            api_key=config.stt_api_key,
            model=config.stt_model
        )
        
        self.llm_adapter = QwenLLMAdapter(
            api_key=config.llm_api_key,
            model=config.llm_model,
            system_prompt=config.llm_system_prompt
        )
        
        self.tts_adapter = QwenTTSAdapter(
            api_key=config.tts_api_key,
            model=config.tts_model,
            voice=config.tts_voice
        )
        
        # 音频处理
        self.audio_buffer = AudioBuffer(sample_rate=config.audio_sample_rate)
        
        if config.vad_enabled:
            self.vad = VADProcessor(
                sample_rate=config.audio_sample_rate,
                aggressiveness=config.vad_aggressiveness
            )
        else:
            self.vad = None
            
        # 对话历史
        self.conversation_history = []
        
        # 状态
        self.is_processing = False
        
    async def run(self):
        """运行会话"""
        logger.info(f"[{self.session_id}] 会话开始")
        
        try:
            async for message in self.websocket:
                await self.handle_message(message)
        except Exception as e:
            logger.error(f"[{self.session_id}] 会话错误: {e}", exc_info=True)
            
    async def handle_message(self, message):
        """处理接收到的消息"""
        
        if isinstance(message, bytes):
            # 二进制音频数据
            await self.handle_audio(message)
        elif isinstance(message, str):
            # 文本消息（元数据）
            try:
                data = json.loads(message)
                await self.handle_text_message(data)
            except json.JSONDecodeError:
                logger.warning(f"[{self.session_id}] 无效的 JSON 消息")
                
    async def handle_audio(self, audio_data: bytes):
        """处理音频数据"""
        
        if self.is_processing:
            # 正在处理，忽略新音频
            return
            
        # 添加到缓冲区
        self.audio_buffer.append(audio_data)
        
        # VAD 检测
        if self.vad:
            speech_ended = await self._check_vad(audio_data)
            
            if speech_ended:
                # 检测到语音结束，处理
                await self._process_speech()
        else:
            # 无 VAD，检查缓冲区是否足够
            if self.audio_buffer.is_ready():
                await self._process_speech()
                
    async def _check_vad(self, audio_data: bytes) -> bool:
        """检查 VAD，返回是否语音结束"""
        
        # 将音频分帧处理
        frame_size = self.vad.frame_size
        
        for i in range(0, len(audio_data), frame_size):
            frame = audio_data[i:i+frame_size]
            
            if len(frame) == frame_size:
                is_speech, speech_ended = self.vad.process_frame(frame)
                
                if speech_ended:
                    return True
                    
        return False
        
    async def _process_speech(self):
        """处理完整语音"""
        
        self.is_processing = True
        
        try:
            # 获取音频数据
            audio_data = self.audio_buffer.get_and_clear()
            
            if len(audio_data) == 0:
                return
                
            logger.info(f"[{self.session_id}] 处理音频: {len(audio_data)} bytes")
            
            # 1. STT: 语音转文本
            text = await self.stt_adapter.transcribe(
                audio_data,
                sample_rate=self.config.audio_sample_rate,
                language=self.config.stt_language
            )
            
            if not text or len(text.strip()) == 0:
                logger.info(f"[{self.session_id}] 未识别到文本")
                return
                
            logger.info(f"[{self.session_id}] STT 结果: {text}")
            
            # 2. LLM: 对话生成
            self.conversation_history.append({
                "role": "user",
                "content": text
            })
            
            response = await self.llm_adapter.chat(
                messages=self.conversation_history,
                temperature=self.config.llm_temperature,
                max_tokens=self.config.llm_max_tokens
            )
            
            logger.info(f"[{self.session_id}] LLM 回复: {response}")
            
            self.conversation_history.append({
                "role": "assistant",
                "content": response
            })
            
            # 3. TTS: 文本转语音
            audio_response = await self.tts_adapter.synthesize(
                text=response,
                voice=self.config.tts_voice,
                sample_rate=self.config.tts_sample_rate
            )
            
            logger.info(f"[{self.session_id}] TTS 生成: {len(audio_response)} bytes")
            
            # 4. 发送音频
            await self.send_audio(audio_response)
            
        except Exception as e:
            logger.error(f"[{self.session_id}] 处理语音错误: {e}", exc_info=True)
        finally:
            self.is_processing = False
            
    async def send_audio(self, audio_data: bytes):
        """发送音频到客户端"""
        
        # Base64 编码
        audio_base64 = base64.b64encode(audio_data).decode('utf-8')
        
        # 构建 streamAudio 消息
        message = {
            "type": "streamAudio",
            "data": {
                "audioDataType": "raw",
                "sampleRate": self.config.tts_sample_rate,
                "audioData": audio_base64
            }
        }
        
        # 发送
        await self.websocket.send(json.dumps(message))
        logger.info(f"[{self.session_id}] 音频已发送")
        
    async def handle_text_message(self, data: dict):
        """处理文本消息"""
        
        msg_type = data.get("type")
        
        if msg_type == "text":
            # 直接文本输入（用于调试）
            content = data.get("content", "")
            logger.info(f"[{self.session_id}] 收到文本: {content}")
            
            # 直接处理为对话
            self.conversation_history.append({
                "role": "user",
                "content": content
            })
            
            response = await self.llm_adapter.chat(
                messages=self.conversation_history
            )
            
            self.conversation_history.append({
                "role": "assistant",
                "content": response
            })
            
            # TTS 并发送
            audio_response = await self.tts_adapter.synthesize(
                text=response,
                voice=self.config.tts_voice,
                sample_rate=self.config.tts_sample_rate
            )
            
            await self.send_audio(audio_response)
            
    async def cleanup(self):
        """清理会话资源"""
        logger.info(f"[{self.session_id}] 清理会话")
        
        # 关闭适配器
        if hasattr(self.stt_adapter, 'close'):
            await self.stt_adapter.close()
        if hasattr(self.llm_adapter, 'close'):
            await self.llm_adapter.close()
        if hasattr(self.tts_adapter, 'close'):
            await self.tts_adapter.close()
```

#### src/config.py

```python
"""
配置管理
"""
import os
from dataclasses import dataclass, field
from typing import Optional
from dotenv import load_dotenv
import yaml

# 加载环境变量
load_dotenv()


@dataclass
class ServiceConfig:
    """服务配置"""
    
    # 服务器配置
    host: str = field(default_factory=lambda: os.getenv("SERVER_HOST", "0.0.0.0"))
    port: int = field(default_factory=lambda: int(os.getenv("SERVER_PORT", "8080")))
    
    # STT 配置
    stt_provider: str = field(default_factory=lambda: os.getenv("STT_PROVIDER", "qwen"))
    stt_api_key: str = field(default_factory=lambda: os.getenv("QWEN_API_KEY", ""))
    stt_model: str = field(default_factory=lambda: os.getenv("STT_MODEL", "qwen-audio-turbo"))
    stt_language: str = field(default_factory=lambda: os.getenv("STT_LANGUAGE", "zh-CN"))
    
    # LLM 配置
    llm_provider: str = field(default_factory=lambda: os.getenv("LLM_PROVIDER", "qwen"))
    llm_api_key: str = field(default_factory=lambda: os.getenv("QWEN_API_KEY", ""))
    llm_model: str = field(default_factory=lambda: os.getenv("LLM_MODEL", "qwen-turbo"))
    llm_system_prompt: str = field(default_factory=lambda: os.getenv("LLM_SYSTEM_PROMPT", "你是一个友好的AI助手。"))
    llm_temperature: float = field(default_factory=lambda: float(os.getenv("LLM_TEMPERATURE", "0.7")))
    llm_max_tokens: int = field(default_factory=lambda: int(os.getenv("LLM_MAX_TOKENS", "2000")))
    
    # TTS 配置
    tts_provider: str = field(default_factory=lambda: os.getenv("TTS_PROVIDER", "qwen"))
    tts_api_key: str = field(default_factory=lambda: os.getenv("QWEN_API_KEY", ""))
    tts_model: str = field(default_factory=lambda: os.getenv("TTS_MODEL", "cosyvoice-v1"))
    tts_voice: str = field(default_factory=lambda: os.getenv("TTS_VOICE", "longxiaochun"))
    tts_sample_rate: int = field(default_factory=lambda: int(os.getenv("TTS_SAMPLE_RATE", "8000")))
    
    # 音频配置
    audio_sample_rate: int = field(default_factory=lambda: int(os.getenv("AUDIO_SAMPLE_RATE", "8000")))
    audio_channels: int = field(default_factory=lambda: int(os.getenv("AUDIO_CHANNELS", "1")))
    
    # VAD 配置
    vad_enabled: bool = field(default_factory=lambda: os.getenv("VAD_ENABLED", "true").lower() == "true")
    vad_aggressiveness: int = field(default_factory=lambda: int(os.getenv("VAD_AGGRESSIVENESS", "2")))
    
    # 日志配置
    log_level: str = field(default_factory=lambda: os.getenv("LOG_LEVEL", "INFO"))
    log_file: Optional[str] = field(default_factory=lambda: os.getenv("LOG_FILE"))


def load_config(config_file: Optional[str] = None) -> ServiceConfig:
    """加载配置"""
    
    if config_file and os.path.exists(config_file):
        # 从 YAML 文件加载
        with open(config_file, 'r', encoding='utf-8') as f:
            config_data = yaml.safe_load(f)
            
        # TODO: 将 YAML 数据合并到配置对象
        pass
        
    return ServiceConfig()
```


#### src/adapters/stt/qwen.py

```python
"""
Qwen STT 适配器实现
"""
import asyncio
import aiohttp
import io
import wave
import logging

logger = logging.getLogger(__name__)


class QwenSTTAdapter:
    """通义千问 STT 适配器"""
    
    def __init__(self, api_key: str, model: str = "qwen-audio-turbo"):
        self.api_key = api_key
        self.model = model
        self.endpoint = "https://dashscope.aliyuncs.com/api/v1/services/audio/asr/transcription"
        self.session = None
        
    async def _ensure_session(self):
        """确保 HTTP 会话存在"""
        if self.session is None:
            self.session = aiohttp.ClientSession()
            
    async def transcribe(self, audio_data: bytes, sample_rate: int, language: str = "zh-CN") -> str:
        """语音识别"""
        
        await self._ensure_session()
        
        try:
            # 转换为 WAV 格式
            wav_data = self._pcm_to_wav(audio_data, sample_rate)
            
            # 构建请求
            form = aiohttp.FormData()
            form.add_field('model', self.model)
            form.add_field('file', wav_data, 
                          filename='audio.wav',
                          content_type='audio/wav')
            form.add_field('language', language)
            
            headers = {
                'Authorization': f'Bearer {self.api_key}'
            }
            
            # 发送请求
            async with self.session.post(self.endpoint, data=form, headers=headers) as resp:
                if resp.status == 200:
                    result = await resp.json()
                    
                    # 解析结果
                    if 'output' in result and 'transcription' in result['output']:
                        text = result['output']['transcription']
                        return text
                    else:
                        logger.warning(f"STT 响应格式异常: {result}")
                        return ""
                else:
                    error_text = await resp.text()
                    logger.error(f"STT 请求失败: {resp.status}, {error_text}")
                    return ""
                    
        except Exception as e:
            logger.error(f"STT 错误: {e}", exc_info=True)
            return ""
            
    def _pcm_to_wav(self, pcm_data: bytes, sample_rate: int, channels: int = 1) -> bytes:
        """将 PCM 数据转换为 WAV 格式"""
        
        wav_buffer = io.BytesIO()
        
        with wave.open(wav_buffer, 'wb') as wav_file:
            wav_file.setnchannels(channels)
            wav_file.setsampwidth(2)  # 16-bit
            wav_file.setframerate(sample_rate)
            wav_file.writeframes(pcm_data)
            
        return wav_buffer.getvalue()
        
    async def close(self):
        """关闭会话"""
        if self.session:
            await self.session.close()
            self.session = None
```

#### src/adapters/llm/qwen.py

```python
"""
Qwen LLM 适配器实现
"""
import aiohttp
import logging
from typing import List, Dict, Optional

logger = logging.getLogger(__name__)


class QwenLLMAdapter:
    """通义千问 LLM 适配器"""
    
    def __init__(self, api_key: str, model: str = "qwen-turbo", system_prompt: str = None):
        self.api_key = api_key
        self.model = model
        self.system_prompt = system_prompt
        self.endpoint = "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation"
        self.session = None
        
    async def _ensure_session(self):
        """确保 HTTP 会话存在"""
        if self.session is None:
            self.session = aiohttp.ClientSession()
            
    async def chat(self, messages: List[Dict[str, str]], 
                   temperature: float = 0.7,
                   max_tokens: int = 2000) -> str:
        """对话生成"""
        
        await self._ensure_session()
        
        try:
            # 准备消息
            chat_messages = messages.copy()
            
            # 添加系统提示
            if self.system_prompt and (not chat_messages or chat_messages[0].get("role") != "system"):
                chat_messages.insert(0, {
                    "role": "system",
                    "content": self.system_prompt
                })
            
            # 构建请求
            payload = {
                "model": self.model,
                "input": {
                    "messages": chat_messages
                },
                "parameters": {
                    "result_format": "message",
                    "temperature": temperature,
                    "max_tokens": max_tokens
                }
            }
            
            headers = {
                'Authorization': f'Bearer {self.api_key}',
                'Content-Type': 'application/json'
            }
            
            # 发送请求
            async with self.session.post(self.endpoint, json=payload, headers=headers) as resp:
                if resp.status == 200:
                    result = await resp.json()
                    
                    # 解析结果
                    if 'output' in result and 'choices' in result['output']:
                        response_text = result['output']['choices'][0]['message']['content']
                        return response_text
                    else:
                        logger.warning(f"LLM 响应格式异常: {result}")
                        return "抱歉，我现在无法回答。"
                else:
                    error_text = await resp.text()
                    logger.error(f"LLM 请求失败: {resp.status}, {error_text}")
                    return "抱歉，服务暂时不可用。"
                    
        except Exception as e:
            logger.error(f"LLM 错误: {e}", exc_info=True)
            return "抱歉，发生了错误。"
            
    async def close(self):
        """关闭会话"""
        if self.session:
            await self.session.close()
            self.session = None
```

#### src/adapters/tts/qwen.py

```python
"""
Qwen TTS 适配器实现
"""
import aiohttp
import logging

logger = logging.getLogger(__name__)


class QwenTTSAdapter:
    """通义千问 TTS 适配器"""
    
    def __init__(self, api_key: str, model: str = "cosyvoice-v1", voice: str = "longxiaochun"):
        self.api_key = api_key
        self.model = model
        self.voice = voice
        self.endpoint = "https://dashscope.aliyuncs.com/api/v1/services/audio/tts/synthesis"
        self.session = None
        
    async def _ensure_session(self):
        """确保 HTTP 会话存在"""
        if self.session is None:
            self.session = aiohttp.ClientSession()
            
    async def synthesize(self, text: str, voice: str = None, sample_rate: int = 8000) -> bytes:
        """文本转语音"""
        
        await self._ensure_session()
        
        try:
            # 使用指定的 voice 或默认 voice
            voice_name = voice if voice else self.voice
            
            # 构建请求
            payload = {
                "model": self.model,
                "input": {
                    "text": text
                },
                "parameters": {
                    "voice": voice_name,
                    "format": "pcm",
                    "sample_rate": sample_rate
                }
            }
            
            headers = {
                'Authorization': f'Bearer {self.api_key}',
                'Content-Type': 'application/json'
            }
            
            # 发送请求
            async with self.session.post(self.endpoint, json=payload, headers=headers) as resp:
                if resp.status == 200:
                    # 返回音频数据
                    audio_data = await resp.read()
                    return audio_data
                else:
                    error_text = await resp.text()
                    logger.error(f"TTS 请求失败: {resp.status}, {error_text}")
                    return b""
                    
        except Exception as e:
            logger.error(f"TTS 错误: {e}", exc_info=True)
            return b""
            
    async def close(self):
        """关闭会话"""
        if self.session:
            await self.session.close()
            self.session = None
```

#### src/audio/buffer.py

```python
"""
音频缓冲器
"""


class AudioBuffer:
    """音频缓冲器"""
    
    def __init__(self, sample_rate: int = 8000, min_duration: float = 0.5):
        self.sample_rate = sample_rate
        self.min_duration = min_duration
        self.buffer = bytearray()
        
    def append(self, data: bytes):
        """添加音频数据"""
        self.buffer.extend(data)
        
    def get_duration(self) -> float:
        """获取缓冲区音频时长（秒）"""
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
        
    def clear(self):
        """清空缓冲区"""
        self.buffer.clear()
```

#### src/audio/vad.py

```python
"""
VAD (语音活动检测) 处理器
"""
import webrtcvad


class VADProcessor:
    """语音活动检测"""
    
    def __init__(self, sample_rate: int = 8000, aggressiveness: int = 2):
        """
        初始化 VAD
        
        参数:
            sample_rate: 采样率 (8000, 16000, 32000, 48000)
            aggressiveness: 激进度 (0-3，越大越严格)
        """
        self.vad = webrtcvad.Vad(aggressiveness)
        self.sample_rate = sample_rate
        self.frame_duration = 30  # ms
        self.frame_size = int(sample_rate * self.frame_duration / 1000) * 2  # bytes
        
        # 状态管理
        self.is_speaking = False
        self.silence_frames = 0
        self.speech_frames = 0
        self.max_silence_frames = 20  # 600ms 静音后结束
        self.min_speech_frames = 3    # 90ms 语音后开始
        
    def process_frame(self, frame: bytes) -> tuple:
        """
        处理音频帧
        
        返回:
            (is_speech, speech_ended)
        """
        
        if len(frame) != self.frame_size:
            return (False, False)
            
        # 检测是否为语音
        try:
            is_speech = self.vad.is_speech(frame, self.sample_rate)
        except:
            return (False, False)
        
        if is_speech:
            self.speech_frames += 1
            self.silence_frames = 0
            
            if not self.is_speaking and self.speech_frames >= self.min_speech_frames:
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
        
    def reset(self):
        """重置状态"""
        self.is_speaking = False
        self.silence_frames = 0
        self.speech_frames = 0
```

#### src/utils/logger.py

```python
"""
日志工具
"""
import logging
import sys
from typing import Optional


def setup_logger(name: str, log_file: Optional[str] = None, level: str = "INFO") -> logging.Logger:
    """设置日志记录器"""
    
    logger = logging.getLogger(name)
    logger.setLevel(getattr(logging, level.upper()))
    
    # 格式化器
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    
    # 控制台处理器
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)
    
    # 文件处理器
    if log_file:
        file_handler = logging.FileHandler(log_file, encoding='utf-8')
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)
    
    return logger
```

### 7.3 requirements.txt

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

## 部署指南

### 8.1 本地开发部署

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

### 8.2 Docker 部署

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

### 8.3 生产环境部署

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

## 测试方案

### 9.1 单元测试

#### tests/test_adapters.py

```python
"""
适配器单元测试
"""
import pytest
from src.adapters.stt.qwen import QwenSTTAdapter
from src.adapters.llm.qwen import QwenLLMAdapter
from src.adapters.tts.qwen import QwenTTSAdapter


@pytest.mark.asyncio
async def test_stt_adapter():
    """测试 STT 适配器"""
    adapter = QwenSTTAdapter(api_key="test_key")
    
    # 模拟音频数据
    audio_data = b'\x00' * 16000  # 1秒 8kHz PCM
    
    # 应该返回文本（或空字符串）
    result = await adapter.transcribe(audio_data, sample_rate=8000)
    assert isinstance(result, str)


@pytest.mark.asyncio
async def test_llm_adapter():
    """测试 LLM 适配器"""
    adapter = QwenLLMAdapter(api_key="test_key")
    
    messages = [
        {"role": "user", "content": "你好"}
    ]
    
    result = await adapter.chat(messages)
    assert isinstance(result, str)
    assert len(result) > 0


@pytest.mark.asyncio
async def test_tts_adapter():
    """测试 TTS 适配器"""
    adapter = QwenTTSAdapter(api_key="test_key")
    
    result = await adapter.synthesize("你好", sample_rate=8000)
    assert isinstance(result, bytes)
```

### 9.2 集成测试

#### tests/test_session.py

```python
"""
会话集成测试
"""
import pytest
import asyncio
from unittest.mock import MagicMock, AsyncMock
from src.session import SessionHandler
from src.config import ServiceConfig


@pytest.mark.asyncio
async def test_session_audio_processing():
    """测试完整的音频处理流程"""
    
    # 模拟 WebSocket
    mock_ws = AsyncMock()
    
    # 创建配置
    config = ServiceConfig()
    config.vad_enabled = False
    
    # 创建会话
    session = SessionHandler(
        websocket=mock_ws,
        session_id="test-session",
        config=config
    )
    
    # 模拟音频数据
    audio_data = b'\x00' * 16000
    
    # 处理音频
    await session.handle_audio(audio_data)
    
    # 等待处理完成
    await asyncio.sleep(1)
    
    # 验证发送了响应
    # assert mock_ws.send.called
```

### 9.3 WebSocket 客户端测试

#### test_client.py

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

## 最佳实践

### 10.1 性能优化

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

### 10.2 错误处理

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

### 10.3 监控和日志

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

### 10.4 安全性

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

---

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

#### WebSocket AI 服务端（发送 TTS 音频）

```python
class SessionHandler:
    async def _process_speech(self):
        """完整的对话处理流程"""
        
        # 1. STT: 语音识别
        text = await self.stt_adapter.transcribe(
            audio_data,
            sample_rate=8000,
            language="zh-CN"
        )
        
        # 2. LLM: 对话生成
        self.conversation_history.append({
            "role": "user",
            "content": text
        })
        
        response = await self.llm_adapter.chat(
            messages=self.conversation_history
        )
        
        self.conversation_history.append({
            "role": "assistant",
            "content": response
        })
        
        # 3. TTS: 文本转语音
        audio_data = await self.tts_adapter.synthesize(
            text=response,
            voice="longxiaochun",
            sample_rate=8000
        )
        
        # 4. 发送音频到 mod_audio_stream
        await self.send_audio(audio_data)
    
    async def send_audio(self, audio_data: bytes):
        """发送音频到 mod_audio_stream"""
        
        # Base64 编码
        audio_base64 = base64.b64encode(audio_data).decode('utf-8')
        
        # 构建 streamAudio 消息
        message = {
            "type": "streamAudio",
            "data": {
                "audioDataType": "raw",
                "sampleRate": self.config.tts_sample_rate,
                "audioData": audio_base64
            }
        }
        
        # 发送
        await self.websocket.send(json.dumps(message))
        
        logger.info(f"[{self.session_id}] TTS 音频已发送到 mod_audio_stream: "
                   f"{len(audio_data)} bytes, "
                   f"采样率: {self.config.tts_sample_rate} Hz")
```

#### FreeSWITCH 端（接收并播放）

**方式 1: 使用 Python ESL 自动播放**

```python
import ESL
import json

# 连接到 FreeSWITCH
con = ESL.ESLconnection("localhost", "8021", "ClueCon")

# 订阅播放事件
con.events("plain", "CUSTOM mod_audio_stream::play")

while True:
    e = con.recvEvent()
    
    if e:
        event_subclass = e.getHeader("Event-Subclass")
        
        if event_subclass == "mod_audio_stream::play":
            uuid = e.getHeader("Unique-ID")
            body = e.getBody()
            
            # 解析事件数据
            data = json.loads(body)
            file_path = data.get("file")
            
            print(f"收到 TTS 音频: {file_path}")
            
            # 播放音频到通道
            con.api(f"uuid_broadcast {uuid} {file_path} both")
            print(f"正在播放 TTS 音频到通道 {uuid}")
```

**方式 2: 使用 Lua 脚本自动播放**

```lua
-- /usr/share/freeswitch/scripts/auto_play_tts.lua

local con = freeswitch.EventConsumer("CUSTOM", "mod_audio_stream::play")
local uuid = session:getVariable("uuid")

while session:ready() do
    local event = con:pop(1)  -- 等待 1 秒
    
    if event then
        local event_uuid = event:getHeader("Unique-ID")
        
        if event_uuid == uuid then
            local body = event:getBody()
            local cjson = require("cjson")
            local data = cjson.decode(body)
            
            if data.file then
                freeswitch.consoleLog("info", "播放 TTS 音频: " .. data.file .. "\n")
                session:streamFile(data.file)
            end
        end
    end
end
```

**拨号计划配置**:

```xml
<extension name="ai_assistant_with_auto_play">
  <condition field="destination_number" expression="^9999$">
    <action application="answer"/>
    
    <!-- 启动音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start ws://ai-service:8080/stream mono 8k"/>
    
    <!-- 运行 Lua 脚本处理自动播放 -->
    <action application="lua" data="auto_play_tts.lua"/>
    
    <!-- 停止音频流 -->
    <action application="uuid_audio_stream" data="${uuid} stop"/>
    
    <action application="hangup"/>
  </condition>
</extension>
```

### A.4 音频格式转换

如果需要发送不同格式的音频：

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

# 使用
wav_data = pcm_to_wav(audio_data, 8000)

message = {
    "type": "streamAudio",
    "data": {
        "audioDataType": "wav",
        "audioData": base64.b64encode(wav_data).decode('utf-8')
    }
}
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

# 使用
mp3_data = pcm_to_mp3(audio_data, 8000)

message = {
    "type": "streamAudio",
    "data": {
        "audioDataType": "mp3",
        "audioData": base64.b64encode(mp3_data).decode('utf-8')
    }
}
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

#### 3. 异步处理

```python
async def process_and_send_tts(self, text: str):
    """异步 TTS 处理"""
    
    # TTS 和其他操作可以并行
    tts_task = asyncio.create_task(
        self.tts_adapter.synthesize(text)
    )
    
    # 同时执行其他任务
    # ...
    
    # 等待 TTS 完成
    audio_data = await tts_task
    await self.send_audio(audio_data)
```

### A.6 错误处理

```python
async def send_audio_safe(self, audio_data: bytes):
    """带错误处理的音频发送"""
    
    try:
        # 验证音频数据
        if not audio_data or len(audio_data) == 0:
            logger.warning("音频数据为空，跳过发送")
            return False
        
        # 检查大小限制
        max_size = 10 * 1024 * 1024  # 10MB
        if len(audio_data) > max_size:
            logger.error(f"音频数据过大: {len(audio_data)} bytes")
            return False
        
        # Base64 编码
        try:
            audio_base64 = base64.b64encode(audio_data).decode('utf-8')
        except Exception as e:
            logger.error(f"Base64 编码失败: {e}")
            return False
        
        # 构建消息
        message = {
            "type": "streamAudio",
            "data": {
                "audioDataType": "raw",
                "sampleRate": self.config.tts_sample_rate,
                "audioData": audio_base64
            }
        }
        
        # 发送
        try:
            await asyncio.wait_for(
                self.websocket.send(json.dumps(message)),
                timeout=5.0  # 5秒超时
            )
            logger.info(f"音频已发送: {len(audio_data)} bytes")
            return True
            
        except asyncio.TimeoutError:
            logger.error("发送音频超时")
            return False
            
        except Exception as e:
            logger.error(f"发送音频失败: {e}")
            return False
            
    except Exception as e:
        logger.error(f"音频发送错误: {e}", exc_info=True)
        return False
```

### A.7 调试和测试

#### 测试音频发送

```python
import asyncio
import websockets
import json
import base64

async def test_send_tts_audio():
    """测试 TTS 音频发送"""
    
    # 连接到 WebSocket 服务
    uri = "ws://localhost:8080/stream"
    
    async with websockets.connect(uri) as websocket:
        print("已连接到服务器")
        
        # 生成测试音频（1秒静音）
        sample_rate = 8000
        duration = 1.0
        num_samples = int(sample_rate * duration)
        test_audio = b'\x00' * (num_samples * 2)  # 16-bit
        
        # 编码
        audio_base64 = base64.b64encode(test_audio).decode('utf-8')
        
        # 构建消息
        message = {
            "type": "streamAudio",
            "data": {
                "audioDataType": "raw",
                "sampleRate": sample_rate,
                "audioData": audio_base64
            }
        }
        
        # 发送
        await websocket.send(json.dumps(message))
        print(f"测试音频已发送: {len(test_audio)} bytes")
        
        # 等待响应
        response = await websocket.recv()
        print(f"收到响应: {response[:100]}...")

# 运行测试
asyncio.run(test_send_tts_audio())
```

#### 验证音频质量

```python
import numpy as np

def validate_audio_quality(audio_data: bytes, sample_rate: int):
    """验证音频质量"""
    
    # 转换为 numpy 数组
    audio_array = np.frombuffer(audio_data, dtype=np.int16)
    
    # 检查音量
    rms = np.sqrt(np.mean(audio_array.astype(float)**2))
    print(f"RMS 音量: {rms:.2f}")
    
    # 检查是否有削波
    max_val = np.max(np.abs(audio_array))
    if max_val >= 32767:
        print("警告: 检测到音频削波")
    
    # 检查是否为静音
    if rms < 100:
        print("警告: 音频可能为静音")
    
    # 检查时长
    duration = len(audio_array) / sample_rate
    print(f"音频时长: {duration:.2f} 秒")
    
    return {
        "rms": rms,
        "max_amplitude": max_val,
        "duration": duration,
        "is_clipping": max_val >= 32767,
        "is_silent": rms < 100
    }
```

### A.8 常见问题

#### Q1: 音频没有播放？

**检查清单**:
1. 确认 WebSocket 连接正常
2. 验证 streamAudio 消息格式正确
3. 检查 mod_audio_stream::play 事件是否触发
4. 确认 FreeSWITCH 通道仍然活跃
5. 验证音频数据不为空

```python
# 添加调试日志
logger.debug(f"音频大小: {len(audio_data)} bytes")
logger.debug(f"采样率: {sample_rate} Hz")
logger.debug(f"Base64 大小: {len(audio_base64)} chars")
```

#### Q2: 音频质量差或有噪音？

**可能原因**:
1. 采样率不匹配
2. 字节序错误
3. 音频数据损坏

**解决方案**:
```python
# 确保采样率匹配
tts_sample_rate = 8000
message["data"]["sampleRate"] = tts_sample_rate

# 验证音频格式
if audioDataType == "raw":
    # PCM 必须是 16-bit, Big-endian
    # 确保 TTS 返回的是正确格式
```

#### Q3: 发送大音频导致延迟？

**解决方案**: 使用流式传输或压缩格式

```python
# 方案 1: 分块发送
await send_audio_chunked(audio_data, chunk_size=16000)

# 方案 2: 使用压缩格式
mp3_data = convert_to_mp3(audio_data)
message["data"]["audioDataType"] = "mp3"
```

### A.9 总结

TTS 音频流传输到 mod_audio_stream 的关键点：

1. ✅ **TTS 生成**: 获取 PCM 格式音频数据
2. ✅ **Base64 编码**: 将二进制数据编码为文本
3. ✅ **streamAudio 封装**: 构建符合规范的 JSON 消息
4. ✅ **WebSocket 发送**: 通过 WebSocket 传输到 mod_audio_stream
5. ✅ **自动播放**: mod_audio_stream 触发事件，应用程序播放

整个流程已在设计文档中完整实现，支持多种音频格式，具有良好的错误处理和性能优化。

---
## 总结

本设计文档详细描述了一个基于 Python 的 WebSocket AI 对话服务的完整实现方案。主要特点：

### ✅ 核心功能
- 实时音频流接收和处理
- STT、LLM、TTS 完整对话链路
- 默认集成通义千问（Qwen）服务
- 模块化、可扩展的适配器设计

### ✅ 技术亮点
- 异步架构，高并发支持
- VAD 智能语音端点检测
- 完善的错误处理和恢复
- 灵活的配置管理

### ✅ 生产就绪
- Docker 容器化部署
- 完整的测试方案
- 监控和日志
- 性能优化建议

### 📚 相关资源

- [mod_audio_stream 集成指南](./FreeSWITCH集成指南.md)
- [Python ESL 使用指南](./Python_ESL_使用指南.md)
- [音频播放控制指南](./音频播放控制指南.md)
- [代码逻辑分析](./代码逻辑分析.md)

---

**版权声明**: 本文档遵循 MIT 许可证

**最后更新**: 2026-02-06 (v1.1.0 - 添加TTS音频流传输详解)
