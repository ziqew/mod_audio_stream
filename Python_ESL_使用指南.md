# Python ESL (Event Socket Library) 使用指南

本指南详细介绍如何使用 Python ESL 来操作和控制 FreeSWITCH，特别是与 mod_audio_stream 模块的集成应用。

## 目录

1. [什么是 ESL](#什么是-esl)
2. [安装和配置](#安装和配置)
3. [ESL 连接模式](#esl-连接模式)
4. [基本操作](#基本操作)
5. [事件处理](#事件处理)
6. [mod_audio_stream 集成](#mod_audio_stream-集成)
7. [完整示例](#完整示例)
8. [最佳实践](#最佳实践)
9. [常见问题](#常见问题)

---

## 什么是 ESL

**ESL (Event Socket Library)** 是 FreeSWITCH 提供的一个强大接口，允许外部应用程序通过 TCP 套接字与 FreeSWITCH 进行通信。

### ESL 的主要功能

- 📞 **呼叫控制**: 发起、应答、挂断、转移呼叫
- 🎛️ **命令执行**: 执行 FreeSWITCH API 命令
- 📡 **事件监听**: 订阅和处理 FreeSWITCH 事件
- 🔧 **实时控制**: 动态修改通道变量和会话状态
- 📊 **监控管理**: 监控系统状态、通话质量等

### ESL 的应用场景

- 自动化呼叫中心系统
- 呼叫路由和智能分配
- 实时通话监控和质量检测
- 与第三方系统集成（CRM、数据库等）
- 开发自定义电话应用

---

## 安装和配置

### 安装 Python ESL

#### 方法一：使用 FreeSWITCH 源码编译（推荐）

```bash
# 1. 安装依赖
sudo apt-get install -y python3-dev swig

# 2. 进入 FreeSWITCH 源码目录
cd /usr/src/freeswitch

# 3. 编译 Python ESL
cd libs/esl
make pymod

# 4. 安装
sudo make pymod-install

# 或者手动复制
sudo cp python/ESL.py /usr/local/lib/python3.x/dist-packages/
sudo cp python/_ESL.so /usr/local/lib/python3.x/dist-packages/
```

#### 方法二：从预编译包安装（Debian/Ubuntu）

```bash
# 如果 FreeSWITCH 从包安装
sudo apt-get install python3-esl
```

#### 方法三：使用 pip 安装（第三方包）

```bash
# 注意：这是社区维护的版本，可能不是最新的
pip install python-ESL
```

### 验证安装

```python
#!/usr/bin/env python3
try:
    import ESL
    print("✓ ESL 模块安装成功")
    print(f"ESL 版本: {ESL.eslSetLogLevel(0)}")
except ImportError as e:
    print(f"✗ ESL 模块未安装: {e}")
```

### 配置 FreeSWITCH Event Socket

编辑 `/etc/freeswitch/autoload_configs/event_socket.conf.xml`:

```xml
<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <!-- 监听地址（0.0.0.0 允许远程连接）-->
    <param name="listen-ip" value="127.0.0.1"/>
    
    <!-- 监听端口 -->
    <param name="listen-port" value="8021"/>
    
    <!-- 密码 -->
    <param name="password" value="ClueCon"/>
    
    <!-- 应用所有访问控制列表 -->
    <!--<param name="apply-inbound-acl" value="lan"/>-->
  </settings>
</configuration>
```

**安全建议**：
- 生产环境使用强密码
- 限制 `listen-ip` 为 `127.0.0.1`（仅本地）或使用 ACL
- 配置防火墙规则

重启 FreeSWITCH 或重新加载配置：

```bash
fs_cli -x "reload mod_event_socket"
```

---

## ESL 连接模式

ESL 支持两种连接模式：**Inbound（入站）** 和 **Outbound（出站）**。

### Inbound 模式（入站连接）

应用程序主动连接到 FreeSWITCH 的 Event Socket。

**特点**：
- ✅ 应用程序控制连接
- ✅ 适合监控和管理
- ✅ 可以执行全局命令
- ✅ 支持事件订阅

**Python 代码示例**：

```python
#!/usr/bin/env python3
import ESL

# 连接到 FreeSWITCH
con = ESL.ESLconnection("localhost", "8021", "ClueCon")

# 检查连接状态
if con.connected():
    print("✓ 成功连接到 FreeSWITCH")
else:
    print("✗ 连接失败")
    exit(1)

# 执行 API 命令
result = con.api("status")
if result:
    print(result.getBody())

# 断开连接
con.disconnect()
```

### Outbound 模式（出站连接）

FreeSWITCH 主动连接到应用程序（通常用于处理特定呼叫）。

**特点**：
- ✅ FreeSWITCH 发起连接
- ✅ 每个呼叫一个连接
- ✅ 适合呼叫处理逻辑
- ✅ 在拨号计划中触发

**拨号计划配置**：

```xml
<extension name="outbound_socket">
  <condition field="destination_number" expression="^5000$">
    <action application="socket" data="localhost:9000 async full"/>
  </condition>
</extension>
```

**Python 服务器示例**：

```python
#!/usr/bin/env python3
import ESL
import socketserver

class ESLRequestHandler(socketserver.BaseRequestHandler):
    def handle(self):
        # 创建 ESL 连接（Outbound 模式）
        con = ESL.ESLconnection(self.request.fileno())
        
        # 获取呼叫信息
        info = con.getInfo()
        uuid = info.getHeader("Unique-ID")
        
        print(f"处理呼叫: {uuid}")
        
        # 应答呼叫
        con.execute("answer", "")
        
        # 播放语音
        con.execute("playback", "/usr/share/freeswitch/sounds/en/us/callie/ivr/8000/ivr-hello.wav")
        
        # 挂断
        con.execute("hangup", "")

# 创建服务器
server = socketserver.ThreadingTCPServer(("localhost", 9000), ESLRequestHandler)
print("ESL Outbound 服务器运行在端口 9000")
server.serve_forever()
```

---

## 基本操作

### 1. 连接和认证

```python
#!/usr/bin/env python3
import ESL

# 创建连接
con = ESL.ESLconnection("192.168.1.100", "8021", "YourPassword")

# 验证连接
if not con.connected():
    print("连接失败！")
    exit(1)

print("连接成功！")
```

### 2. 执行 API 命令

#### 同步执行（api）

```python
# 执行命令并等待结果
result = con.api("status")

if result:
    print("命令执行成功")
    print(f"结果: {result.getBody()}")
else:
    print("命令执行失败")
```

**常用 API 命令**：

```python
# 查看系统状态
con.api("status")

# 查看活动通道
con.api("show channels")

# 查看活动呼叫
con.api("show calls")

# 发起呼叫
con.api("originate user/1000 &echo")

# 挂断指定通道
con.api("uuid_kill <uuid>")

# 获取通道变量
con.api("uuid_getvar <uuid> caller_id_number")

# 设置通道变量
con.api("uuid_setvar <uuid> my_var my_value")
```

#### 异步执行（bgapi）

```python
# 异步执行命令（不等待结果）
result = con.bgapi("originate user/1000 &echo")

if result:
    job_uuid = result.getHeader("Job-UUID")
    print(f"后台任务 UUID: {job_uuid}")
```

**处理后台任务结果**：

```python
# 订阅后台任务事件
con.events("plain", "BACKGROUND_JOB")

# 执行后台任务
result = con.bgapi("status")
job_uuid = result.getHeader("Job-UUID")

# 等待任务完成
while True:
    e = con.recvEvent()
    if e:
        event_job_uuid = e.getHeader("Job-UUID")
        if event_job_uuid == job_uuid:
            print(f"任务结果: {e.getBody()}")
            break
```

### 3. 发送消息（sendmsg）

直接向特定通道发送命令。

```python
# 向指定 UUID 的通道发送命令
con.sendmsg(
    uuid="<channel-uuid>",
    command="execute",
    app="playback",
    args="/path/to/audio.wav"
)
```

**sendmsg 示例**：

```python
def play_audio(con, uuid, audio_file):
    """播放音频到指定通道"""
    con.sendmsg(uuid, 
                "execute\n"
                f"execute-app-name: playback\n"
                f"execute-app-arg: {audio_file}\n")

def hangup_call(con, uuid):
    """挂断指定通道"""
    con.sendmsg(uuid,
                "execute\n"
                "execute-app-name: hangup\n")
```

### 4. 设置和获取变量

```python
def set_channel_variable(con, uuid, var_name, var_value):
    """设置通道变量"""
    cmd = f"uuid_setvar {uuid} {var_name} {var_value}"
    result = con.api(cmd)
    return result.getBody().strip() == "+OK"

def get_channel_variable(con, uuid, var_name):
    """获取通道变量"""
    cmd = f"uuid_getvar {uuid} {var_name}"
    result = con.api(cmd)
    if result:
        value = result.getBody().strip()
        return value if value != "_undef_" else None
    return None

# 使用示例
uuid = "xxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx"
set_channel_variable(con, uuid, "my_custom_var", "test_value")
value = get_channel_variable(con, uuid, "my_custom_var")
print(f"变量值: {value}")
```

---

## 事件处理

### 1. 订阅事件

```python
# 订阅所有事件
con.events("plain", "all")

# 订阅特定事件类型
con.events("plain", "CHANNEL_CREATE")
con.events("plain", "CHANNEL_DESTROY")

# 订阅多个事件类型
con.events("plain", "CHANNEL_CREATE CHANNEL_DESTROY CHANNEL_ANSWER")

# 订阅自定义事件
con.events("plain", "CUSTOM mod_audio_stream::connect")
```

### 2. 接收和处理事件

```python
#!/usr/bin/env python3
import ESL

con = ESL.ESLconnection("localhost", "8021", "ClueCon")

if not con.connected():
    print("连接失败")
    exit(1)

# 订阅通道事件
con.events("plain", "CHANNEL_CREATE CHANNEL_DESTROY CHANNEL_ANSWER")

print("监听事件...")

while True:
    # 接收事件（阻塞）
    e = con.recvEvent()
    
    if e:
        # 获取事件名称
        event_name = e.getHeader("Event-Name")
        
        # 获取通道 UUID
        uuid = e.getHeader("Unique-ID")
        
        # 获取呼叫者号码
        caller_id = e.getHeader("Caller-Caller-ID-Number")
        
        print(f"事件: {event_name}, UUID: {uuid}, 呼叫者: {caller_id}")
        
        # 根据事件类型处理
        if event_name == "CHANNEL_CREATE":
            print(f"  → 新通道创建")
        elif event_name == "CHANNEL_ANSWER":
            print(f"  → 通道已应答")
        elif event_name == "CHANNEL_DESTROY":
            print(f"  → 通道已销毁")
```

### 3. 事件过滤

使用事件过滤器减少接收的事件数量。

```python
# 只接收特定呼叫者号码的事件
con.filter("Caller-Caller-ID-Number", "1234567890")

# 只接收特定目标号码的事件
con.filter("Caller-Destination-Number", "5000")

# 删除过滤器
con.filter_delete("Caller-Caller-ID-Number", "1234567890")
```

### 4. 非阻塞事件接收

```python
import ESL
import select
import time

con = ESL.ESLconnection("localhost", "8021", "ClueCon")

if not con.connected():
    exit(1)

con.events("plain", "CHANNEL_CREATE")

# 使用 select 进行非阻塞接收
while True:
    # 执行其他任务
    print("执行其他任务...")
    time.sleep(1)
    
    # 检查是否有事件
    e = con.recvEventTimed(0)  # 不阻塞
    if e:
        event_name = e.getHeader("Event-Name")
        print(f"收到事件: {event_name}")
```

---

## mod_audio_stream 集成

### 1. 控制音频流

#### 启动音频流

```python
#!/usr/bin/env python3
import ESL

def start_audio_stream(con, uuid, ws_url, mix_type="mono", sample_rate="8k", metadata=None):
    """
    启动音频流
    
    参数:
        con: ESL 连接对象
        uuid: 通道 UUID
        ws_url: WebSocket URL
        mix_type: 混合类型 (mono/mixed/stereo)
        sample_rate: 采样率 (8k/16k)
        metadata: 可选的 JSON 元数据
    """
    cmd = f"uuid_audio_stream {uuid} start {ws_url} {mix_type} {sample_rate}"
    
    if metadata:
        cmd += f" {metadata}"
    
    result = con.api(cmd)
    
    if result:
        response = result.getBody().strip()
        if response == "+OK Success":
            print(f"✓ 音频流启动成功: {uuid}")
            return True
        else:
            print(f"✗ 音频流启动失败: {response}")
            return False
    return False

# 使用示例
con = ESL.ESLconnection("localhost", "8021", "ClueCon")

if con.connected():
    uuid = "your-channel-uuid"
    metadata = '{"caller":"1234567890","language":"zh-CN"}'
    
    start_audio_stream(
        con, 
        uuid, 
        "ws://localhost:8080/stream",
        mix_type="mono",
        sample_rate="16k",
        metadata=metadata
    )
```

#### 停止音频流

```python
def stop_audio_stream(con, uuid, final_message=None):
    """
    停止音频流
    
    参数:
        con: ESL 连接对象
        uuid: 通道 UUID
        final_message: 可选的最终消息
    """
    cmd = f"uuid_audio_stream {uuid} stop"
    
    if final_message:
        cmd += f" {final_message}"
    
    result = con.api(cmd)
    
    if result:
        response = result.getBody().strip()
        if response == "+OK Success":
            print(f"✓ 音频流停止成功: {uuid}")
            return True
        else:
            print(f"✗ 音频流停止失败: {response}")
            return False
    return False

# 使用示例
final_msg = '{"reason":"call_ended","duration":120}'
stop_audio_stream(con, uuid, final_msg)
```

#### 暂停和恢复音频流

```python
def pause_audio_stream(con, uuid):
    """暂停音频流"""
    result = con.api(f"uuid_audio_stream {uuid} pause")
    return result.getBody().strip() == "+OK Success"

def resume_audio_stream(con, uuid):
    """恢复音频流"""
    result = con.api(f"uuid_audio_stream {uuid} resume")
    return result.getBody().strip() == "+OK Success"

# 使用示例
pause_audio_stream(con, uuid)
time.sleep(5)
resume_audio_stream(con, uuid)
```

#### 发送文本消息

```python
def send_text_to_stream(con, uuid, text):
    """
    向音频流发送文本消息
    
    参数:
        con: ESL 连接对象
        uuid: 通道 UUID
        text: JSON 格式的文本消息
    """
    cmd = f"uuid_audio_stream {uuid} send_text {text}"
    result = con.api(cmd)
    
    if result:
        response = result.getBody().strip()
        return response == "+OK Success"
    return False

# 使用示例
message = '{"command":"start_recognition","language":"zh-CN"}'
send_text_to_stream(con, uuid, message)
```

### 2. 监听 mod_audio_stream 事件

```python
#!/usr/bin/env python3
import ESL
import json

def handle_audio_stream_event(e):
    """处理 mod_audio_stream 事件"""
    event_subclass = e.getHeader("Event-Subclass")
    uuid = e.getHeader("Unique-ID")
    body = e.getBody()
    
    print(f"\n{'='*60}")
    print(f"事件类型: {event_subclass}")
    print(f"通道 UUID: {uuid}")
    
    try:
        data = json.loads(body) if body else {}
        print(f"事件数据: {json.dumps(data, indent=2, ensure_ascii=False)}")
    except:
        print(f"事件数据: {body}")
    
    # 根据事件类型处理
    if event_subclass == "mod_audio_stream::connect":
        print("✓ WebSocket 连接成功")
        
    elif event_subclass == "mod_audio_stream::disconnect":
        print("✓ WebSocket 断开连接")
        if isinstance(data, dict):
            code = data.get("message", {}).get("code")
            reason = data.get("message", {}).get("reason")
            print(f"  关闭代码: {code}, 原因: {reason}")
        
    elif event_subclass == "mod_audio_stream::error":
        print("✗ 发生错误")
        if isinstance(data, dict):
            error_code = data.get("message", {}).get("code")
            error_msg = data.get("message", {}).get("error")
            print(f"  错误代码: {error_code}, 消息: {error_msg}")
        
    elif event_subclass == "mod_audio_stream::json":
        print("📨 收到 JSON 消息")
        # 处理服务器响应
        
    elif event_subclass == "mod_audio_stream::play":
        print("🔊 收到播放音频")
        if isinstance(data, dict):
            file_path = data.get("file")
            print(f"  音频文件: {file_path}")

def monitor_audio_streams():
    """监控所有音频流事件"""
    con = ESL.ESLconnection("localhost", "8021", "ClueCon")
    
    if not con.connected():
        print("无法连接到 FreeSWITCH")
        return
    
    # 订阅所有 mod_audio_stream 事件
    con.events("plain", "CUSTOM mod_audio_stream::connect")
    con.events("plain", "CUSTOM mod_audio_stream::disconnect")
    con.events("plain", "CUSTOM mod_audio_stream::error")
    con.events("plain", "CUSTOM mod_audio_stream::json")
    con.events("plain", "CUSTOM mod_audio_stream::play")
    
    print("开始监听 mod_audio_stream 事件...")
    
    while True:
        e = con.recvEvent()
        if e:
            handle_audio_stream_event(e)

if __name__ == "__main__":
    try:
        monitor_audio_streams()
    except KeyboardInterrupt:
        print("\n停止监听")
```

### 3. 完整的音频流管理类

请参考本文档前面的"完整示例"部分中的 `AudioStreamManager` 类实现。


---

## 完整示例

### 示例 1: 音频流管理器类

```python
#!/usr/bin/env python3
import ESL
import json
import threading
import time

class AudioStreamManager:
    """完整的音频流管理器"""
    
    def __init__(self, host="localhost", port="8021", password="ClueCon"):
        """初始化连接"""
        self.con = ESL.ESLconnection(host, port, password)
        
        if not self.con.connected():
            raise Exception("无法连接到 FreeSWITCH")
        
        self.active_streams = {}  # uuid -> stream_info
        self.event_thread = None
        self.running = False
    
    def start_monitoring(self):
        """启动事件监控"""
        # 订阅事件
        self.con.events("plain", "CUSTOM mod_audio_stream::connect")
        self.con.events("plain", "CUSTOM mod_audio_stream::disconnect")
        self.con.events("plain", "CUSTOM mod_audio_stream::error")
        self.con.events("plain", "CUSTOM mod_audio_stream::json")
        self.con.events("plain", "CUSTOM mod_audio_stream::play")
        self.con.events("plain", "CHANNEL_HANGUP")
        
        # 启动事件处理线程
        self.running = True
        self.event_thread = threading.Thread(target=self._event_loop)
        self.event_thread.daemon = True
        self.event_thread.start()
        
        print("✓ 事件监控已启动")
    
    def _event_loop(self):
        """事件处理循环"""
        while self.running:
            e = self.con.recvEvent()
            if e:
                self._handle_event(e)
    
    def _handle_event(self, e):
        """处理事件"""
        event_name = e.getHeader("Event-Name")
        uuid = e.getHeader("Unique-ID")
        
        if event_name == "CUSTOM":
            subclass = e.getHeader("Event-Subclass")
            print(f"[{uuid}] {subclass}")
            
            if subclass == "mod_audio_stream::connect":
                if uuid in self.active_streams:
                    self.active_streams[uuid]["connected"] = True
                    
            elif subclass == "mod_audio_stream::disconnect":
                if uuid in self.active_streams:
                    self.active_streams[uuid]["connected"] = False
                    
            elif subclass == "mod_audio_stream::error":
                body = e.getBody()
                print(f"  错误: {body}")
                
        elif event_name == "CHANNEL_HANGUP":
            # 通道挂断时清理
            if uuid in self.active_streams:
                self.stop_stream(uuid)
                del self.active_streams[uuid]
    
    def start_stream(self, uuid, ws_url, mix_type="mono", sample_rate="8k", 
                     metadata=None, channel_vars=None):
        """启动音频流"""
        # 设置通道变量
        if channel_vars:
            for var, value in channel_vars.items():
                self.con.api(f"uuid_setvar {uuid} {var} {value}")
        
        # 启动流
        cmd = f"uuid_audio_stream {uuid} start {ws_url} {mix_type} {sample_rate}"
        if metadata:
            cmd += f" {metadata}"
        
        result = self.con.api(cmd)
        
        if result and result.getBody().strip() == "+OK Success":
            # 记录活动流
            self.active_streams[uuid] = {
                "ws_url": ws_url,
                "mix_type": mix_type,
                "sample_rate": sample_rate,
                "connected": False,
                "start_time": time.time()
            }
            print(f"✓ [{uuid}] 音频流已启动")
            return True
        else:
            print(f"✗ [{uuid}] 音频流启动失败")
            return False
    
    def stop_stream(self, uuid, final_message=None):
        """停止音频流"""
        cmd = f"uuid_audio_stream {uuid} stop"
        if final_message:
            cmd += f" {final_message}"
        
        result = self.con.api(cmd)
        
        if result and result.getBody().strip() == "+OK Success":
            if uuid in self.active_streams:
                duration = time.time() - self.active_streams[uuid]["start_time"]
                print(f"✓ [{uuid}] 音频流已停止 (持续: {duration:.1f}秒)")
            return True
        return False
    
    def pause_stream(self, uuid):
        """暂停音频流"""
        result = self.con.api(f"uuid_audio_stream {uuid} pause")
        return result.getBody().strip() == "+OK Success"
    
    def resume_stream(self, uuid):
        """恢复音频流"""
        result = self.con.api(f"uuid_audio_stream {uuid} resume")
        return result.getBody().strip() == "+OK Success"
    
    def send_text(self, uuid, text):
        """发送文本消息"""
        result = self.con.api(f"uuid_audio_stream {uuid} send_text {text}")
        return result.getBody().strip() == "+OK Success"
    
    def get_active_streams(self):
        """获取活动流列表"""
        return self.active_streams.copy()
    
    def stop_monitoring(self):
        """停止事件监控"""
        self.running = False
        if self.event_thread:
            self.event_thread.join(timeout=2)
        print("✓ 事件监控已停止")
    
    def disconnect(self):
        """断开连接"""
        self.stop_monitoring()
        self.con.disconnect()
        print("✓ 已断开连接")


# 使用示例
if __name__ == "__main__":
    # 创建管理器
    manager = AudioStreamManager("localhost", "8021", "ClueCon")
    
    # 启动监控
    manager.start_monitoring()
    
    # 假设已有一个活动通道
    uuid = "your-channel-uuid"
    
    # 配置通道变量
    channel_vars = {
        "STREAM_BUFFER_SIZE": "60",
        "STREAM_HEART_BEAT": "30"
    }
    
    # 启动音频流
    manager.start_stream(
        uuid,
        "ws://localhost:8080/stream",
        mix_type="mono",
        sample_rate="16k",
        metadata='{"session":"test"}',
        channel_vars=channel_vars
    )
    
    # 等待一段时间
    time.sleep(10)
    
    # 发送文本消息
    manager.send_text(uuid, '{"command":"status"}')
    
    # 暂停
    time.sleep(5)
    manager.pause_stream(uuid)
    
    # 恢复
    time.sleep(2)
    manager.resume_stream(uuid)
    
    # 停止
    time.sleep(10)
    manager.stop_stream(uuid, '{"reason":"test_complete"}')
    
    # 查看活动流
    print("活动流:", manager.get_active_streams())
    
    # 清理
    time.sleep(2)
    manager.disconnect()
```

### 示例 2: 自动呼叫并启动音频流

```python
#!/usr/bin/env python3
import ESL
import time
import json

def auto_call_with_stream(phone_number, ws_url):
    """
    自动呼叫号码并启动音频流
    
    参数:
        phone_number: 目标电话号码
        ws_url: WebSocket 服务器 URL
    """
    con = ESL.ESLconnection("localhost", "8021", "ClueCon")
    
    if not con.connected():
        print("连接失败")
        return
    
    # 订阅事件
    con.events("plain", "CHANNEL_CREATE CHANNEL_ANSWER CHANNEL_HANGUP")
    con.events("plain", "CUSTOM mod_audio_stream::connect")
    
    # 发起呼叫
    originate_cmd = (
        f"originate {{origination_caller_id_number=1234567890}}"
        f"user/{phone_number} &park"
    )
    
    print(f"正在呼叫 {phone_number}...")
    result = con.bgapi(originate_cmd)
    job_uuid = result.getHeader("Job-UUID")
    
    uuid = None
    
    # 等待呼叫建立
    timeout = time.time() + 30  # 30秒超时
    while time.time() < timeout:
        e = con.recvEventTimed(100)
        
        if e:
            event_name = e.getHeader("Event-Name")
            
            # 获取呼叫 UUID
            if event_name == "CHANNEL_CREATE":
                uuid = e.getHeader("Unique-ID")
                print(f"通道已创建: {uuid}")
            
            # 呼叫已应答
            elif event_name == "CHANNEL_ANSWER" and uuid:
                print(f"呼叫已应答: {uuid}")
                
                # 启动音频流
                metadata = json.dumps({
                    "phone_number": phone_number,
                    "call_time": time.strftime("%Y-%m-%d %H:%M:%S")
                })
                
                stream_cmd = f"uuid_audio_stream {uuid} start {ws_url} mono 16k {metadata}"
                stream_result = con.api(stream_cmd)
                
                if stream_result.getBody().strip() == "+OK Success":
                    print("✓ 音频流已启动")
                else:
                    print("✗ 音频流启动失败")
                
                # 保持通话30秒
                time.sleep(30)
                
                # 停止音频流
                stop_cmd = f"uuid_audio_stream {uuid} stop"
                con.api(stop_cmd)
                print("✓ 音频流已停止")
                
                # 挂断呼叫
                con.api(f"uuid_kill {uuid}")
                break
            
            # 呼叫挂断
            elif event_name == "CHANNEL_HANGUP" and uuid:
                print(f"呼叫已挂断: {uuid}")
                break
    
    con.disconnect()

# 使用示例
if __name__ == "__main__":
    auto_call_with_stream("1000", "ws://localhost:8080/stream")
```

### 示例 3: 实时通话转录监控

```python
#!/usr/bin/env python3
import ESL
import json
import threading
import queue
import time

class CallTranscriptionMonitor:
    """通话转录监控器"""
    
    def __init__(self):
        self.con = ESL.ESLconnection("localhost", "8021", "ClueCon")
        
        if not self.con.connected():
            raise Exception("连接失败")
        
        self.transcriptions = {}  # uuid -> transcription_data
        self.message_queue = queue.Queue()
    
    def start(self):
        """启动监控"""
        # 订阅事件
        self.con.events("plain", "CHANNEL_ANSWER")
        self.con.events("plain", "CHANNEL_HANGUP")
        self.con.events("plain", "CUSTOM mod_audio_stream::json")
        
        # 启动事件处理线程
        self.running = True
        threading.Thread(target=self._event_loop, daemon=True).start()
        threading.Thread(target=self._process_messages, daemon=True).start()
        
        print("✓ 转录监控已启动")
    
    def _event_loop(self):
        """事件循环"""
        while self.running:
            e = self.con.recvEvent()
            if e:
                self.message_queue.put(e)
    
    def _process_messages(self):
        """处理消息"""
        while self.running:
            try:
                e = self.message_queue.get(timeout=1)
                self._handle_event(e)
            except queue.Empty:
                continue
    
    def _handle_event(self, e):
        """处理事件"""
        event_name = e.getHeader("Event-Name")
        uuid = e.getHeader("Unique-ID")
        
        if event_name == "CHANNEL_ANSWER":
            # 新呼叫应答，启动音频流
            caller = e.getHeader("Caller-Caller-ID-Number")
            callee = e.getHeader("Caller-Destination-Number")
            
            print(f"\n新呼叫: {caller} → {callee} ({uuid})")
            
            # 初始化转录数据
            self.transcriptions[uuid] = {
                "caller": caller,
                "callee": callee,
                "start_time": time.time(),
                "transcripts": []
            }
            
            # 启动音频流到转录服务
            metadata = json.dumps({
                "uuid": uuid,
                "caller": caller,
                "callee": callee
            })
            
            cmd = f"uuid_audio_stream {uuid} start wss://transcription-service.com/stream mono 16k {metadata}"
            self.con.api(cmd)
            
        elif event_name == "CHANNEL_HANGUP":
            # 呼叫挂断
            if uuid in self.transcriptions:
                duration = time.time() - self.transcriptions[uuid]["start_time"]
                print(f"\n呼叫结束: {uuid} (时长: {duration:.1f}秒)")
                
                # 保存转录结果
                self._save_transcription(uuid)
                
                # 清理
                del self.transcriptions[uuid]
        
        elif event_name == "CUSTOM":
            subclass = e.getHeader("Event-Subclass")
            
            if subclass == "mod_audio_stream::json":
                # 收到转录结果
                body = e.getBody()
                
                try:
                    data = json.loads(body)
                    
                    if uuid in self.transcriptions:
                        # 添加转录片段
                        if "transcript" in data:
                            transcript = {
                                "time": time.time(),
                                "text": data["transcript"],
                                "confidence": data.get("confidence", 0.0)
                            }
                            
                            self.transcriptions[uuid]["transcripts"].append(transcript)
                            
                            print(f"[{uuid}] 转录: {data['transcript']}")
                except json.JSONDecodeError:
                    pass
    
    def _save_transcription(self, uuid):
        """保存转录结果"""
        if uuid not in self.transcriptions:
            return
        
        data = self.transcriptions[uuid]
        
        # 生成完整转录
        full_transcript = "\n".join([
            f"[{t['time']:.1f}] {t['text']}"
            for t in data["transcripts"]
        ])
        
        # 保存到文件
        filename = f"transcription_{uuid}.txt"
        with open(filename, "w", encoding="utf-8") as f:
            f.write(f"呼叫者: {data['caller']}\n")
            f.write(f"被叫者: {data['callee']}\n")
            f.write(f"开始时间: {data['start_time']}\n")
            f.write(f"\n转录内容:\n{full_transcript}\n")
        
        print(f"✓ 转录已保存到: {filename}")
    
    def stop(self):
        """停止监控"""
        self.running = False
        self.con.disconnect()

# 使用示例
if __name__ == "__main__":
    monitor = CallTranscriptionMonitor()
    monitor.start()
    
    try:
        # 保持运行
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n停止监控...")
        monitor.stop()
```

---

## 最佳实践

### 1. 连接管理

```python
class ESLConnectionManager:
    """ESL 连接管理器（支持重连）"""
    
    def __init__(self, host, port, password, max_retries=3):
        self.host = host
        self.port = port
        self.password = password
        self.max_retries = max_retries
        self.con = None
        self._connect()
    
    def _connect(self):
        """建立连接"""
        for attempt in range(self.max_retries):
            try:
                self.con = ESL.ESLconnection(self.host, self.port, self.password)
                
                if self.con.connected():
                    print(f"✓ 连接成功 (尝试 {attempt + 1}/{self.max_retries})")
                    return True
                else:
                    print(f"✗ 连接失败 (尝试 {attempt + 1}/{self.max_retries})")
            except Exception as e:
                print(f"✗ 连接异常: {e}")
            
            if attempt < self.max_retries - 1:
                time.sleep(2 ** attempt)  # 指数退避
        
        raise Exception("无法连接到 FreeSWITCH")
    
    def ensure_connected(self):
        """确保连接可用"""
        if not self.con or not self.con.connected():
            print("连接丢失，重新连接...")
            self._connect()
    
    def execute(self, func, *args, **kwargs):
        """执行操作（自动重连）"""
        self.ensure_connected()
        
        try:
            return func(*args, **kwargs)
        except Exception as e:
            print(f"执行失败: {e}")
            # 尝试重连并重试一次
            self._connect()
            return func(*args, **kwargs)
```

### 2. 错误处理

```python
def safe_api_call(con, command, default_value=None):
    """安全的 API 调用"""
    try:
        result = con.api(command)
        
        if result:
            body = result.getBody().strip()
            
            # 检查错误
            if body.startswith("-ERR"):
                print(f"API 错误: {body}")
                return default_value
            
            return body
        else:
            print("API 调用返回 None")
            return default_value
            
    except Exception as e:
        print(f"API 调用异常: {e}")
        return default_value

# 使用示例
result = safe_api_call(con, "status", default_value="N/A")
```

### 3. 日志记录

```python
import logging
from datetime import datetime

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler('esl_app.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

class LoggedESLConnection:
    """带日志的 ESL 连接"""
    
    def __init__(self, host, port, password):
        self.con = ESL.ESLconnection(host, port, password)
        
        if self.con.connected():
            logger.info(f"已连接到 {host}:{port}")
        else:
            logger.error(f"连接失败: {host}:{port}")
    
    def api(self, command):
        """执行 API 命令（带日志）"""
        logger.debug(f"执行命令: {command}")
        
        start_time = time.time()
        result = self.con.api(command)
        duration = time.time() - start_time
        
        if result:
            response = result.getBody().strip()
            logger.debug(f"命令响应 ({duration:.3f}s): {response[:100]}")
            return result
        else:
            logger.error(f"命令失败: {command}")
            return None
```

### 4. 性能优化

```python
# 使用 bgapi 进行并发操作
def parallel_api_calls(con, commands):
    """并行执行多个 API 命令"""
    job_uuids = []
    
    # 订阅后台任务事件
    con.events("plain", "BACKGROUND_JOB")
    
    # 提交所有命令
    for cmd in commands:
        result = con.bgapi(cmd)
        if result:
            job_uuid = result.getHeader("Job-UUID")
            job_uuids.append((job_uuid, cmd))
    
    # 收集结果
    results = {}
    remaining = set(job_uuids)
    
    while remaining:
        e = con.recvEventTimed(5000)  # 5秒超时
        
        if e:
            job_uuid = e.getHeader("Job-UUID")
            
            for jid, cmd in remaining.copy():
                if jid == job_uuid:
                    results[cmd] = e.getBody()
                    remaining.remove((jid, cmd))
                    break
        else:
            # 超时
            break
    
    return results

# 使用示例
commands = [
    "status",
    "show channels",
    "show calls"
]

results = parallel_api_calls(con, commands)
for cmd, result in results.items():
    print(f"{cmd}: {result[:100]}")
```

---

## 常见问题

### 问题 1: ImportError: No module named 'ESL'

**解决方案**：

```bash
# 检查 ESL 模块位置
find /usr -name "ESL.py" 2>/dev/null

# 添加到 Python 路径
export PYTHONPATH=/usr/local/lib/python3.x/dist-packages:$PYTHONPATH

# 或在代码中添加
import sys
sys.path.append('/usr/local/lib/python3.x/dist-packages')
import ESL
```

### 问题 2: 连接被拒绝

**症状**：
```
Connection refused
```

**解决方案**：

1. 检查 Event Socket 是否运行：
```bash
fs_cli -x "module_exists mod_event_socket"
```

2. 检查监听端口：
```bash
netstat -tulpn | grep 8021
```

3. 检查防火墙：
```bash
sudo ufw allow 8021/tcp
```

### 问题 3: 认证失败

**症状**：
```
Authentication failed
```

**解决方案**：

检查密码配置：
```bash
# 查看配置
cat /etc/freeswitch/autoload_configs/event_socket.conf.xml | grep password

# 使用正确的密码
con = ESL.ESLconnection("localhost", "8021", "CorrectPassword")
```

### 问题 4: 事件丢失

**症状**: 某些事件未被接收。

**解决方案**：

```python
# 1. 确保正确订阅
con.events("plain", "all")  # 订阅所有事件

# 2. 使用非阻塞接收避免事件堆积
while True:
    e = con.recvEventTimed(10)  # 10ms 超时
    if e:
        process_event(e)
    else:
        # 执行其他任务
        do_other_work()

# 3. 使用事件队列
import queue
import threading

event_queue = queue.Queue()

def event_receiver():
    while True:
        e = con.recvEvent()
        if e:
            event_queue.put(e)

# 启动接收线程
threading.Thread(target=event_receiver, daemon=True).start()

# 主线程处理事件
while True:
    try:
        e = event_queue.get(timeout=1)
        process_event(e)
    except queue.Empty:
        continue
```

### 问题 5: 内存泄漏

**症状**: Python 进程内存持续增长。

**解决方案**：

```python
# 1. 显式释放事件对象
e = con.recvEvent()
if e:
    process_event(e)
    del e  # 释放

# 2. 定期重连
connection_time = time.time()
MAX_CONNECTION_TIME = 3600  # 1小时

while True:
    if time.time() - connection_time > MAX_CONNECTION_TIME:
        print("重新连接...")
        con.disconnect()
        con = ESL.ESLconnection("localhost", "8021", "ClueCon")
        connection_time = time.time()
    
    # 正常处理...
```

---

## 总结

通过本指南，您应该能够：

1. ✅ 安装和配置 Python ESL
2. ✅ 理解 Inbound 和 Outbound 模式
3. ✅ 执行基本的 FreeSWITCH 操作
4. ✅ 订阅和处理事件
5. ✅ 集成 mod_audio_stream 模块
6. ✅ 开发生产级应用
7. ✅ 处理常见问题

### 相关资源

- [FreeSWITCH 官方文档](https://freeswitch.org/confluence/)
- [ESL API 参考](https://freeswitch.org/confluence/display/FREESWITCH/Event+Socket+Library)
- [mod_audio_stream 集成指南](./FreeSWITCH集成指南.md)
- [代码逻辑分析](./代码逻辑分析.md)

### 获取帮助

- FreeSWITCH 社区: https://freeswitch.org/confluence/
- GitHub Issues: https://github.com/amigniter/mod_audio_stream/issues

---

**祝您开发顺利！** 🚀
