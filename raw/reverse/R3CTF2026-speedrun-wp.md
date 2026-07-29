# speedrun

## 题目简述

题目运行在一个所有队伍共享的 Minecraft Paper 1.21.10 服务器上。正常流程是从随机世界开始速通，在一小时内击杀第一条末影龙，再根据服务器广播的坐标找到末地宝箱。宝箱书中只有一个形如 `RUN-...` 的兑换码，取得后服务器会在 10 秒倒计时结束时重启。

兑换码本身不是 flag。选手还要连接 TCP checker，依次提交本队 token 和兑换码；checker 调用比赛平台 API 把 token 解析成 `team_id`，再返回本队专属的动态 flag：

```text
r3ctf{<由 team_id 确定的 UUID>}
```

附件 `SpeedRunController.jar` 的 SHA-256 为：

```text
ab668a90ad9d42d20c11aa8966803dcfa0a57cae4a75d21e7135380b0ba4f84f
```

仅分析选手附件可以还原游戏流程，但不能直接列出兑换码，因为 JAR 运行时从服务器文件读取码表。官方仓库同时给出了部署目录、完整 `codes.json` 和 checker 源码，因此可以从源码审计得到一条确定解法。决定性工作是逆向 Java 插件并串联服务端校验逻辑，归入 `reverse`。

## 解题过程

### 还原 Minecraft 插件逻辑

列出 JAR 中的类：

```powershell
jar tf "SpeedRunController.jar"
```

关键类为：

```text
yuu/mc/speedRunController/CodeRepository.class
yuu/mc/speedRunController/RunMilestone.class
yuu/mc/speedRunController/SidebarManager.class
yuu/mc/speedRunController/SpeedRunController.class
```

使用 `javap -c -p` 或 Java 反编译器检查 `SpeedRunController`。`onDragonDeath()` 只处理本轮第一次末影龙死亡事件，记录击杀者后调用 `spawnFlagChest()`。后者通过 `CodeRepository.drawRandomCode()` 从 `SPEEDRUN_CODES_FILE` 指向的 JSON 数组随机选码，然后在末地寻找宝箱位置。

默认选点约束来自 `config.yml`：

```yaml
end-chest:
  min-coordinate: -3000
  max-coordinate: 3000
  exclude-center-radius: 500
  max-attempts: 600
```

具体逻辑是随机选择 $x,z\in[-3000,3000]$，排除 $x^2+z^2<500^2$ 的中心区域；候选位置的最高阻挡方块必须是末地石，上方连续两格必须为空气。插件最多尝试 600 次，然后在合法位置放置名为 `Code Chest` 的箱子，并把一本成书放在槽位 13。

书页明文包含选中的兑换码，同时 persistent data container 也以字符串保存同一个值。玩家通过捡起物品、点击或拖拽把这本书放入背包时，插件会确认其中的自定义 `NamespacedKey`，标记本轮已经领取，删除 Minecraft 容器自己那份码表中的该码，并启动 10 秒重启倒计时。

这解释了题面中的两个提示：

- 击杀龙后并不是直接显示 flag，而是广播末地宝箱坐标；
- 10 秒限制发生在领取书之后，所以应立即复制书中的 `RUN-...` 字符串。

### 从官方部署仓库取得合法兑换码

部署时，Minecraft 镜像和 checker 镜像分别执行：

```dockerfile
COPY shared/codes.json /shared/codes.json
```

两者没有挂载同一个可写 volume，而是在构建时各自复制一份初始码表。因此 Minecraft 插件领取后删除的是游戏容器中的副本；checker 中的副本保持不变，而且 `check_code.py` 明确只做成员检查，不会消费兑换码：

```python
def load_codes() -> set[str]:
    raw = json.loads(CODES_FILE.read_text(encoding="utf-8"))
    return {str(c) for c in raw}

code = read_line("Enter code:")
if not code or code not in codes:
    print(INVALID_MESSAGE, flush=True)
    return 0

print(make_flag(team_id), flush=True)
```

官方 `deploy/shared/codes.json` 共含 200 个合法值。仓库中第一个值即可作为输入：

```text
RUN-2F1A3D37A6B5B78CC0E2C89D
```

这是仓库公开后才能采用的源码复现捷径；比赛时若只有选手 JAR，则仍需按题目流程击杀末影龙并领取宝箱中的随机码。不能声称从 JAR 静态提取了码表，因为 `CodeRepository` 只保存文件路径和 JSON 读写逻辑，实际 200 个值不在附件内。

### 解析动态 flag 生成流程

checker 先读取本队 token：

```python
url = TEAM_API_URL + "?" + urllib.parse.urlencode({"token": token})
data = json.loads(urllib.request.urlopen(url).read().decode())
team_id = data["id"]
```

只有平台返回整数 `team_id` 后，checker 才检查兑换码。合法时执行：

```python
content = encode_uuid(FLAG_TEMPLATE, FLAG_KEY, team_id, True)
flag = f"r3ctf{{{content}}}"
```

部署中的参数为：

```text
FLAG_TEMPLATE = speedrun_mc_r3ctf_q6le8rxq
FLAG_KEY      = sBMu9o4ZIMWF9LusIYi8WlTc36JRMwN5
```

`encode_uuid()` 的主要步骤如下：

1. 计算 `SHA-1(FLAG_TEMPLATE)`，截取字节 2 至 17，得到 16 字节模板块；
2. 将 `team_id` 按 little-endian 有符号 64 位整数编码，再用 `FLAG_KEY` 做 XXTEA；
3. 把前 8 个加密字节异或到模板块的奇数位置；salt 固定为四个零字节，因此相同输入总是得到相同结果；
4. 对 `FLAG_KEY` 做 SHA-256，摘要前 32 字节作为 ChaCha20-Poly1305 密钥，末 12 字节作为 nonce；
5. 加密 16 字节模板块，取 ciphertext 的前 16 字节格式化为 UUID。

本地用仓库脚本对测试 `team_id=12345` 做往返验证：

```powershell
& "D:/文档/新建文件夹/venv/Scripts/python.exe" `
  "speedrun/deploy/checker/uuid_stego.py" `
  encode `
  "speedrun_mc_r3ctf_q6le8rxq" `
  "sBMu9o4ZIMWF9LusIYi8WlTc36JRMwN5" `
  12345
```

输出：

```text
flag{6adb8717-b544-326c-e256-3e79e985e6d8}
```

再把 UUID 交给 `decode` 会恢复 `12345`。这只是算法自测，不是任何队伍的比赛 flag。真实答案必须使用自己的 token 所对应的 `team_id`。

比赛服务仍可用时，完整提交方式为：

```text
$ nc challenge.ctf2026.r3kapig.com 30337
Enter your team token:
<本队 token>
Enter code:
RUN-2F1A3D37A6B5B78CC0E2C89D
r3ctf{<本队动态 UUID>}
```

仓库没有提供可替代个人 token 的固定 `team_id`，因此本文不伪造一条“通用 flag”。可验证的结论是：上述码存在于 checker 的合法集合中，而最终 UUID 由本队身份确定。

## 方法总结

- 共享游戏实例与动态 flag 是两个不同的共享边界。兑换码可以复用，但 token 解析出的 `team_id` 使不同队伍获得不同 flag。
- 分析附件时要区分“代码读取逻辑”和“运行时数据”。JAR 证明码来自 JSON，却不包含 200 个具体值；具体码只能从游戏流程或官方部署仓库取得。
- 两个容器都使用路径 `/shared/codes.json`，不代表它们共享文件。Dockerfile 各自 `COPY` 且部署没有共享 volume，解释了游戏端删除兑换码后 checker 仍能验证它。
- 动态题不能把测试向量冒充最终答案。应记录生成算法和已验证往返，同时把个人 token、`team_id` 与真实 flag 留在实际提交阶段。
