# 无法获得的秘密

## 题目简述

远程 Linux 桌面只能通过 VNC 接收键盘、鼠标输入和屏幕输出，没有网络，VNC 剪贴板也被服务端配置为双向禁用。随机文件以只读方式挂载到 `/secret`，网页提交接口会逐字节比较上传文件与同一会话生成的秘密，并同时检查长度。因此目标不是读取文本 flag，而是在受限的人机界面上建立一条可靠的屏幕数据通道，把任意二进制文件丝毫不差地传出。

## 解题过程

### 把字节映射为像素

远程环境包含 Firefox。可以通过键盘在远端创建一个本地 HTML 页面，让浏览器读取 `/secret`，再把每个字节映射为一个灰度像素：

```html
<input type="file" id="file-input">
<button id="run">run</button>
<canvas id="data" width="1024" height="512"></canvas>
<script>
const width = 1024;
document.querySelector('#run').onclick = async () => {
    const file = document.querySelector('#file-input').files[0];
    const bytes = new Uint8Array(await file.arrayBuffer());
    const canvas = document.querySelector('#data');
    const ctx = canvas.getContext('2d');
    const image = ctx.createImageData(canvas.width, canvas.height);
    for (let i = 0; i < bytes.length; i++) {
        image.data[4 * i + 0] = bytes[i];
        image.data[4 * i + 1] = bytes[i];
        image.data[4 * i + 2] = bytes[i];
        image.data[4 * i + 3] = 255;
    }
    ctx.putImageData(image, 0, 0);
};
</script>
```

在远程终端先执行 `stat -c %s /secret` 记录精确长度，再在文件选择器中选择 `/secret`。一字节对应一个像素，位置为 `(i % 1024, i // 1024)`，因此截图必须保持画布 1:1 缩放并精确裁切。

### 避免 noVNC 的有损传输

直接从浏览器 noVNC 截图时，画面编码可能让少数字节发生 ±1 的变化。最直接的修复是把题目 `/connect` WebSocket 转为本地 TCP，然后使用支持 Raw、TightPNG 等无损编码的 VNC 客户端连接。代理的核心逻辑如下，地址和会话 Cookie 均使用自己的值：

```python
import asyncio
import websockets

COOKIE = open("cookie", encoding="utf-8").read().strip()
WS_URL = "ws://<题目主机>/connect"

async def handler(reader, writer):
    async with websockets.connect(
        WS_URL,
        extra_headers=[("Cookie", COOKIE)],
    ) as ws:
        async def tcp_to_ws():
            while data := await reader.read(4096):
                await ws.send(data)

        async def ws_to_tcp():
            async for data in ws:
                writer.write(data)
                await writer.drain()

        tasks = [asyncio.create_task(tcp_to_ws()),
                 asyncio.create_task(ws_to_tcp())]
        _, pending = await asyncio.wait(
            tasks, return_when=asyncio.FIRST_COMPLETED
        )
        for task in pending:
            task.cancel()
        writer.close()

async def main():
    server = await asyncio.start_server(handler, "127.0.0.1", 12010)
    async with server:
        await server.serve_forever()

asyncio.run(main())
```

本地 VNC 客户端连到 `127.0.0.1:12010` 后，切换为无损编码并抓取画布。按记录的文件长度解码并截断尾部未使用像素：

```python
from PIL import Image
import numpy as np

secret_size = 0  # 替换为远端 stat 输出
img = Image.open("canvas.png").convert("RGB")
data = np.asarray(img, dtype=np.uint8)[:, :, 0].reshape(-1)
open("secret.bin", "wb").write(data[:secret_size].tobytes())
```

远端和本地分别执行 `sha256sum /secret`、`sha256sum secret.bin`。只有长度和 SHA-256 都完全一致时才上传 `secret.bin`；提交接口验证通过后返回 flag。若只能使用有损屏幕链路，则必须为每帧加入纠错码、序号和校验和，不能继续使用无冗余的一像素一字节方案。

## 方法总结

- 核心技巧：把任意二进制编码为屏幕像素，通过 VNC 视频通道外带，再以长度和 SHA-256 做端到端完整性验证。
- 识别信号：网络、共享目录和剪贴板均不可用，但仍允许键盘输入、浏览器本地文件读取和屏幕输出；服务端要求精确文件而非可读文本。
- 复用要点：先区分信道是否有损。无损信道可直接映射字节；有损信道必须使用无损 VNC 编码或加入纠错。截图缩放、裁剪范围、颜色管理和尾部长度都会破坏字节级恢复，最终必须用强哈希闭环验证。
