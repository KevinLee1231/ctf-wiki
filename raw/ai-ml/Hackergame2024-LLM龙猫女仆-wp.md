# 先不说关于我从零开始独自在异世界转生成某大厂家的 LLM 龙猫女仆这件事可不可能这么离谱，发现 Hackergame 内容审查委员会忘记审查题目标题了ごめんね，以及「这么长都快赶上轻小说了真的不会影响用户体验吗🤣」

## 题目简述

题目使用固定 SHA-256 的 Qwen2.5-3B-Instruct Q8_0 GGUF 模型和固定 prompt 随机生成英文文本，然后逐字符审查。第一问把所有属于字符串 `hackergame` 的字符替换成 `x`，第二问扩大为 `hackergame of ustc`。选手得到审查后的 `after.txt` 以及原文 `before.txt` 的 SHA-256，目标是恢复字节完全一致、且不能多出末尾换行的原文。最终 flag 后缀取恢复文本 SHA-512 的前 16 个十六进制字符。

这不是普通掩码替换或 Web 外壳题。原文是特定 LLM 在固定 prompt 下的一条采样轨迹，恢复过程必须利用模型给出的 token 概率与 tokenizer 分词，因此决定性障碍是模型行为，归入 `ai-ml`。

## 解题过程

### 把审查文本变成 token 级约束

对候选 token，先用模型 tokenizer 得到其原始 token piece，再按字节与 `after.txt` 当前未消费位置比较。审查集合只有 ASCII 小写字母，未被替换的 UTF-8 字节则原样匹配：

- token 中若字符属于审查字符集合，则目标位置必须为 `x`；
- 其他字符必须与目标文本完全相同；
- token piece 长度不能超过 `after.txt` 的剩余字节数；
- 只有恰好消费完全文时才允许 EOG `<|im_end|>`。

可以把约束写成：

```text
compatible(piece, masked, pos) :=
  len(piece) <= len(masked) - pos，且
  对 piece 中每个字符 c：
    c 属于审查集合  => masked[pos+i] == 'x'
    c 不属于审查集合 => masked[pos+i] == c
```

在 llama.cpp sampler 链完成 repetition penalty、top-k、tail-free、typical、top-p、min-p、temperature 和 softmax 后，将所有不兼容 token 的概率设为 0。剩余 token 就是这个位置上既符合模型采样规则、又能生成当前掩码前缀的分支。

### 重放正确的模型与 prompt

必须与出题环境保持一致：模型文件 SHA-256 为
`6dcc22694c8654b045ec40bbe350212b88893fd9010e8474bae5b19a43578ba1`，模型是 Qwen2.5-3B-Instruct Q8_0，系统消息为 `You are a professional CTF player.`。两问的用户消息分别是：

```text
Write a few sentences for Hackergame 2024 (中国科学技术大学 (University of Science and Technology of China) 第十一届信息安全大赛) in English. The more funny and unreal the better. About 100 words.
```

```text
Write a short article for Hackergame 2024 (中国科学技术大学 (University of Science and Technology of China) 第十一届信息安全大赛) in English. The more funny and unreal the better. About 500 words.
```

原生成程序使用随机 seed，但恢复时不需要猜中 seed：搜索枚举的是每个已知前缀经过同一 sampler 截断后仍有非零概率的全部 token，最终再由原文哈希选择真实采样轨迹。

官方 patch 基于 llama.cpp 提交 `c421ac072d46172ab18924e1e8be53680b54ed3b`，修改 sampler 使其返回所有概率非零的候选 token，而不是只随机抽一个。官方 `grammar_set_p` 通过与字符串结尾的 `NUL` 不匹配来排除越界 piece；更稳妥的实现应像上面的约束一样先显式检查剩余长度，避免读取 `after.txt` 缓冲区边界之外。每个搜索节点至少保存：

```cpp
struct Node {
    std::vector<llama_token> tokens; // prompt 加已恢复 token
    const char *pos;                 // after.txt 中当前位置
    Node *parent;
    std::vector<Node *> children;
    int n;                           // 访问次数
    float p;                         // 模型给该分支的概率
};
```

### 用概率引导搜索，而不是盲目 DFS

如果当前位置只有一个合法 token，直接沿该 token 前进；出现多个合法 token 时建立子节点。单纯 DFS 会让某个早期但概率极低的分支占用大量推理，因此官方解法用类似 MCTS/UCT 的分支选择：

```text
score(child) = p(child)
             + 0.5 * sqrt(2 * log(parent.visits) / (child.visits + 1))
```

第一项优先探索模型认为更自然的 token，第二项保证其他尚未充分访问的可行分支仍有机会。每次走到 EOG 或死路，就把当前 token 序列转回字节串并计算 SHA-256：

- 等于给定 `before.sha256`：找到唯一需要的原文并写入 `output.txt`；
- 不等：删除该叶节点，向上剪去已无孩子的分支，再按 UCT 选择下一条路径。

哈希是最终 oracle，掩码兼容性只负责大幅剪枝。第二问故意选择了次优分支较多的生成结果，因此保留概率排序与探索项比朴素 DFS 重要得多。

### 输出与 flag 验证

恢复文件必须按 token piece 原样拼接，不能让编辑器自动添加末尾换行。对官方给出的恢复结果进行本地只读哈希核对：

```text
level 1: 717 字节
SHA-256 = 809101c781f829a33021750de895b7f5130ba6c8f42862e955650dbf7f3c21d7
SHA-512 前 16 位 = fa7b655c38bc8847

level 2: 3088 字节
SHA-256 = f0d1d40fdef63ea6a6dc97ba78a59512deb07ad9ecad1e3fd16c83151d51fe58
SHA-512 前 16 位 = dd7bbc89d6984575
```

两份文件均确认末尾没有 CR/LF。按题目格式将相应的 SHA-512 前缀填入 `flag{llm_lm_lm_koshitantan_<prefix>}` 即可。此次整理没有编译 llama.cpp 或运行数十分钟的模型搜索；搜索性能与完整复现路径依据官方 patch 静态核对，最终文本哈希则已在本地验证。

## 方法总结

- 核心技巧：把被审查文本转成 tokenizer 级硬约束，用原模型的采样概率引导树搜索，并用原文 SHA-256 作为精确终止 oracle。
- 识别信号：固定模型/prompt、随机采样输出、字符级遮罩和原文哈希共同说明不能只做词典填空，必须重放模型概率分布。
- 复用要点：模型文件、量化版本、prompt 模板、sampler 顺序和 tokenizer 任一不一致都会改变搜索树；输出按字节验收时尤其要检查 token piece、UTF-8 和末尾换行。对于多分支长文本，概率优先加探索奖励通常显著优于纯 DFS。
