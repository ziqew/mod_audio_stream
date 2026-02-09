# mod_audio_stream FreeSWITCH 集成指南

本指南详细说明如何在 FreeSWITCH 中集成和使用 mod_audio_stream 模块。

## 目录

1. [系统要求](#系统要求)
2. [安装方式](#安装方式)
3. [FreeSWITCH 配置](#freeswitch-配置)
4. [拨号计划示例](#拨号计划示例)
5. [事件处理](#事件处理)
6. [测试验证](#测试验证)
7. [常见问题](#常见问题)

---

## 系统要求

### 支持的操作系统

- Debian 10+
- Ubuntu 18.04+
- CentOS 7+（需要手动安装依赖）

### FreeSWITCH 版本

- FreeSWITCH 1.8.x
- FreeSWITCH 1.10.x

### 必需的依赖

```bash
# Debian/Ubuntu
sudo apt-get install -y \
    libfreeswitch-dev \
    libssl-dev \
    zlib1g-dev \
    libevent-dev \
    libspeexdsp-dev \
    cmake \
    build-essential \
    git
```

---

## 安装方式

### 方式一：使用预编译的 DEB 包（推荐）

适用于 Debian 12 系统。

#### 1. 下载安装包

从 [Releases 页面](https://github.com/amigniter/mod_audio_stream/releases) 下载最新版本：

```bash
wget https://github.com/amigniter/mod_audio_stream/releases/download/v1.0.3/mod-audio-stream_1.0.3_amd64.deb
```

#### 2. 安装

```bash
sudo dpkg -i mod-audio-stream_1.0.3_amd64.deb
```

#### 3. 验证安装

```bash
# 检查模块文件是否存在
ls -l /usr/lib/freeswitch/mod/mod_audio_stream.so
```

### 方式二：从源码编译安装

适用于所有支持的系统，或者需要自定义编译选项。

#### 1. 克隆仓库

```bash
cd /usr/src/
sudo git clone https://github.com/amigniter/mod_audio_stream.git
cd mod_audio_stream
```

#### 2. 初始化子模块

**重要**: 必须初始化 libwsc 子模块！

```bash
sudo git submodule init
sudo git submodule update
```

#### 3. 配置 PKG_CONFIG_PATH（如果从源码构建 FreeSWITCH）

如果 FreeSWITCH 安装在非标准位置（如 `/usr/local/freeswitch`）：

```bash
export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig:$PKG_CONFIG_PATH
```

对于标准安装（从 Debian/Ubuntu 包安装），通常不需要设置。

#### 4. 编译

**标准编译（不含 TLS）**：

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
```

**启用 TLS 支持**：

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DUSE_TLS=ON ..
make
```

#### 5. 安装

```bash
sudo make install
```

模块将被安装到 FreeSWITCH 模块目录（通常是 `/usr/lib/freeswitch/mod/`）。

#### 6. 验证安装

```bash
# 查找模块文件
find /usr -name "mod_audio_stream.so" 2>/dev/null
```

### 方式三：自动化脚本安装

使用仓库提供的自动化脚本：

```bash
sudo apt-get -y install git \
    && cd /usr/src/ \
    && git clone https://github.com/amigniter/mod_audio_stream.git \
    && cd mod_audio_stream \
    && sudo bash ./build-mod-audio-stream.sh
```

---

## FreeSWITCH 配置

### 步骤 1: 加载模块

#### 方法 A: 在 modules.conf.xml 中启用（推荐）

编辑 FreeSWITCH 配置文件：

```bash
sudo nano /etc/freeswitch/autoload_configs/modules.conf.xml
```

在 `<modules>` 部分添加：

```xml
<load module="mod_audio_stream"/>
```

**完整示例**：

```xml
<configuration name="modules.conf" description="Modules">
  <modules>
    <!-- 其他模块 -->
    <load module="mod_console"/>
    <load module="mod_logfile"/>
    
    <!-- ... -->
    
    <!-- 添加 mod_audio_stream -->
    <load module="mod_audio_stream"/>
    
    <!-- ... -->
  </modules>
</configuration>
```

#### 方法 B: 在 FreeSWITCH 控制台手动加载

连接到 FreeSWITCH 控制台：

```bash
fs_cli
```

加载模块：

```
freeswitch@localhost> load mod_audio_stream
```

验证模块已加载：

```
freeswitch@localhost> module_exists mod_audio_stream
true
```

### 步骤 2: 重启 FreeSWITCH（如果使用 modules.conf.xml）

```bash
sudo systemctl restart freeswitch
```

或者在 fs_cli 中：

```
freeswitch@localhost> reloadxml
freeswitch@localhost> reload mod_audio_stream
```

### 步骤 3: 验证模块可用

在 fs_cli 中测试 API 命令：

```
freeswitch@localhost> uuid_audio_stream
-USAGE: <uuid> [start | stop | send_text | pause | resume | graceful-shutdown ] [wss-url | path] [mono | mixed | stereo] [8000 | 16000] [metadata]
```

如果看到用法信息，说明模块已成功加载。

---

## 拨号计划示例

### 示例 1: 基本音频流传输（单声道，8kHz）

将呼叫者的音频流传输到 WebSocket 服务器。

**拨号计划** (`/etc/freeswitch/dialplan/default.xml` 或自定义文件)：

```xml
<extension name="audio_stream_test">
  <condition field="destination_number" expression="^9000$">
    <!-- 应答呼叫 -->
    <action application="answer"/>
    
    <!-- 启动音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start ws://your-server.com:8080/stream mono 8k"/>
    
    <!-- 播放静音，保持通话 30 秒 -->
    <action application="playback" data="silence_stream://30000"/>
    
    <!-- 停止音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} stop"/>
    
    <!-- 挂断 -->
    <action application="hangup"/>
  </condition>
</extension>
```

**使用方法**: 拨打 9000 将触发此拨号计划。

### 示例 2: 使用 HTTPS (WSS) 和初始元数据

启用 TLS 加密并发送初始元数据。

```xml
<extension name="audio_stream_secure">
  <condition field="destination_number" expression="^9001$">
    <action application="answer"/>
    
    <!-- 配置通道变量 -->
    <action application="set" data="STREAM_BUFFER_SIZE=60"/>
    <action application="set" data="STREAM_HEART_BEAT=30"/>
    <action application="set" data="STREAM_EXTRA_HEADERS={'Authorization':'Bearer YOUR_TOKEN'}"/>
    
    <!-- 启动音频流，发送 JSON 元数据 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start wss://secure-server.com/stream mono 16k {'caller':'${caller_id_number}','language':'zh-CN'}"/>
    
    <!-- 保持通话 -->
    <action application="playback" data="silence_stream://60000"/>
    
    <!-- 停止时发送关闭消息 -->
    <action application="uuid_audio_stream" 
            data="${uuid} stop {'reason':'call_ended'}"/>
    
    <action application="hangup"/>
  </condition>
</extension>
```

### 示例 3: 双向音频（立体声模式）

捕获呼叫者和被叫者的音频。

```xml
<extension name="audio_stream_stereo">
  <condition field="destination_number" expression="^9002$">
    <action application="answer"/>
    
    <!-- 启动立体声音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start ws://server.com:8080/stream stereo 8k"/>
    
    <!-- 桥接到另一个号码（双方音频都会被捕获）-->
    <action application="bridge" data="sofia/internal/1000@domain.com"/>
    
    <!-- 通话结束后自动停止 -->
    <action application="uuid_audio_stream" 
            data="${uuid} stop"/>
  </condition>
</extension>
```

### 示例 4: 实时语音识别（ASR）

将音频流传输到 ASR 服务并处理响应。

**拨号计划**：

```xml
<extension name="asr_stream">
  <condition field="destination_number" expression="^9003$">
    <action application="answer"/>
    
    <!-- 设置缓冲区为 100ms（降低延迟） -->
    <action application="set" data="STREAM_BUFFER_SIZE=100"/>
    
    <!-- 启动音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} start wss://asr.example.com/recognize mono 16k {'session_id':'${uuid}','language':'zh-CN'}"/>
    
    <!-- 播放提示音 -->
    <action application="playback" data="ivr/ivr-please_state_your_name.wav"/>
    
    <!-- 录音或保持通话，等待 ASR 结果 -->
    <action application="sleep" data="10000"/>
    
    <!-- 停止音频流 -->
    <action application="uuid_audio_stream" 
            data="${uuid} stop"/>
    
    <action application="hangup"/>
  </condition>
</extension>
```

### 示例 5: 使用 ESL 动态控制

通过 Event Socket Library 动态控制音频流。

**Python ESL 示例**：

```python
#!/usr/bin/env python3
import ESL

# 连接到 FreeSWITCH
con = ESL.ESLconnection("localhost", "8021", "ClueCon")

if not con.connected():
    print("连接失败")
    exit(1)

# 获取通道 UUID（假设已知）
uuid = "YOUR-CHANNEL-UUID"

# 启动音频流
cmd = f"uuid_audio_stream {uuid} start ws://localhost:8080/stream mono 8k"
con.api(cmd)

# 等待一段时间
import time
time.sleep(10)

# 发送文本消息
cmd = f"uuid_audio_stream {uuid} send_text {{'command':'start_transcription'}}"
con.api(cmd)

# 暂停音频流
time.sleep(5)
cmd = f"uuid_audio_stream {uuid} pause"
con.api(cmd)

# 恢复音频流
time.sleep(2)
cmd = f"uuid_audio_stream {uuid} resume"
con.api(cmd)

# 停止音频流
time.sleep(5)
cmd = f"uuid_audio_stream {uuid} stop {{'reason':'test_complete'}}"
con.api(cmd)
```

### 示例 6: 在通话中间启动流

在通话进行中动态启动音频流。

```xml
<extension name="bridge_with_optional_stream">
  <condition field="destination_number" expression="^1(\d{3})$">
    <action application="answer"/>
    
    <!-- 正常桥接 -->
    <action application="bridge" data="sofia/internal/$1@domain.com"/>
    
    <!-- 通话结束后的处理 -->
    <action application="log" data="INFO Call ended with status: ${bridge_hangup_cause}"/>
  </condition>
</extension>
```

**在通话中通过 ESL 启动流**：

```bash
# 在 fs_cli 中
uuid_audio_stream <uuid> start ws://server.com/stream mono 8k
```

---

## 事件处理

### 订阅模块事件

mod_audio_stream 生成的事件可以通过 ESL 或 Lua/JavaScript 脚本监听。

### 方法 1: 使用 fs_cli 监听事件

```bash
fs_cli
```

订阅所有 mod_audio_stream 事件：

```
freeswitch@localhost> events plain CUSTOM mod_audio_stream::connect
freeswitch@localhost> events plain CUSTOM mod_audio_stream::disconnect
freeswitch@localhost> events plain CUSTOM mod_audio_stream::error
freeswitch@localhost> events plain CUSTOM mod_audio_stream::json
freeswitch@localhost> events plain CUSTOM mod_audio_stream::play
```

### 方法 2: 使用 ESL Python 脚本

**完整的事件监听器示例**：

```python
#!/usr/bin/env python3
import ESL
import json

def handle_event(e):
    """处理接收到的事件"""
    event_name = e.getHeader("Event-Subclass")
    uuid = e.getHeader("Unique-ID")
    body = e.getBody()
    
    print(f"\n{'='*60}")
    print(f"事件类型: {event_name}")
    print(f"通道 UUID: {uuid}")
    
    try:
        data = json.loads(body) if body else {}
        print(f"事件数据: {json.dumps(data, indent=2, ensure_ascii=False)}")
    except:
        print(f"事件数据: {body}")
    
    # 根据事件类型进行处理
    if event_name == "mod_audio_stream::connect":
        print("✓ WebSocket 连接成功")
        
    elif event_name == "mod_audio_stream::disconnect":
        print("✓ WebSocket 断开连接")
        code = data.get("message", {}).get("code")
        reason = data.get("message", {}).get("reason")
        print(f"  关闭代码: {code}, 原因: {reason}")
        
    elif event_name == "mod_audio_stream::error":
        print("✗ 发生错误")
        error_code = data.get("message", {}).get("code")
        error_msg = data.get("message", {}).get("error")
        print(f"  错误代码: {error_code}, 消息: {error_msg}")
        
    elif event_name == "mod_audio_stream::json":
        print("收到 JSON 消息")
        # 处理服务器响应
        
    elif event_name == "mod_audio_stream::play":
        print("收到播放音频")
        file_path = data.get("file")
        print(f"  音频文件: {file_path}")
        # 可以使用 uuid_broadcast 播放音频
        # con.api(f"uuid_broadcast {uuid} {file_path}")

def main():
    # 连接到 FreeSWITCH
    con = ESL.ESLconnection("localhost", "8021", "ClueCon")
    
    if not con.connected():
        print("无法连接到 FreeSWITCH")
        return
    
    print("已连接到 FreeSWITCH，监听 mod_audio_stream 事件...")
    
    # 订阅事件
    con.events("plain", "CUSTOM mod_audio_stream::connect")
    con.events("plain", "CUSTOM mod_audio_stream::disconnect")
    con.events("plain", "CUSTOM mod_audio_stream::error")
    con.events("plain", "CUSTOM mod_audio_stream::json")
    con.events("plain", "CUSTOM mod_audio_stream::play")
    
    # 事件循环
    while True:
        e = con.recvEvent()
        if e:
            handle_event(e)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n停止监听")
```

### 方法 3: 在拨号计划中处理事件

使用 Lua 脚本在拨号计划中处理事件。

**拨号计划**：

```xml
<extension name="asr_with_event_handling">
  <condition field="destination_number" expression="^9004$">
    <action application="answer"/>
    <action application="lua" data="asr_handler.lua"/>
  </condition>
</extension>
```

**Lua 脚本** (`/usr/share/freeswitch/scripts/asr_handler.lua`)：

```lua
-- 订阅事件
local con = freeswitch.EventConsumer("CUSTOM", "mod_audio_stream::json")

-- 启动音频流
api = freeswitch.API()
local uuid = session:getVariable("uuid")
api:execute("uuid_audio_stream", uuid .. " start ws://localhost:8080 mono 16k")

-- 监听响应
session:setVariable("asr_result", "")

for i = 1, 30 do
    local event = con:pop(1)
    if event then
        local body = event:getBody()
        freeswitch.consoleLog("info", "收到 ASR 结果: " .. body .. "\n")
        
        -- 解析 JSON 并提取结果
        -- 这里可以使用 cjson 或其他 JSON 库
        session:setVariable("asr_result", body)
        
        -- 如果识别完成，退出循环
        if string.find(body, "final") then
            break
        end
    end
end

-- 停止音频流
api:execute("uuid_audio_stream", uuid .. " stop")

-- 根据结果执行操作
local result = session:getVariable("asr_result")
if result and string.len(result) > 0 then
    session:execute("playback", "ivr/ivr-thank_you.wav")
else
    session:execute("playback", "ivr/ivr-please_try_again.wav")
end
```

---

## 测试验证

### 测试 1: 使用简单的 WebSocket 回显服务器

创建一个简单的 WebSocket 服务器用于测试。

**Node.js 回显服务器**：

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

console.log('WebSocket 服务器运行在 ws://localhost:8080');

wss.on('connection', (ws) => {
    console.log('客户端已连接');
    
    ws.on('message', (message) => {
        if (typeof message === 'string') {
            console.log('收到文本:', message);
            // 回显文本消息
            ws.send(JSON.stringify({
                type: 'response',
                message: '收到: ' + message
            }));
        } else {
            console.log('收到二进制数据:', message.length, '字节');
            // 不回显音频，只记录
        }
    });
    
    ws.on('close', () => {
        console.log('客户端已断开');
    });
    
    ws.on('error', (error) => {
        console.error('WebSocket 错误:', error);
    });
});
```

**运行服务器**：

```bash
npm install ws
node websocket-server.js
```

### 测试 2: 进行测试呼叫

1. **启动 WebSocket 服务器**（如上）

2. **启动事件监听**：

```bash
fs_cli
freeswitch@localhost> events plain CUSTOM mod_audio_stream::connect
freeswitch@localhost> events plain CUSTOM mod_audio_stream::disconnect
```

3. **拨打测试号码**（如 9000）

4. **观察日志**：

- FreeSWITCH 日志应显示模块活动
- WebSocket 服务器应显示连接和数据接收
- fs_cli 应显示事件

### 测试 3: 验证音频传输

**Python 脚本保存音频数据**：

```python
import asyncio
import websockets
import struct

async def audio_receiver(websocket, path):
    print(f"客户端连接: {path}")
    
    audio_data = bytearray()
    
    async for message in websocket:
        if isinstance(message, bytes):
            # 二进制音频数据
            audio_data.extend(message)
            print(f"收到 {len(message)} 字节音频数据，总计: {len(audio_data)} 字节")
        else:
            # 文本消息
            print(f"收到文本: {message}")
    
    # 保存音频
    if len(audio_data) > 0:
        with open('received_audio.raw', 'wb') as f:
            f.write(audio_data)
        print(f"音频已保存到 received_audio.raw ({len(audio_data)} 字节)")

async def main():
    async with websockets.serve(audio_receiver, "0.0.0.0", 8080):
        print("WebSocket 服务器运行在 ws://0.0.0.0:8080")
        await asyncio.Future()  # 永久运行

asyncio.run(main())
```

**转换和播放音频**：

```bash
# 使用 ffmpeg 转换为 WAV
ffmpeg -f s16be -ar 8000 -ac 1 -i received_audio.raw output.wav

# 播放
aplay output.wav  # Linux
# 或
ffplay output.wav
```

### 测试 4: 测试所有 API 命令

```bash
# 在 fs_cli 中
# 1. 发起呼叫
originate user/1000 &park

# 2. 获取 UUID
show channels

# 3. 启动流
uuid_audio_stream <uuid> start ws://localhost:8080/stream mono 8k

# 4. 发送文本
uuid_audio_stream <uuid> send_text {"command":"test"}

# 5. 暂停
uuid_audio_stream <uuid> pause

# 6. 恢复
uuid_audio_stream <uuid> resume

# 7. 停止
uuid_audio_stream <uuid> stop {"reason":"test"}
```

---

## 常见问题

### 问题 1: 模块加载失败

**症状**：
```
Cannot load library mod_audio_stream.so
```

**解决方案**：

1. 检查模块文件是否存在：
```bash
ls -l /usr/lib/freeswitch/mod/mod_audio_stream.so
```

2. 检查依赖库：
```bash
ldd /usr/lib/freeswitch/mod/mod_audio_stream.so
```

3. 安装缺失的依赖：
```bash
sudo apt-get install libspeexdsp1 libssl1.1 zlib1g
```

4. 检查文件权限：
```bash
sudo chmod 755 /usr/lib/freeswitch/mod/mod_audio_stream.so
```

### 问题 2: WebSocket 连接失败

**症状**：
```
Event-Subclass: mod_audio_stream::error
code: 6 (CONNECT_FAILED)
```

**解决方案**：

1. **检查 URL 格式**：
   - 正确：`ws://server.com:8080/path`
   - 错误：`http://server.com:8080/path`

2. **测试网络连接**：
```bash
telnet server.com 8080
```

3. **检查防火墙**：
```bash
sudo ufw allow 8080/tcp
```

4. **验证 WebSocket 服务器**：
```bash
# 使用 wscat 测试
npm install -g wscat
wscat -c ws://server.com:8080/path
```

### 问题 3: TLS/SSL 握手失败

**症状**：
```
Event-Subclass: mod_audio_stream::error
code: 8 (SSL_HANDSHAKE_FAILED)
```

**解决方案**：

1. **验证证书**：
```bash
openssl s_client -connect server.com:443 -servername server.com
```

2. **使用自定义 CA**：
```xml
<action application="set" data="STREAM_TLS_CA_FILE=/path/to/ca-bundle.crt"/>
```

3. **禁用主机名验证（仅用于测试）**：
```xml
<action application="set" data="STREAM_TLS_DISABLE_HOSTNAME_VALIDATION=1"/>
```

4. **禁用证书验证（不推荐）**：
```xml
<action application="set" data="STREAM_TLS_CA_FILE=NONE"/>
```

### 问题 4: 音频延迟过高

**症状**: 音频传输有明显延迟。

**解决方案**：

1. **减小缓冲区大小**：
```xml
<action application="set" data="STREAM_BUFFER_SIZE=20"/>
```

2. **使用更高采样率**（如果服务器支持）：
```xml
<!-- 16kHz 可能比 8kHz 有更好的处理性能 -->
<action application="uuid_audio_stream" data="${uuid} start ws://... mono 16k"/>
```

3. **禁用压缩**（如果 CPU 是瓶颈）：
```xml
<action application="set" data="STREAM_MESSAGE_DEFLATE=1"/>
```

4. **检查网络延迟**：
```bash
ping -c 10 server.com
```

### 问题 5: 内存使用持续增长

**症状**: FreeSWITCH 内存使用不断增加。

**解决方案**：

1. **确保正确停止流**：
   - 总是调用 `uuid_audio_stream <uuid> stop`
   - 或在拨号计划中使用 `hangup_hook`

2. **检查临时文件**：
```bash
# 检查 /tmp 目录
ls -lh /tmp/freeswitch/
```

3. **监控内存**：
```bash
watch -n 1 'ps aux | grep freeswitch'
```

4. **重启 FreeSWITCH**（如果问题持续）：
```bash
sudo systemctl restart freeswitch
```

### 问题 6: 无法接收 mod_audio_stream 事件

**症状**: ESL 脚本未收到事件。

**解决方案**：

1. **正确订阅事件**：
```python
# 正确方式
con.events("plain", "CUSTOM mod_audio_stream::connect")

# 错误方式
con.events("plain", "mod_audio_stream::connect")  # 缺少 CUSTOM
```

2. **检查事件套接字配置**：
```bash
# 编辑 /etc/freeswitch/autoload_configs/event_socket.conf.xml
# 确保启用了 event_socket
```

3. **验证 ESL 连接**：
```python
if con.connected():
    print("已连接")
else:
    print("连接失败")
```

### 问题 7: 音频质量差或有噪音

**症状**: 接收的音频质量不佳。

**解决方案**：

1. **使用更高采样率**：
```xml
<action application="uuid_audio_stream" data="${uuid} start ws://... mono 16k"/>
```

2. **检查网络丢包**：
```bash
netstat -s | grep -i loss
```

3. **增加缓冲区大小**（减少丢包影响）：
```xml
<action application="set" data="STREAM_BUFFER_SIZE=60"/>
```

4. **启用压缩**（默认启用）：
```xml
<!-- 确保未禁用压缩 -->
<action application="unset" data="STREAM_MESSAGE_DEFLATE"/>
```

### 问题 8: 编译错误 - 找不到 FreeSWITCH 头文件

**症状**：
```
fatal error: switch.h: No such file or directory
```

**解决方案**：

1. **安装开发包**：
```bash
sudo apt-get install libfreeswitch-dev
```

2. **设置 PKG_CONFIG_PATH**：
```bash
export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig:$PKG_CONFIG_PATH
```

3. **验证 pkg-config**：
```bash
pkg-config --cflags freeswitch
pkg-config --libs freeswitch
```

### 问题 9: 子模块未初始化

**症状**：
```
libs/libwsc/src/WebSocketClient.h: No such file or directory
```

**解决方案**：

```bash
git submodule init
git submodule update
```

### 问题 10: 权限被拒绝

**症状**：
```
Permission denied when loading module
```

**解决方案**：

1. **检查文件所有者**：
```bash
ls -l /usr/lib/freeswitch/mod/mod_audio_stream.so
```

2. **修正所有者**：
```bash
sudo chown root:root /usr/lib/freeswitch/mod/mod_audio_stream.so
```

3. **检查 SELinux（CentOS/RHEL）**：
```bash
# 临时禁用
sudo setenforce 0

# 或设置正确的上下文
sudo chcon -t lib_t /usr/lib/freeswitch/mod/mod_audio_stream.so
```

---

## 高级配置

### 使用客户端证书认证

某些 WebSocket 服务器需要客户端证书。

```xml
<extension name="stream_with_client_cert">
  <condition field="destination_number" expression="^9010$">
    <action application="answer"/>
    
    <!-- 配置 TLS 客户端证书 -->
    <action application="set" data="STREAM_TLS_CERT_FILE=/path/to/client.crt"/>
    <action application="set" data="STREAM_TLS_KEY_FILE=/path/to/client.key"/>
    <action application="set" data="STREAM_TLS_CA_FILE=/path/to/ca.crt"/>
    
    <action application="uuid_audio_stream" 
            data="${uuid} start wss://secure-server.com/stream mono 16k"/>
    
    <action application="playback" data="silence_stream://30000"/>
    <action application="uuid_audio_stream" data="${uuid} stop"/>
  </condition>
</extension>
```

### 动态调整缓冲区大小

根据网络条件动态调整。

```xml
<extension name="adaptive_buffer">
  <condition field="destination_number" expression="^9011$">
    <action application="answer"/>
    
    <!-- 检查网络类型，设置不同的缓冲区 -->
    <action application="set" data="network_type=${network_addr}"/>
    
    <!-- 本地网络使用小缓冲区 -->
    <action application="set" data="STREAM_BUFFER_SIZE=20" inline="true"/>
    
    <!-- 如果是外部网络，使用大缓冲区 -->
    <action application="set" data="STREAM_BUFFER_SIZE=100" 
            condition="${network_addr} !~ /^192\.168\./"/>
    
    <action application="uuid_audio_stream" 
            data="${uuid} start ws://server.com/stream mono 8k"/>
    
    <action application="playback" data="silence_stream://30000"/>
    <action application="uuid_audio_stream" data="${uuid} stop"/>
  </condition>
</extension>
```

### 错误处理和重连

虽然 libwsc 不支持自动重连，但可以通过事件监听实现手动重连。

```python
#!/usr/bin/env python3
import ESL
import time

def monitor_and_reconnect():
    con = ESL.ESLconnection("localhost", "8021", "ClueCon")
    
    # 订阅错误事件
    con.events("plain", "CUSTOM mod_audio_stream::error")
    con.events("plain", "CUSTOM mod_audio_stream::disconnect")
    
    active_streams = {}  # uuid -> stream_config
    
    while True:
        e = con.recvEvent()
        if e:
            event_name = e.getHeader("Event-Subclass")
            uuid = e.getHeader("Unique-ID")
            
            if event_name == "mod_audio_stream::error":
                print(f"流 {uuid} 发生错误，尝试重连...")
                
                # 等待一小段时间
                time.sleep(2)
                
                # 重新启动流
                if uuid in active_streams:
                    config = active_streams[uuid]
                    cmd = f"uuid_audio_stream {uuid} start {config['url']} {config['mode']} {config['rate']}"
                    con.api(cmd)
                    
            elif event_name == "mod_audio_stream::disconnect":
                print(f"流 {uuid} 断开连接")
                # 从活动列表中移除
                if uuid in active_streams:
                    del active_streams[uuid]

if __name__ == "__main__":
    monitor_and_reconnect()
```

---

## 性能优化建议

### 1. 系统级优化

**增加文件描述符限制**：

```bash
# 编辑 /etc/security/limits.conf
freeswitch soft nofile 999999
freeswitch hard nofile 999999
```

**优化网络栈**：

```bash
# 编辑 /etc/sysctl.conf
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
```

应用更改：
```bash
sudo sysctl -p
```

### 2. FreeSWITCH 优化

**增加 RTP 缓冲区**：

编辑 `/etc/freeswitch/autoload_configs/switch.conf.xml`：

```xml
<param name="rtp-start-port" value="16384"/>
<param name="rtp-end-port" value="32768"/>
<param name="max-sessions" value="5000"/>
```

### 3. mod_audio_stream 特定优化

**对于高并发场景**：

- 使用较大的 `STREAM_BUFFER_SIZE`（60-100ms）减少系统调用
- 禁用压缩如果 CPU 是瓶颈
- 使用本地 WebSocket 服务器减少网络延迟
- 考虑使用 8kHz 而非 16kHz 如果带宽有限

**低延迟场景**：

- 使用 `STREAM_BUFFER_SIZE=20` 或更小
- 使用 16kHz 以获得更好的音质
- 确保 WebSocket 服务器就近部署
- 启用压缩节省带宽

---

## 监控和日志

### 启用详细日志

编辑 `/etc/freeswitch/autoload_configs/logfile.conf.xml`：

```xml
<param name="logfile" value="/var/log/freeswitch/freeswitch.log"/>
<param name="rollover" value="10485760"/>
<map name="all" value="console,debug,info,notice,warning,err,crit,alert"/>
```

在 fs_cli 中临时启用调试：

```
freeswitch@localhost> console loglevel debug
```

### 监控活动流

```bash
# 在 fs_cli 中查看活动的媒体钩子
show media_bugs
```

### 日志分析

```bash
# 过滤 mod_audio_stream 日志
tail -f /var/log/freeswitch/freeswitch.log | grep audio_stream

# 统计错误
grep "mod_audio_stream::error" /var/log/freeswitch/freeswitch.log | wc -l
```

---

## 完整部署示例

### 场景: 实时客服质量监控系统

**需求**：
- 监控所有呼入客服的通话
- 实时流式传输到质量分析服务器
- 检测关键词和情绪
- 记录所有交互

**架构**：

```
呼叫者 → FreeSWITCH → mod_audio_stream → WebSocket 服务器
                ↓                              ↓
            客服人员                      AI 分析服务
                                              ↓
                                         质量评分系统
```

**FreeSWITCH 拨号计划** (`/etc/freeswitch/dialplan/public/quality_monitor.xml`)：

```xml
<include>
  <extension name="inbound_with_quality_monitor">
    <condition field="destination_number" expression="^(400\d{7})$">
      <!-- 应答 -->
      <action application="answer"/>
      
      <!-- 设置通道变量 -->
      <action application="set" data="call_id=${uuid}"/>
      <action application="set" data="caller_number=${caller_id_number}"/>
      <action application="set" data="called_number=$1"/>
      
      <!-- 配置音频流参数 -->
      <action application="set" data="STREAM_BUFFER_SIZE=60"/>
      <action application="set" data="STREAM_HEART_BEAT=30"/>
      <action application="set" data="STREAM_SUPPRESS_LOG=1"/>
      <action application="set" data="STREAM_EXTRA_HEADERS={'Authorization':'Bearer ${quality_api_token}'}"/>
      
      <!-- 准备元数据 -->
      <action application="set" data="metadata={'call_id':'${call_id}','caller':'${caller_number}','agent':'${agent_id}','queue':'customer_service'}"/>
      
      <!-- 启动质量监控流（立体声，捕获双方音频） -->
      <action application="uuid_audio_stream" 
              data="${uuid} start wss://quality.example.com/monitor stereo 16k ${metadata}"/>
      
      <!-- 播放欢迎语 -->
      <action application="playback" data="ivr/ivr-welcome.wav"/>
      
      <!-- 桥接到客服 -->
      <action application="bridge" data="user/${agent_id}"/>
      
      <!-- 通话结束，发送关闭元数据 -->
      <action application="set" data="end_metadata={'call_id':'${call_id}','duration':'${duration}','hangup_cause':'${hangup_cause}'}"/>
      <action application="uuid_audio_stream" 
              data="${uuid} stop ${end_metadata}"/>
      
      <!-- 挂断 -->
      <action application="hangup"/>
    </condition>
  </extension>
</include>
```

**质量监控服务器**（Node.js 示例）：

```javascript
const WebSocket = require('ws');
const fs = require('fs');
const path = require('path');

const wss = new WebSocket.Server({ port: 8443 });

wss.on('connection', (ws, req) => {
    let callData = null;
    let audioFile = null;
    let audioStream = null;
    
    ws.on('message', (message) => {
        if (typeof message === 'string') {
            // JSON 元数据
            try {
                const data = JSON.parse(message);
                
                if (!callData) {
                    // 初始元数据
                    callData = data;
                    console.log('新通话:', callData.call_id);
                    
                    // 创建音频文件
                    const filename = `${callData.call_id}_${Date.now()}.raw`;
                    audioFile = path.join('/var/recordings', filename);
                    audioStream = fs.createWriteStream(audioFile);
                    
                    // 开始质量分析
                    startQualityAnalysis(callData);
                } else {
                    // 结束元数据
                    console.log('通话结束:', callData.call_id, data);
                    
                    if (audioStream) {
                        audioStream.end();
                        processRecording(audioFile, callData);
                    }
                }
            } catch (e) {
                console.error('JSON 解析错误:', e);
            }
        } else {
            // 二进制音频数据
            if (audioStream) {
                audioStream.write(message);
            }
            
            // 实时分析音频
            analyzeAudioChunk(message, callData);
        }
    });
    
    ws.on('close', () => {
        console.log('连接关闭:', callData?.call_id);
        if (audioStream) {
            audioStream.end();
        }
    });
});

function startQualityAnalysis(callData) {
    // 初始化质量分析会话
    console.log('开始质量分析:', callData.call_id);
}

function analyzeAudioChunk(audioData, callData) {
    // 实时分析音频块
    // 例如：情绪检测、关键词识别、音量检测等
}

function processRecording(audioFile, callData) {
    // 后处理录音
    console.log('处理录音:', audioFile);
    
    // 转换为 WAV
    // 进行完整转录
    // 生成质量报告
}

console.log('质量监控服务器运行在 wss://localhost:8443');
```

---

## 总结

通过本指南，您应该能够：

1. ✅ 成功安装 mod_audio_stream 模块
2. ✅ 在 FreeSWITCH 中加载和配置模块
3. ✅ 创建各种场景的拨号计划
4. ✅ 处理模块事件
5. ✅ 测试和验证集成
6. ✅ 排查常见问题
7. ✅ 优化性能
8. ✅ 部署生产系统

### 下一步

- 查看 [代码逻辑分析.md](./代码逻辑分析.md) 了解模块内部工作原理
- 访问 [GitHub 仓库](https://github.com/amigniter/mod_audio_stream) 获取最新更新
- 加入社区讨论获取支持

### 获取帮助

如果遇到问题：

1. 检查 FreeSWITCH 日志 (`/var/log/freeswitch/freeswitch.log`)
2. 查看本指南的"常见问题"部分
3. 在 GitHub 上提交 issue
4. 联系技术支持: amsoftswitch@gmail.com

---

**祝您集成成功！** 🎉
