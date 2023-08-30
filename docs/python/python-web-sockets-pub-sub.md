# Python Websockets pub-sub


```html title="index.html"
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>WebSocket chat</h1>
    <form action="" onsubmit="sendMessage(event)">
        <input type="text" id="messageText" autocomplete="off"/>
        <button>Send</button>
    </form>
 
    <ul id="messages">
 
    </ul>
<script src="scripts/main.js"></script>
</body>
</html>
```

```javascript title="main.js"
let ws = new WebSocket("ws://localhost:1234")
 
ws.onmessage = function (event) {
    let messages = document.getElementById("messages");
    let message = document.createElement('li');
    let content = document.createTextNode(event.data);
    message.appendChild(content);
    messages.appendChild(message);
}
 
function sendMessage(event) {
    let input = document.getElementById("messageText");
    ws.send(input.value);
    input.value = "";
    event.preventDefault();
}
```

```python title="ws_server.py"
import asyncio
from functools import partial
 
from websockets import serve
from websockets.legacy.server import WebSocketServerProtocol
 
 
class PubSub:
    def __init__(self):
        self.waiter = asyncio.Future()
 
    async def publish(self, value):
        waiter, self.waiter = self.waiter, asyncio.Future()
        waiter.set_result((value, self.waiter))
 
    async def subscribe(self):
        waiter = self.waiter
        while True:
            value, waiter = await waiter
            yield value
 
    __aiter__ = subscribe
 
 
async def check_message(websocket: WebSocketServerProtocol, pub_sub: PubSub):
    async for message in pub_sub:
        await websocket.send(message)
 
 
async def handler(websocket: WebSocketServerProtocol, pub_sub: PubSub):
    print(f"Got a connection from {websocket.remote_address}")
    task = asyncio.create_task(check_message(websocket, pub_sub))
    try:
        while True:
            message = await websocket.recv()
            await pub_sub.publish(message)
    finally:
        task.cancel()
 
 
async def main():
    pub_sub = PubSub()
    async with serve(partial(handler, pub_sub=pub_sub), "localhost", 1234):
        await asyncio.Future()
 
if __name__ == '__main__':
    asyncio.run(main(), debug=True)
```