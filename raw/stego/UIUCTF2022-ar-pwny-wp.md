# AR Pwny

## 题目简述

页面没有可利用的后端逻辑，只用 `<model-viewer>` 加载 `pwny.glb`，允许旋转、缩放并进入 WebXR、Scene Viewer 或 Quick Look 的 AR 模式。仓库 README 直接说明“flag embedded in the mesh”，所以决定性障碍是观察 3D 模型中刻意隐藏的视觉信息，而不是攻击 Flask 静态文件服务。

模型里藏有两张 QR Code，分别编码 flag 的前半段和后半段。题目页面建议在手机上进入 AR，原因是移动观察点更容易看到位于模型不同朝向或内部表面上的二维码。

## 解题过程

### 获取并检查 3D 载体

从页面源代码可见真正的载体：

```html
<model-viewer
  src="pwny.glb"
  ar
  ar-modes="webxr scene-viewer quick-look"
  camera-controls
  enable-pan>
</model-viewer>
```

浏览器中可以直接旋转、平移和放大模型；手机支持时也可以点击 AR 按钮，绕模型移动观察。若普通视角一直被 pwny 外壳遮挡，则下载 `pwny.glb`，在任意 glTF/GLB 查看器中隐藏外层网格或把相机移入模型。不要把页面下方的导航二维码当成 flag：它只编码比赛期间的题目网址。

### 分别扫描两张隐藏二维码

第一张隐藏图对应 flag 前半段：

![AR Pwny 模型内部用于编码 flag 前半段的第一张 QR](UIUCTF2022-ar-pwny-wp/qr-flag-first-half.png)

对其解码得到：

```text
uiuctf{welcome_2_the_meataverse_
```

第二张隐藏图对应后半段：

![AR Pwny 模型另一朝向用于编码 flag 后半段的第二张 QR](UIUCTF2022-ar-pwny-wp/qr-flag-second-half.png)

解码得到：

```text
erm_i_meant_pwnyverse}
```

两段按观察顺序直接拼接，不需要再做 Base 编码或密码运算：

```text
uiuctf{welcome_2_the_meataverse_erm_i_meant_pwnyverse}
```

仓库中的两张源素材已分别用二维码解码器核对；页面的 `linkback.png` 解码结果只是旧挑战地址，因而没有作为题解图片保留。

## 方法总结

- 核心技巧：把可交互 3D 模型视为具有遮挡和朝向的空间隐写载体，通过改变相机位置或拆分网格找到两张 QR，并按语义顺序重组文本。
- 识别信号：服务端仅静态返回页面与 GLB，题面强调 AR/手机，模型可自由旋转，而源码又明确提示信息位于 mesh 中。
- 复用要点：3D 题要同时检查外表面、背面、内部面、纹理和独立节点；场景中的每张二维码都应单独解码并判断用途，不能把导航码、装饰码和 flag 码混为一谈。
