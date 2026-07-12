# Performance_artist

## 题目简述

本题是图像分类恢复题。附件中的图片可按训练集类别还原出一串十六进制数据，题目没有加噪音，样本也直接来自训练集，因此既可以人工目视分类，也可以用官方 notebook 训练 16 类分类器自动匹配。除了分类本身，图片还被修改过 PNG 高度，内嵌 ZIP 又做了伪加密，需要先修复载体和压缩包结构，才能得到最终恢复文件。

## 解题过程

简单的图像分类，没想到大家都直接用眼睛做了。题目里的提示文字是：

```text
手写体字画通过人脸匹配训练集搞出来了
```

还好长度不是太长（压缩包里放个图片还做不做了emm）

按说把hint放的数据集放搜索引擎找一下就有很多带详解的代码了。为了降低难度，题目都是直接用的训练集，没有加噪音，也就是说不用考虑过拟合问题。（好像师傅们都是把图分出来直接匹配数据集去了= =）

这次拿到的官方 notebook 里，分类器并不是只识别 0-9。它先加载 MNIST 数字，再加载 `emnist-letters.mat`，只取 EMNIST 中标签小于 7 的前 6 类字母，并把这些标签整体加 9，拼成 `0-9` 与 `a-f` 共 16 类。这样每个小图块的分类结果就可以直接当作一个十六进制半字节。

训练模型是一个简单 CNN：

```python
model = models.Sequential()
model.add(layers.Conv2D(32, (3, 3), activation='relu', input_shape=[28, 28, 1]))
model.add(layers.Conv2D(64, (3, 3), activation='relu'))
model.add(layers.MaxPooling2D((2, 2)))
model.add(layers.Conv2D(64, (3, 3), activation='relu'))
model.add(layers.Flatten())
model.add(layers.Dense(64, activation='relu'))
model.add(layers.Dense(16))
```

训练 20 个 epoch 后，notebook 中测试集准确率约为 `0.9870`。之后读取 `attachment.png`，按 `32` 列、`23` 行切成 `28 * 28` 的小块，对每个小块预测类别：

```python
pic = plt.imread('attachment.png')
out_text = []
for i in range(23):
    for j in range(32):
        picc = np.vsplit(np.hsplit(pic, 32)[j], 23)[i]
        out_text.append(model.predict(picc.reshape(1, 28, 28, 1, order='A')).argmax())
```

随后每两个分类结果组成一个字节。`0-9` 直接作为十六进制数字，大于 9 的类别转换为 `a-f`，最终写出 `out.zip`：

```python
overfile = open('out.zip', 'wb')
for i in range(0, len(out_text), 2):
    if out_text[i] > 9:
        out_text[i] = chr(out_text[i] + 87)
    if out_text[i + 1] > 9:
        out_text[i + 1] = chr(out_text[i + 1] + 87)
    a = str(out_text[i])
    b = str(out_text[i + 1])
    overfile.write(bytes.fromhex(a + b))
overfile.close()
```

因此解题主线可以稳定保留为：修正 PNG 高度使完整图块可见；把图片按 `23 * 32` 个 `28 * 28` 样本切分；用 MNIST + EMNIST 前 6 类构造十六进制分类器；把预测结果两两合并为字节恢复 `out.zip`；最后处理 ZIP 伪加密。

哦 差点忘了 png图片有修改图片高度，压缩包进行了伪加密（比赛时有师傅发现图片显示的zip没有文件尾，没想到改png高度emm 个人感觉这两个知识点国内很常见 - -）

## 方法总结

图像分类题要先判断样本是否来自公开训练集、是否加噪、是否需要训练模型。本题因为直接使用训练集，核心不是泛化分类，而是样本匹配和载体修复。这里的可复用点是把 `0-9` 和 `a-f` 统一成 16 类，把图块分类结果按十六进制半字节还原文件。若图片中疑似还藏有压缩包，需要同时检查 PNG 高度、文件尾、ZIP magic 和伪加密位，避免把分类结果和容器修复割裂开。
