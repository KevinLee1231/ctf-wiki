# Machoke's Flex

## 题目简述

题目程序用 OpenGL 渲染 Machoke 模型，并提示玩家被模型吸引得看不到 flag。源码显示程序还使用 FreeType 生成前 128 个 ASCII 字符的纹理，并在模型之外绘制一组文字四边形。

决定性线索有两处：

- 文本顶点统一被送到 $z=60$，而初始相机位于 $z=50$ 并朝相反方向观察，文字实际位于相机背后；
- 数组 `tid` 记录绘制每个四边形时绑定的纹理 ID。

## 解题过程

FreeType 初始化循环按 ASCII 值从 0 到 127 依次调用 `glGenTextures`：

```cpp
for (unsigned char c = 0; c < 128; c++) {
    unsigned int texture;
    glGenTextures(1, &texture);
    // 把字符 c 的 glyph 上传到 texture
    Characters.insert({c, {texture, /* ... */}});
}
```

在该程序的纹理分配顺序中，字符 $c$ 对应的纹理 ID 为 $c+1$。渲染循环却没有按字符表查询，而是直接遍历硬编码数组：

```cpp
const unsigned int tid[] = {
    86, 78, 69, 68, 85, 71, 124, 120, /* ... */
};

glBindTexture(GL_TEXTURE_2D, tid[i]);
```

因此逐项减 1 并转为 ASCII 即可静态恢复文本：

```python
texture_ids = [
    86, 78, 69, 68, 85, 71, 124, 120, 112, 120, 96, 106,
    96, 99, 102, 117, 96, 122, 112, 118, 96, 111, 102, 119,
    102, 115, 96, 116, 98, 120, 96, 117, 105, 106, 116, 96,
    100, 112, 110, 106, 111, 104, 126,
]

print("".join(chr(texture_id - 1) for texture_id in texture_ids))
```

也可以动态修改文本顶点着色器，把
`vec4(vertex.xy, 60.0, 1.0)` 中的深度改到相机可见范围，或旋转相机观察身后；但静态解码不依赖缺失的模型头文件，也更直接。

结果为：

```text
UMDCTF{wow_i_bet_you_never_saw_this_coming}
```

## 方法总结

- 核心技巧：追踪 FreeType glyph 到 OpenGL texture ID 的分配顺序，把硬编码纹理 ID 数组还原成 ASCII。
- 识别信号：3D 程序中除了主模型还存在文字 shader、glyph atlas 和一组离屏顶点时，flag 可能被渲染在相机视锥之外。
- 复用要点：纹理 ID 不是通用字符编码；只有结合本程序的创建顺序才能得出 `id - 1`。应先用 `UMDCTF` 前缀验证这一映射。
