# accuracy

## 题目简述

附件一是带标签的 CSV，每行共有 785 列：第一列 `label`，其余 $784=28\times28$ 列是灰度像素；附件二包含 12272 张按编号排列的 $28\times28$ 黑白图片。类别为十六进制字符 `0-9a-f`，目标是训练分类器识别全部图片，按文件编号拼接预测字符并提交。服务端要求准确率高于 95%，因此决定性障碍是模型训练与批量推理，归入 AI/ML。

## 解题过程

先检查 CSV：像素取值位于 0 至 255，标签共有 16 类，图片尺寸与特征列数吻合。训练样本类似 MNIST，但额外包含 `a-f`。题目图片实际上从训练分布中生成；为了阻止直接逐像素匹配，每个非零像素都被减了 1，因此应训练能容忍微小像素扰动的分类器，而不是建立图片哈希表。

官方方案使用一层卷积、池化和两层全连接的 CNN。训练数据按 255 归一化；推理图片的黑白方向与 CSV 相反，所以读入后先计算 `255 - image`，再使用相同归一化。下面把 PDF 中跨三页的训练与预测代码整理成一份可执行脚本：

```python
from pathlib import Path

import numpy as np
import pandas as pd
import tensorflow as tf
from PIL import Image
from sklearn.model_selection import train_test_split


csv_path = Path("full_Hex.csv")
image_dir = Path("png")
model_path = Path("hex-cnn.keras")
result_path = Path("result.txt")

dataset = pd.read_csv(csv_path)
features = dataset.drop(columns=["label"]).to_numpy(dtype=np.float32)
labels = dataset["label"].to_numpy(dtype=np.int64)

if features.shape[1] != 28 * 28:
    raise ValueError(f"unexpected feature count: {features.shape[1]}")

features = (features / 255.0).reshape(-1, 28, 28, 1)
x_train, x_test, y_train, y_test = train_test_split(
    features,
    labels,
    test_size=0.25,
    random_state=2021,
    stratify=labels,
)
y_train = tf.keras.utils.to_categorical(y_train, num_classes=16)
y_test = tf.keras.utils.to_categorical(y_test, num_classes=16)

model = tf.keras.Sequential(
    [
        tf.keras.layers.Input(shape=(28, 28, 1)),
        tf.keras.layers.Conv2D(32, (5, 5), activation="relu"),
        tf.keras.layers.MaxPooling2D(pool_size=(2, 2)),
        tf.keras.layers.Dropout(0.3),
        tf.keras.layers.Flatten(),
        tf.keras.layers.Dense(128, activation="relu"),
        tf.keras.layers.Dense(64, activation="relu"),
        tf.keras.layers.Dense(16, activation="softmax"),
    ]
)
model.compile(
    optimizer="adam",
    loss="categorical_crossentropy",
    metrics=["accuracy"],
)
model.fit(
    x_train,
    y_train,
    validation_data=(x_test, y_test),
    epochs=5,
    batch_size=200,
    verbose=2,
)

_, test_accuracy = model.evaluate(x_test, y_test, verbose=0)
print(f"validation accuracy: {test_accuracy:.4%}")
model.save(model_path)

image_paths = sorted(image_dir.glob("*.png"), key=lambda path: int(path.stem))
if len(image_paths) != 12272:
    raise ValueError(f"unexpected image count: {len(image_paths)}")

images = []
for image_path in image_paths:
    image = Image.open(image_path).convert("L")
    array = np.asarray(image, dtype=np.float32)
    if array.shape != (28, 28):
        raise ValueError(f"unexpected image shape: {image_path} -> {array.shape}")
    images.append((255.0 - array) / 255.0)

batch = np.asarray(images, dtype=np.float32)[..., np.newaxis]
predicted_labels = model.predict(batch, batch_size=256).argmax(axis=1)
alphabet = "0123456789abcdef"
answer = "".join(alphabet[index] for index in predicted_labels)
result_path.write_text(answer, encoding="utf-8")
print(f"wrote {len(answer)} characters to {result_path}")
```

模型的 16 个输出节点依次映射到 `0123456789abcdef`，不能在推理阶段沿用 PDF 训练图表里展示用的大写 `A-F`。文件必须按数值编号排序，不能使用默认字典序，否则 `10.png` 会排在 `2.png` 前面并破坏整串答案。官方 PDF 报告该结构未经精调约有 98% 验证准确率，已经超过 95% 门槛。

当前仓库只有官方总 PDF，没有 `full_Hex.csv`、12272 张待识别图片或原提交端点，因此无法重新训练并核验最终提交结果；PDF 也没有记录静态 flag。这里保留完整可复现流程，但不臆造输出。

## 方法总结

先由维度、像素范围和标签数识别出十六进制字符分类任务，再保证训练和推理采用一致的缩放、通道形状与类别映射。题目对非零像素做减一扰动，只能阻止精确匹配，对经过归一化的 CNN 影响很小。此类题最后最容易出错的地方不是网络结构，而是黑白反相、编号排序、大小写映射和训练/推理预处理不一致；这些数据管线细节必须与模型准确率一起验证。
