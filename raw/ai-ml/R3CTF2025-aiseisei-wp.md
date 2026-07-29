# aiseisei

## 题目简述

服务端接收一张不超过 1 MB 的 Base64 TIFF，先用 `exiftool` 提取其中的 ICC Profile，再把 TIFF 的像素区替换为随机打乱的 MNIST 测试集，最后调用 `iccApplyProfiles` 处理整批图片。输出中每个样本的最大通道下标会被当作分类结果；只有 10000 张测试图全部分类正确才会给出 flag。

随机打乱并没有隐藏图像与标签的对应关系，真正的限制是：选手只能提交 TIFF 和其中的 ICC Profile，不能直接提交模型代码。官方解法利用 ICC 的 Calculator/CLUT 等处理元素，把 MNIST CNN 的完整推理过程塞进色彩配置文件，让“颜色转换”本身变成分类器。

## 解题过程

### 还原输入布局

`set_input_data()` 把测试集整理成形状为 `(10000, 784)` 的数组，并把 TIFF 的 `ImageWidth` 改为 10000。配套的 `tiff_creator.c` 则把每个像素定义为 784 个输入样本，因此一个像素正好承载一张 $28\times28$ 的 MNIST 灰度图。

ICC 的输入和输出可以抽象为：

$$
\mathbb{R}^{784}\longrightarrow\mathbb{R}^{10}
$$

服务端随后对十个输出通道取 `argmax`。所以只要 ICC 对每个 784 维像素执行与官方 CNN 等价的计算，打乱顺序不会影响正确率。

### 把 CNN 编译进 ICC

官方 `model_dump.py` 与 `model_weight_dump.py` 给出了具体实现。处理链按模型结构展开：

1. 从 TIFF 像素读取 784 个归一化输入；
2. 用 CalculatorElement 实现卷积、偏置、ReLU 与池化；
3. 把 `best_mnist_cnn_996.pth` 中的权重逐层写入 ICC；
4. 执行全连接层并输出十个类别分数；
5. 对少数基础模型会误判的样本加入定向修正，使测试集准确率从 99.6% 补到 100%。

这里不是让 ICC 调用外部 Python，而是把乘加、比较和查表操作直接编码到 ICC 执行图中。服务端提取 Profile 后即使替换了原始像素，恶意 Profile 仍会作用于新写入的 MNIST 数据。

### 构造并提交 TIFF

用 `tiff_creator.c` 创建满足通道布局的 TIFF，并把生成的 Profile 嵌入 `ICCProfile` 标签。最终提交逻辑很简单：

```python
with open("input.tiff", "rb") as f:
    payload = base64.b64encode(f.read())

io.sendlineafter(b"Input Image", payload)
```

服务端依次提取 ICC、写入测试集并执行 Profile。输出达到 `10000 / 10000` 后即可得到 flag。

## 方法总结

本题的关键不是预测数据集的随机顺序，而是识别 ICC Profile 是一个可编程的数据处理容器。只要评测方把不可信 Profile 应用于秘密或后写入的数据，提交者就能把计算逻辑带进评测流程。

分析这类题时应先画清楚“提交内容、服务端替换内容、最终执行内容”三者的边界。题目源码与官方 `solution` 已包含模型、权重导出器和 TIFF 生成器，因此无需依赖外链即可复现完整机制。
