# 3ctu4_card_game

## 题目简述

服务端下发一批经过随机旋转并加入噪点的卡牌图片，要求在 10 秒内判断每张图属于宝可梦集换式卡牌游戏（PTCG）还是游戏王（YGO），总准确率必须高于 90%。题目不提供训练集，因此需要自行收集两类卡图、训练二分类模型，并在推理前把倾斜卡片矫正为统一尺寸。

决定解法的障碍是图像分类模型及其预处理流程，所以本题应归入 AI/ML，而不是沿用官方总 PDF 中的 Misc 外层分类。

## 解题过程

### 准备训练数据

PTCG 与 YGO 都有大量公开卡图。训练集可以从卡牌资料站、官方图鉴或本地游戏资源中整理，目录约定为：

```text
dataset/
├── ptcg/
│   ├── 0001.png
│   └── ...
└── ygo/
    ├── 0001.png
    └── ...
```

两类样本数量应尽量接近，并划出独立验证集。真正重要的是让训练数据覆盖服务端会施加的旋转、缩放和噪声；只用完全干净、方向一致的卡图，很容易在本地获得高准确率，却无法通过在线测试。

### 透视矫正

服务端图片以白色为背景，卡牌是面积最大的非白色连通区域。先转灰度，以 `254` 为阈值反色二值化，再进行开、闭运算去除孤立噪点并连接边缘。取最大外轮廓的最小外接旋转矩形，通过四点透视变换把卡牌拉正。

下面是一份完整的训练与 ZIP 批量预测脚本。它把所有输入统一为高 400、宽 300 的竖版图像，并确保 OpenCV 的 `(width, height)` 参数与 Keras 的 `(height, width, channels)` 输入形状一致：

```python
from io import BytesIO
from pathlib import Path
from zipfile import ZipFile

import cv2
import numpy as np
from sklearn.model_selection import train_test_split
from tensorflow import keras
from tensorflow.keras import layers

HEIGHT = 400
WIDTH = 300
LABELS = ["ptcg", "ygo"]


def order_points(points):
    points = np.asarray(points, dtype=np.float32)
    ordered = np.zeros((4, 2), dtype=np.float32)
    coordinate_sum = points.sum(axis=1)
    coordinate_diff = np.diff(points, axis=1).ravel()
    ordered[0] = points[np.argmin(coordinate_sum)]   # top-left
    ordered[2] = points[np.argmax(coordinate_sum)]   # bottom-right
    ordered[1] = points[np.argmin(coordinate_diff)]  # top-right
    ordered[3] = points[np.argmax(coordinate_diff)]  # bottom-left
    return ordered


def rectify_card(image):
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    _, binary = cv2.threshold(gray, 254, 255, cv2.THRESH_BINARY_INV)
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (3, 3))
    binary = cv2.morphologyEx(binary, cv2.MORPH_OPEN, kernel)
    binary = cv2.morphologyEx(binary, cv2.MORPH_CLOSE, kernel)

    contours, _ = cv2.findContours(
        binary,
        cv2.RETR_EXTERNAL,
        cv2.CHAIN_APPROX_SIMPLE,
    )
    if not contours:
        raise ValueError("card contour not found")

    contour = max(contours, key=cv2.contourArea)
    source = order_points(cv2.boxPoints(cv2.minAreaRect(contour)))
    destination = np.array(
        [
            [0, 0],
            [WIDTH - 1, 0],
            [WIDTH - 1, HEIGHT - 1],
            [0, HEIGHT - 1],
        ],
        dtype=np.float32,
    )
    transform = cv2.getPerspectiveTransform(source, destination)
    return cv2.warpPerspective(image, transform, (WIDTH, HEIGHT))


def preprocess(image):
    card = rectify_card(image)
    return card.astype(np.float32) / 255.0


def load_dataset(root):
    images = []
    labels = []
    root = Path(root)

    for label, class_name in enumerate(LABELS):
        for path in sorted((root / class_name).iterdir()):
            image = cv2.imread(str(path))
            if image is None:
                continue
            try:
                images.append(preprocess(image))
                labels.append(label)
            except ValueError:
                continue

    return np.asarray(images), np.asarray(labels, dtype=np.int64)


def build_model():
    model = keras.Sequential(
        [
            layers.Input(shape=(HEIGHT, WIDTH, 3)),
            layers.Conv2D(
                filters=128,
                kernel_size=16,
                padding="same",
                activation="relu",
                kernel_initializer="he_normal",
            ),
            layers.MaxPooling2D(pool_size=16),
            layers.Dropout(0.3),
            layers.GlobalAveragePooling2D(),
            layers.Dense(2, activation="softmax"),
        ]
    )
    model.compile(
        optimizer="adam",
        loss="sparse_categorical_crossentropy",
        metrics=["accuracy"],
    )
    return model


def train(dataset_root, output_model="cards.keras"):
    images, labels = load_dataset(dataset_root)
    train_x, valid_x, train_y, valid_y = train_test_split(
        images,
        labels,
        test_size=0.3,
        random_state=1,
        stratify=labels,
    )

    model = build_model()
    checkpoint = keras.callbacks.ModelCheckpoint(
        output_model,
        monitor="val_accuracy",
        save_best_only=True,
    )
    model.fit(
        train_x,
        train_y,
        validation_data=(valid_x, valid_y),
        epochs=15,
        batch_size=64,
        callbacks=[checkpoint],
    )

    best_model = keras.models.load_model(output_model)
    loss, accuracy = best_model.evaluate(valid_x, valid_y, verbose=0)
    print(f"validation loss: {loss:.4f}")
    print(f"validation accuracy: {accuracy:.4%}")


def read_zip_images(zip_path):
    with ZipFile(zip_path, "r") as archive:
        for name in archive.namelist():
            raw = archive.read(name)
            image = cv2.imdecode(np.frombuffer(raw, np.uint8), cv2.IMREAD_COLOR)
            if image is not None:
                yield name, image


def predict_zip(model_path, zip_path):
    model = keras.models.load_model(model_path)
    names = []
    batch = []

    for name, image in read_zip_images(zip_path):
        names.append(name)
        batch.append(preprocess(image))

    probabilities = model.predict(np.asarray(batch), batch_size=64, verbose=0)
    predicted = np.argmax(probabilities, axis=1)
    return [
        (name, LABELS[label], float(probabilities[index, label]))
        for index, (name, label) in enumerate(zip(names, predicted))
    ]


if __name__ == "__main__":
    # 首次执行时训练；在线答题前只加载一次模型并整批预测。
    train("dataset")
    for result in predict_zip("cards.keras", "cards.zip"):
        print(*result)
```

在线交互时应把 `train("dataset")` 移到赛前离线阶段，只在程序启动时加载一次 `cards.keras`。收到 ZIP 后批量预处理、一次性调用 `model.predict`，再按服务端要求提交 `ptcg` 或 `ygo` 标签；否则反复加载模型很容易超过 10 秒限制。

官方题解没有记录在线实例返回的固定 flag，但其验收条件是分类正确率超过 90%，模型达到阈值后服务端即返回 flag。

## 方法总结

本题由“几何归一化”和“视觉分类”两部分组成。最大轮廓与透视变换先消除旋转和位置差异，CNN 再学习两类卡牌在版式、边框与图像风格上的区别。对有严格时限的在线图像题，模型大小不是唯一指标；模型只加载一次、图像批量推理、训练与在线阶段彻底分离，往往同样决定能否通过验收。
