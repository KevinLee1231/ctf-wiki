# impossible stego

## 题目简述

附件包含一张 $828\times449$ 的 RGBA PNG 和约 23 MB 的 `messages.log`。图像中的载荷不是按固定顺序写入的普通 LSB，而是由作者在一次 Claude 编程会话中生成的自定义 `stego` 包写入。题目的突破口不在像素统计，而在日志泄露了生成程序的完整源码历史。

![嵌入自定义散布 LSB 载荷的黄色旗帜 RGBA PNG 载体](SekaiCTF2026-impossible-stego-wp/flag-lsb-carrier.png)

## 解题过程

`messages.log` 是 60 行 NDJSON。每行的 `req_body`、`resp_body` 都经过 Base64 编码；Anthropic Messages API 会在后续请求中重复携带先前对话，因此最大的请求体已经包含完整的 114 条消息。解码后可以看到 `Write`、`Edit`、`Read`、`Bash` 等工具调用。

恢复源码时只需重放会改变文件的两类记录：

```python
for message in messages:
    for block in message.get("content", []):
        if block.get("type") != "tool_use":
            continue
        args = block["input"]
        if block["name"] == "Write":
            files[args["file_path"]] = args["content"]
        elif block["name"] == "Edit":
            count = -1 if args.get("replace_all") else 1
            path = args["file_path"]
            files[path] = files[path].replace(
                args["old_string"], args["new_string"], count
            )
```

所有操作必须按对话顺序执行，否则后面的 `Edit` 会落在错误版本上。日志中共能追踪到 20 个写入目标；其中根目录的 `stego.py` 是被放弃的首版，最终应使用多文件的 `stego/` 包。最关键的 `stego/secret.py` 直接暴露了 64 字节 `ROOT_SECRET`、HKDF 的 extract/expand salt、S-box seed 和各层的域分离标签。

最终嵌入流程为：

```text
MAGIC | VERSION | LENGTH | DATA | CRC32
  -> ChaCha20
  -> S-box
  -> whitening
  -> append truncated HMAC-SHA256
  -> keyed multi-round scatter
  -> ±1 matched RGB LSB
```

帧头使用小端格式 `<4sBI`：magic 为 `SkG2`，版本为 2，随后是 4 字节数据长度，末尾 CRC32 覆盖头部和数据。密钥调度先用 `ROOT_SECRET`、extract salt 和图像的 `width || height || channels` 做 HKDF-Extract，再按不同标签做 HKDF-Expand，分别得到 ChaCha20、nonce、HMAC、S-box、whitening 和 scatter 所需材料。

散布顺序由四个独立子密钥控制，依次执行：

1. 将全部 RGB 槽位切成 64 至 256 个块并打乱块序；
2. 做一次全局循环移位，再分别旋转各窗口；
3. 用 4 轮 Feistel 网络置换索引，并通过 cycle walking 把结果限制回实际槽位范围；
4. 对整个序列再做一次 Fisher-Yates。

每个载荷 bit 对应一个 RGB 样本。若样本最低位已经正确便保持原值，否则根据密钥流决定加一或减一；边界值 0、255 则只能向内调整。这样单个颜色分量最多变化 1，alpha 全程不动。

提取不是简单地把整个容量读完。程序先按相同散布顺序读取 9 字节编码头，依次逆 whitening、逆 S-box、ChaCha20 解密，得到 magic、版本和数据长度。随后按该长度重新读取完整编码帧与 16 字节 HMAC：

1. 先验证 HMAC-SHA256 截断标签是否匹配编码帧；
2. 再逆 whitening、逆 S-box 并用 ChaCha20 解密；
3. 最后校验 magic、版本、声明长度和 CRC32，取出数据字段。

恢复包后调用 `stego.extract("flag.png")` 即可。一个包含重放脚本与恢复后源码的[可复现题解仓库](https://github.com/Abdelkad3r/SekaiCTF-2026/blob/ef87965f07e32162595457285da138f181492f54/misc/impossible-stego/README.md)可用于交叉核对，但关键算法和操作顺序已经在上文给全，无需依赖外链理解解法。

恢复结果为：

```text
SEKAI{th3y_d1dn7_4ctually_l3t_m3_us3_my7h0s_f0r_7h1s_0n3_s4dly_i_h4d_i7_r3r0ut3d_t0_0pus}
```

## 方法总结

日志本身才是证据层：它不仅保存自然语言回答，还保存每次工具调用里的完整源码和修改参数。面对“AI 生成工具 + 最终产物”的题，应先检查代理、网关和会话日志是否重放了全量上下文，再考虑逆向像素算法。恢复时要区分被弃用的首版与最终包，并利用 magic、HMAC 和 CRC32 三层校验确认源码版本及提取结果一致。
