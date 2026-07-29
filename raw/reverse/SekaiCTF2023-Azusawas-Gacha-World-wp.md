# Azusawa's Gacha World

## 题目简述

题目提供一个 Unity Mono 抽卡游戏。客户端初始只有 1000 水晶，而目标四星卡在累计第 1,000,000 抽时才必定出现。主仓库中的题目目录是空的 Git submodule；README 指向的 [作者仓库](https://github.com/jktrn/azusawas-gacha-world) 包含完整 Unity 工程、后端源码和构建产物，作者也提供了[原始分析文章](https://enscribe.dev/blog/sekaictf-2023/azusawas-gacha-world/)。

核心不是暴力点击，而是逆向 `Assembly-CSharp.dll` 中的请求协议，并注意后端完全信任客户端提交的 `crystals` 与 `pulls`。

## 解题过程

Unity Mono 构建把游戏逻辑保存在：

```text
Asusawa's Gacha World_Data/Managed/Assembly-CSharp.dll
```

用 dnSpy、ILSpy 等 .NET 反编译器打开后，`GameState` 显示初始状态：

```csharp
public int crystals = 1000;
public int pulls = 0;
```

`GachaManager.SendGachaRequest` 则把三个字段序列化成 JSON：

```json
{
  "crystals": 1000,
  "pulls": 0,
  "numPulls": 1
}
```

请求为 HTTP POST，路径是 `/gacha`，并要求：

```http
Content-Type: application/json
User-Agent: SekaiCTF
```

客户端中的服务地址只是 Base64 编码，并非加密。具体比赛 IP 已失效，不应作为复现前提；协议字段和 `/gacha` 路径才是稳定机制。

后端源码把累计抽数先约化到一百万以内，然后逐抽递增：

```typescript
let totalPulls = pulls % 1000000

for (let i = 0; i < numPulls; i++) {
    totalPulls += 1
    if (totalPulls >= 1000000) {
        rarity = 4
        totalPulls -= 1000000

        const imageBuffer = fs.readFileSync('src/flag.png')
        result.characters.push({
            ...fourStarCharacter,
            flag: imageBuffer.toString('base64'),
        })
    }
}
```

服务端只检查三项都是 number、`crystals >= 100 * numPulls`，并限制 `numPulls` 为 1 或 10；它没有保存服务器侧账户状态。因此直接提交：

```json
{
  "crystals": 999999,
  "pulls": 999999,
  "numPulls": 1
}
```

即可让下一抽进入四星分支。也可以在反编译器中把 `GameState.crystals` 和 `GameState.pulls` 的初值都改为 `999999`，保存修改后的程序集，再从游戏界面抽一次。

成功响应的 `characters[0].flag` 是 Base64 编码 PNG。解码后图中给出的文字为：

```text
SEKAI{D0N7_73LL_53G4_1_C0P13D_7H31R_G4M3}
```

原始图片主要是装饰背景与上述文字，已直接转写，不再保留冗余截图。

## 方法总结

Unity Mono 题应先检查 `Assembly-CSharp.dll`，通常比从原生 UnityPlayer 入手直接。这里客户端不仅泄露了接口和请求头，还能任意声明水晶数与历史抽数；后端把客户端字段当权威状态，导致一次请求即可伪造累计百万抽。Base64 在两处都只是表示层：一处隐藏 URL，另一处承载 flag 图片，均不构成安全边界。
