# GlacierCTF 2025 typst-lotto

## 题目简述

服务启动一个长期运行的 `typst watch`。每轮管理员先编译形如 `位置编号 + 随机数字` 的文档，数字范围为 0 到 9；玩家随后可以上传 Typst 文档并下载对应 profiling JSON，最后猜数字。连续猜中 15 轮后返回 flag。

同一 watcher 内的 Typst memoization 状态不会清空。若玩家编译的文档与本轮管理员文档相同，字体子集化等 PDF 阶段会命中缓存，profiling 时间显著缩短甚至省略大量事件。决定性障碍是还原 Typst 的缓存和 trace 行为，因此归入逆向。

## 解题过程

### 1. 排除 PDF 本身

仓库附带的 `main.pdf` 只有一张空白页。逐页视觉检查没有文字、图像或隐蔽答案；PDF 在本题中只是编译产物，真正的观测面是 `profile/<n>.json`。因此不应从空白页截图或对象流强行推导 flag。

### 2. 为每个候选复现管理员输入

第 $i$ 轮管理员文档是两个字符的串：

```text
<i><secret_digit>
```

对候选 $d\in\{0,1,\ldots,9\}$，上传完全相同模板的 `f"{i}{d}"`。每次构建后下载刚生成的 profile JSON。位置编号必须包含在文档里，否则先前轮次或自己刚做的候选测试也可能产生同内容缓存命中，无法把信号归因于当前管理员文档。

### 3. 用 `subset font` 时间识别缓存命中

普通候选会执行较完整的 PDF pipeline；正确候选复用管理员刚计算过的结果，`subset font` 区间明显更短。官方 solver 在固定版本 trace 中读取第 41、42 个事件：

```python
def subset_font_time(events):
    if len(events) < 43:
        return 0.0
    begin, end = events[41], events[42]
    assert begin["name"] == end["name"] == "subset font"
    assert begin["ph"] == "B" and end["ph"] == "E"
    return float(end["ts"]) - float(begin["ts"])
```

trace 很短本身也表示整个构建已被缓存，因此记为 0。更稳健的实现应按 `name == "subset font"` 和 B/E 配对搜索，而不是永久依赖数组下标。

收集十个候选的时长，取最小值对应的数字，然后选择菜单中的 Continue 并提交猜测。profiling 有调度噪声；参考 solver 仅在“恰有一个候选落入合理低时延区间”时接受，否则重连重试，避免一次异常测量毁掉后续 15 轮。

源码实例最终返回：

```text
gctf{b7c86a8166_th4nks_f0r_pl4y1ng_th3_typst_l0tt0_a59050c8e}
```

## 方法总结

本题是一条缓存侧信道，而不是暴力猜测。攻击者控制只改变候选数字，位置编号、文档模板、watcher 和编译环境保持不变；正确输入命中管理员留下的 memoized computation，造成 `subset font` 时间或 trace 长度的可测差异。设计此类服务时，不能把同一编译进程、缓存和细粒度 profiling 同时暴露给不同信任域；每轮应使用独立进程并限制 timings 下载。
