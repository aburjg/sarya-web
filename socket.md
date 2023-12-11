
1. User hits `/post-endpoint/` with JSON
2. Server passes that out to client
3. Client json loads `message in websocket` does process the `json.dumps` back to server

Complexity in `global active_websocket` the server has to maintain a directory of connections

Would this work?

```py
import asyncio
import websockets
import uvicorn
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from pydantic import BaseModel

app = FastAPI()

# To store the active WebSocket connection
active_websocket = None

class Item(BaseModel):
    data: dict

@app.post("/post-endpoint/")
async def post_endpoint(item: Item):
    if active_websocket:
        await active_websocket.send(str(item.data))
        response = await active_websocket.recv()
        return {"received_data": response}
    else:
        return {"error": "No active WebSocket connection"}

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    global active_websocket
    await websocket.accept()
    active_websocket = websocket
    try:
        while True:
            data = await websocket.receive_text()
            # Process data or just echo it back
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        active_websocket = None

if __name__ == "__main__":
    uvicorn.run(app, host="localhost", port=8000)
```

client
```py
import asyncio
import websockets
import json

async def connect():
    while True:
        try:
            async with websockets.connect("ws://localhost:8000/ws") as websocket:
                # Initial message to server (optional)
                await websocket.send(json.dumps({"message": "Hello Server!"}))

                # Listen for messages from the server
                async for message in websocket:
                    data = json.loads(message)  # Parse JSON
                    print(f"Received from server: {data}")

                    # Process data (example: add a new key-value pair)
                    processed_data = {**data, "processed": True}

                    # Send processed data back to the server
                    await websocket.send(json.dumps(processed_data))

        except (websockets.ConnectionClosed, ConnectionRefusedError):
            print("Connection lost... retrying")
            await asyncio.sleep(5)

asyncio.run(connect())
```
