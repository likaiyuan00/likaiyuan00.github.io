---
title: websocket
date: 2025-05-28 15:43:49
tags: websocket
---
# 异步
因为websocket会使用到异步操作先了解一下异步
```python
import asyncio
import time


async def task(name, duration):
    print(f"[{time.strftime('%H:%M:%S')}] 任务 {name} 开始")
    await asyncio.sleep(duration)  # 模拟并发等待
    print(f"[{time.strftime('%H:%M:%S')}] 任务 {name} 完成")


def task_(name, duration):
    print(f"[{time.strftime('%H:%M:%S')}] 任务 {name} 开始")
    time.sleep(duration)  # 模拟耗时操作
    print(f"[{time.strftime('%H:%M:%S')}] 任务 {name} 完成")


async def main():
    start_time = time.time()
    print(f"[{time.strftime('%H:%M:%S')}] 异步任务开始")

    await asyncio.gather(
        task("A", 2),
        task("B", 3),
        task("C", 1)
    )

    end_time = time.time()
    print(f"[{time.strftime('%H:%M:%S')}] 异步任务总耗时: {end_time - start_time:.2f} 秒")


def main_():
    start_time = time.time()
    print(f"[{time.strftime('%H:%M:%S')}] 同步任务开始")

    task_("A", 2)
    task_("B", 3)
    task_("C", 1)

    end_time = time.time()
    print(f"[{time.strftime('%H:%M:%S')}] 同步任务总耗时: {end_time - start_time:.2f} 秒")


if __name__ == "__main__":
    print("======================异步==========================")
    asyncio.run(main())
    print("======================同步==========================")
    main_()


#结果可以看出异步不需要等待会直接执行下一步操作，任务完成可以使用await来回调处理完成结果
======================异步==========================
[14:45:43] 异步任务开始
[14:45:43] 任务 A 开始
[14:45:43] 任务 B 开始
[14:45:43] 任务 C 开始
[14:45:44] 任务 C 完成
[14:45:45] 任务 A 完成
[14:45:46] 任务 B 完成
[14:45:46] 异步任务总耗时: 3.00 秒
======================同步==========================
[14:45:46] 同步任务开始
[14:45:46] 任务 A 开始
[14:45:48] 任务 A 完成
[14:45:48] 任务 B 开始
[14:45:51] 任务 B 完成
[14:45:51] 任务 C 开始
[14:45:52] 任务 C 完成
[14:45:52] 同步任务总耗时: 6.00 秒
```
# websocket

## 服务端
```python
import asyncio
import websockets
#https://websockets.readthedocs.io/en/stable/
# 处理客户端连接
async def handle_client(websocket):
    async for message in websocket:
        print(f"收到客户端消息: {message}")
        reply = f"机器人回复：你说的是 '{message}' 对吗？"
        await websocket.send(reply)


# async def main_logic(websocket, path):
#    # await check_permit(websocket)
#
#     await handle_client(websocket)

# 启动服务器
async def main():
    async with websockets.serve(handle_client, "localhost", 8765):
        print("WebSocket 服务器已启动，端口 8765")
        await asyncio.Future()  # 永久运行

asyncio.run(main())
```



## 客户端
```python
import asyncio
import websockets

async def client():
    async with websockets.connect("ws://localhost:8765") as websocket:
        while True:
            message = input("请输入消息（输入 q 退出）: ")
            if message == 'q':
                break
            await websocket.send(message)
            response = await websocket.recv()
            print(f"收到回复: {response}")

asyncio.run(client())
#效果，相当于打开了一个通道双方都可以发消息
WebSocket 服务器已启动，端口 8765
请输入消息（输入 q 退出）: hello websockets
收到回复: 机器人回复：你说的是 'hello websockets' 对吗？
请输入消息（输入 q 退出）: 
```
# 额外
## fastapi框架使用websocket
```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse
from fastapi.middleware.cors import CORSMiddleware
import json

app = FastAPI()

# 配置CORS跨域
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# HTML页面（修改了前端WebSocket实现）
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>FastAPI 聊天</title>
    <style>
        body { max-width: 800px; margin: 20px auto; padding: 20px; }
        #output { 
            height: 300px; 
            border: 1px solid #ccc; 
            overflow-y: auto; 
            padding: 10px; 
            margin-bottom: 10px;
        }
        #input { 
            width: 80%; 
            padding: 8px;
            margin-right: 10px;
        }
        button {
            padding: 8px 16px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
        button:hover {
            opacity: 0.8;
        }
    </style>
</head>
<body>
    <div id="output"></div>
    <input type="text" id="input" placeholder="输入消息...">
    <button onclick="sendMessage()">发送</button>

    <script>
        // 初始化WebSocket连接
        const socket = new WebSocket(`ws://${window.location.host}/ws`);

        // 连接成功回调
        socket.onopen = () => {
            addMessage('系统', '已连接到服务器');
        };

        // 接收消息处理
        socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            addMessage('机器人', data.message);
        };

        // 错误处理
        socket.onerror = (error) => {
            addMessage('系统', `连接错误: ${error.message}`);
        };

        // 关闭连接处理
        socket.onclose = () => {
            addMessage('系统', '连接已断开');
        };

        // 发送消息
        function sendMessage() {
            const input = document.getElementById('input');
            const message = input.value.trim();
            if (message) {
                socket.send(JSON.stringify({
                    type: "user_message",
                    content: message
                }));
                addMessage('我', message);
                input.value = '';
            }
        }

        // 添加消息到界面
        function addMessage(sender, content) {
            const output = document.getElementById('output');
            const div = document.createElement('div');
            div.innerHTML = `<strong>${sender}:</strong> ${content}`;
            output.appendChild(div);
            // 自动滚动到底部
            output.scrollTop = output.scrollHeight;
        }
    </script>
</body>
</html>
'''


@app.get("/")
async def index():
    return HTMLResponse(HTML_TEMPLATE)


# WebSocket端点
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            # 接收客户端消息
            data = await websocket.receive_text()
            message_data = json.loads(data)

            # 处理客户端消息
            if message_data['type'] == 'user_message':
                print(f"收到客户端消息: {message_data['content']}")

                # 构造回复消息
                reply = {
                    "type": "server_response",
                    "message": f"机器人回复：你说的是 '{message_data['content']}' 对吗？"
                }

                # 发送回复
                await websocket.send_json(reply)

    except WebSocketDisconnect:
        print("客户端断开连接")
    except Exception as e:
        print(f"发生错误: {str(e)}")


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8001)
```
## flask使用websocket
```python
import eventlet
eventlet.monkey_patch()  # 关键：启用异步支持
from flask import Flask, render_template_string
#pip install flask-socketio eventlet
from flask_socketio import SocketIO, emit


app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret!'
socketio = SocketIO(app, cors_allowed_origins="*")  # 允许跨域


@app.route('/')
def index():
    return render_template_string('''
    <!DOCTYPE html>
    <html>
    <head>
        <title>Socket.IO 聊天</title>
        <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.7.2/socket.io.js"></script>
        <style>
            /* 保持原有样式不变 */
            body { max-width: 800px; margin: 20px auto; padding: 20px; }
            #output { height: 300px; border: 1px solid #ccc; overflow-y: auto; padding: 10px; }
        </style>
    </head>
    <body>
        <div id="output"></div>
        <input id="input" placeholder="输入消息">
        <button onclick="send()">发送</button>

        <script>
            const socket = io();  // 自动连接当前域名

            // 连接成功回调
            socket.on('connect', () => {
                addMessage('系统', '已连接到服务器');
            });

            // 接收服务器消息
            socket.on('server_response', (data) => {
                addMessage('机器人', data.message);
            });

            // 发送消息
            function send() {
                const input = document.getElementById('input');
                const message = input.value.trim();
                if (message) {
                    socket.emit('client_message', message);
                    addMessage('我', message);
                    input.value = '';
                }
            }

            // 添加消息到界面
            function addMessage(sender, content) {
                const div = document.createElement('div');
                div.innerHTML = `<strong>${sender}:</strong> ${content}`;
                document.getElementById('output').appendChild(div);
                // 自动滚动到底部
                const output = document.getElementById('output');
                output.scrollTop = output.scrollHeight;
            }
        </script>
    </body>
    </html>
    ''')


# Socket.IO 事件处理
@socketio.on('client_message')
def handle_message(message):
    print(f'收到客户端消息: {message}')
    # 构造回复消息
    reply = f"机器人回复：你说的是 '{message}' 对吗？"
    # 发送消息给客户端
    emit('server_response', {'message': reply})


if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=8000, debug=True)
```
## 大模型使用websocket聊天
```python
# main.py
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse
import requests
import json

app = FastAPI()

# 存储对话历史 (生产环境建议使用数据库)
conversation_history = []

# 集成前端页面与后端逻辑
HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>AI 对话助手</title>
    <style>
        body {
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            font-family: Arial, sans-serif;
        }
        #chatContainer {
            height: 60vh;
            border: 1px solid #ddd;
            border-radius: 8px;
            overflow-y: auto;
            padding: 15px;
            margin-bottom: 15px;
            background: #f9f9f9;
        }
        .message {
            margin: 10px 0;
            padding: 12px;
            border-radius: 15px;
            max-width: 80%;
            word-wrap: break-word;
        }
        .user-message {
            background: #e3f2fd;
            margin-left: auto;
            border-bottom-right-radius: 5px;
        }
        .bot-message {
            background: #fff;
            border: 1px solid #eee;
            margin-right: auto;
            border-bottom-left-radius: 5px;
        }
        #inputContainer {
            display: flex;
            gap: 10px;
        }
        #userInput {
            flex: 1;
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 25px;
            outline: none;
        }
        button {
            padding: 12px 25px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 25px;
            cursor: pointer;
            transition: background 0.3s;
        }
        button:hover {
            background: #0056b3;
        }
        .status {
            color: #666;
            text-align: center;
            padding: 10px;
        }
    </style>
</head>
<body>
    <h1>AI 对话助手</h1>
    <div id="chatContainer"></div>
    <div id="inputContainer">
        <input type="text" id="userInput" placeholder="输入消息..." />
        <button onclick="sendMessage()">发送</button>
    </div>
    <div class="status" id="status">连接状态：正常</div>
    
   // <iframe
   //      src="http://47.237.81.149/chatbot/9h9nyQcblGTesiGJ"
    //     style="width: 100%; height: 100%; min-height: 700px"
   //      frameborder="0"
  //       allow="microphone">
   // </iframe>

    <script>
        const ws = new WebSocket('ws://' + window.location.host + '/ws');
        const chatContainer = document.getElementById('chatContainer');
        let isBotResponding = false;

        // WebSocket 事件处理
        ws.onopen = () => updateStatus('已连接');
        ws.onclose = () => updateStatus('连接已断开');
        ws.onerror = () => updateStatus('连接错误');

        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            handleResponse(data);
        };

        function handleResponse(data) {
            switch(data.type) {
                case 'user_message':
                    appendMessage(data.content, 'user');
                    break;
                case 'assistant_start':
                    isBotResponding = true;
                    appendMessage('', 'bot');
                    break;
                case 'assistant_chunk':
                    appendChunk(data.content);
                    break;
                case 'assistant_end':
                    isBotResponding = false;
                    break;
                case 'error':
                    appendMessage(`错误：${data.content}`, 'error');
                    break;
            }
        }

        function appendMessage(content, role) {
            const div = document.createElement('div');
            div.className = `message ${role}-message`;
            div.textContent = content;
            chatContainer.appendChild(div);
            scrollToBottom();
        }

        function appendChunk(content) {
            const messages = document.getElementsByClassName('bot-message');
            const lastMsg = messages[messages.length - 1];
            lastMsg.textContent += content;
            scrollToBottom();
        }

        function scrollToBottom() {
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        function updateStatus(text) {
            document.getElementById('status').textContent = `状态：${text}`;
        }

        function sendMessage() {
            const input = document.getElementById('userInput');
            const message = input.value.trim();
            if (message && !isBotResponding) {
                ws.send(message);
                input.value = '';
            }
        }

        // 支持回车发送
        document.getElementById('userInput').addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });
    </script>
</body>
</html>
"""


@app.get("/")
async def get():
    return HTMLResponse(HTML_TEMPLATE)


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            # 接收用户消息
            user_message = await websocket.receive_text()

            # 更新对话历史
            conversation_history.append({"role": "user", "content": user_message})

            # 发送用户消息到前端
            await websocket.send_json({
                "type": "user_message",
                "content": user_message
            })

            # 准备流式请求
            await websocket.send_json({"type": "assistant_start"})

            # 构造请求数据
            request_data = {
                "model": "deepseek-r1:latest",
                "messages": conversation_history,
                "stream": True
            }

            # 流式获取响应
            full_response = []
            with requests.post(
                    "http://1.1.1.1:11434/api/chat",#大模型接口地址
                    json=request_data,
                    stream=True
            ) as response:
                response.raise_for_status()
                for line in response.iter_lines():
                    if line:
                        chunk = json.loads(line.decode('utf-8'))
                        if 'message' in chunk:
                            content = chunk['message']['content']
                            full_response.append(content)
                            await websocket.send_json({
                                "type": "assistant_chunk",
                                "content": content
                            })

            # 保存完整响应
            conversation_history.append({
                "role": "assistant",
                "content": "".join(full_response)
            })
            await websocket.send_json({"type": "assistant_end"})

    except Exception as e:
        await websocket.send_json({
            "type": "error",
            "content": f"系统错误: {str(e)}"
        })
    finally:
        await websocket.close()


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8000)
```