# WebSocket AI 服务 - 快速开始

本文档提供 WebSocket AI 对话服务的快速入门指南。

完整设计文档请查看：[WebSocket_AI_服务设计文档.md](./WebSocket_AI_服务设计文档.md)

---

## 快速概览

### 系统功能

```
音频输入 → STT → LLM → TTS → 音频输出
```

- 📥 接收来自 mod_audio_stream 的实时音频流
- 🎤 使用 STT 将语音转换为文本
- 🤖 通过 LLM 生成智能回复
- 🔊 使用 TTS 将回复转换为语音
- 📤 发送音频回 mod_audio_stream 播放

### 默认配置

- **STT**: Qwen Audio Turbo
- **LLM**: Qwen Turbo
- **TTS**: CosyVoice V1
- **采样率**: 8000 Hz
- **音频格式**: L16 PCM

---

## 5分钟快速部署

### 步骤 1: 克隆项目

```bash
git clone https://github.com/your-org/websocket-ai-service.git
cd websocket-ai-service
```

### 步骤 2: 安装依赖

```bash
# 创建虚拟环境
python3 -m venv venv
source venv/bin/activate  # Linux/Mac

# 安装依赖
pip install -r requirements.txt
```

**requirements.txt**:
```txt
websockets>=12.0
aiohttp>=3.9.0
webrtcvad>=2.0.10
python-dotenv>=1.0.0
PyYAML>=6.0
```

### 步骤 3: 配置环境

创建 `.env` 文件：

```bash
# Qwen API 密钥（必填）
QWEN_API_KEY=your_qwen_api_key_here

# 服务器配置
SERVER_HOST=0.0.0.0
SERVER_PORT=8080

# STT 配置
STT_PROVIDER=qwen
STT_MODEL=qwen-audio-turbo
STT_LANGUAGE=zh-CN

# LLM 配置
LLM_PROVIDER=qwen
LLM_MODEL=qwen-turbo
LLM_SYSTEM_PROMPT=你是一个友好、专业的AI助手。

# TTS 配置
TTS_PROVIDER=qwen
TTS_MODEL=cosyvoice-v1
TTS_VOICE=longxiaochun
TTS_SAMPLE_RATE=8000

# 日志
LOG_LEVEL=INFO
```

### 步骤 4: 运行服务

```bash
python main.py
```

输出：
```
2026-02-06 10:00:00 - INFO - 启动 WebSocket AI 服务: ws://0.0.0.0:8080
```

---

## 最小实现代码

### main.py

```python
#!/usr/bin/env python3
import asyncio
from src.server import AudioStreamServer
from src.config import load_config

async def main():
    config = load_config()
    server = AudioStreamServer(
        host=config.host,
        port=config.port,
        config=config
    )
    await server.start()

if __name__ == "__main__":
    asyncio.run(main())
```

### 项目结构

```
websocket-ai-service/
├── main.py                 # 入口文件
├── requirements.txt        # 依赖
├── .env                    # 配置
├── src/
│   ├── server.py          # WebSocket 服务器
│   ├── session.py         # 会话处理
│   ├── config.py          # 配置管理
│   ├── adapters/          # 服务适配器
│   │   ├── stt/
│   │   │   └── qwen.py    # Qwen STT
│   │   ├── llm/
│   │   │   └── qwen.py    # Qwen LLM
│   │   └── tts/
│   │       └── qwen.py    # Qwen TTS
│   └── audio/             # 音频处理
│       ├── buffer.py
│       └── vad.py
```

---

## 与 mod_audio_stream 集成

### FreeSWITCH 拨号计划

```xml
<extension name="ai_assistant">
  <condition field="destination_number" expression="^9999$">
    <action application="answer"/>
    
    <!-- 启动音频流到 AI 服务 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start ws://your-server:8080/stream mono 8k"/>
    
    <!-- 保持通话活跃 -->
    <action application="sleep" data="300000"/>
    
    <!-- 停止音频流 -->
    <action application="uuid_audio_stream" data="${uuid} stop"/>
    
    <action application="hangup"/>
  </condition>
</extension>
```

### 测试呼叫

1. 拨打 `9999`
2. 说话（例如："你好，请介绍一下你自己"）
3. 等待 AI 语音回复
4. 继续对话

---

## 核心 API

### WebSocket 协议

**连接**: `ws://hostname:8080/stream`

**接收音频**:
- 类型: Binary
- 格式: L16 PCM (16-bit, Big-endian)
- 采样率: 8000 Hz 或 16000 Hz

**发送音频**:
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

### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `SERVER_HOST` | 0.0.0.0 | 监听地址 |
| `SERVER_PORT` | 8080 | 监听端口 |
| `QWEN_API_KEY` | - | Qwen API 密钥（必填）|
| `STT_MODEL` | qwen-audio-turbo | STT 模型 |
| `LLM_MODEL` | qwen-turbo | LLM 模型 |
| `TTS_MODEL` | cosyvoice-v1 | TTS 模型 |
| `TTS_VOICE` | longxiaochun | TTS 语音 |
| `VAD_ENABLED` | true | 启用 VAD |
| `LOG_LEVEL` | INFO | 日志级别 |

---

## Docker 快速部署

### docker-compose.yml

```yaml
version: '3.8'

services:
  websocket-ai:
    build: .
    ports:
      - "8080:8080"
    environment:
      - QWEN_API_KEY=${QWEN_API_KEY}
      - SERVER_HOST=0.0.0.0
      - SERVER_PORT=8080
    restart: unless-stopped
```

### 运行

```bash
# 构建
docker-compose build

# 启动
docker-compose up -d

# 查看日志
docker-compose logs -f
```

---

## 测试

### 简单测试客户端

```python
import asyncio
import websockets
import json
import base64

async def test():
    uri = "ws://localhost:8080/stream"
    
    async with websockets.connect(uri) as ws:
        # 发送文本测试
        msg = {
            "type": "text",
            "content": "你好"
        }
        await ws.send(json.dumps(msg))
        
        # 接收响应
        response = await ws.recv()
        data = json.loads(response)
        
        if data["type"] == "streamAudio":
            audio = base64.b64decode(data["data"]["audioData"])
            print(f"收到音频: {len(audio)} bytes")

asyncio.run(test())
```

---

## 常见问题

### Q: 如何获取 Qwen API 密钥？

访问 [阿里云 DashScope](https://dashscope.console.aliyun.com/) 注册并创建 API Key。

### Q: 如何更改 AI 的性格？

修改 `LLM_SYSTEM_PROMPT`:
```bash
LLM_SYSTEM_PROMPT="你是一个幽默风趣的AI助手，喜欢用生动的比喻解释问题。"
```

### Q: 如何支持其他语言？

修改 `STT_LANGUAGE`:
```bash
STT_LANGUAGE=en-US  # 英语
STT_LANGUAGE=ja-JP  # 日语
```

### Q: 如何调整 VAD 灵敏度？

```bash
VAD_AGGRESSIVENESS=3  # 0-3，越大越严格
```

### Q: 服务器日志在哪里？

```bash
# 控制台输出
python main.py

# 或指定日志文件
LOG_FILE=/var/log/websocket-ai.log
```

---

## 性能优化建议

1. **使用 16kHz 采样率**可获得更好的识别质量
2. **启用 VAD**可减少不必要的处理
3. **调整缓冲区大小**平衡延迟和稳定性
4. **使用 Docker**简化部署和管理
5. **配置 Nginx 反向代理**用于生产环境

---

## 下一步

- 📖 阅读完整设计文档：[WebSocket_AI_服务设计文档.md](./WebSocket_AI_服务设计文档.md)
- 🔧 查看 FreeSWITCH 集成：[FreeSWITCH集成指南.md](./FreeSWITCH集成指南.md)
- 🎵 了解音频播放控制：[音频播放控制指南.md](./音频播放控制指南.md)
- 🐍 学习 Python ESL：[Python_ESL_使用指南.md](./Python_ESL_使用指南.md)

---

**提示**: 这是快速入门指南。详细的架构设计、完整代码实现和高级配置，请参考完整设计文档。
