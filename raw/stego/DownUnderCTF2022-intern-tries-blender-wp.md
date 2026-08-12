# DownUnderCTF 2022 Intern Tries Blender Writeup

## 题目简述

题目提供一个 Blender 扩展脚本。插件在 3D Viewport 侧栏注册 `DUCTF Flag Generator`，弹窗中有 26 个真假难辨的布尔选项；不同组合会创建文字、网格和相机。真正的 flag 不是某个源码字符串，而是四组错位文字从正确视角叠合后的空间信息。

## 解题过程

在 Blender 的 Scripting 工作区载入并运行 `2022_flag_gen_ext.py`，然后回到 3D Viewport，按 `N` 打开右侧栏。插件会同时加入一个无关的 Text Tool 面板和真正的 `DUCTF` 面板，后者的 `Gen Flag` 按钮打开选项对话框。

源码把选项 `a` 到 `z` 分别绑定到大量几何操作。多数分支生成明显的假 flag、猴头、圆环或干扰文本。结合选项名称和各分支创建文字的坐标、字距，正确组合是：

```text
h  Attempt 17
n  Flag4DUCTF
t  Make Flag 3
w  Solves Flag
```

只勾选这四项并执行生成。它们分别创建以下看似无意义的文字对象，并设置不同的平移、字符间距和挤出深度：

```text
F0E T 4HgCXUS
C wr x noE  TADRST
U{SeNr
DT tTiv}rta  tnina h4rd A 3F
```

不要从对象字符串逐项拼接；应切到顶视图（小键盘 `7`），再用正交视图和 `Home`/Frame All 把所有生成对象纳入画面。字符在 XY 平面上的错位会互相补全，从上向下读出：

```text
DUCTF{w0rStExTenTioNEv4r}
```

题面说明 flag 不含空格，且 `ExTenTioN` 的拼写就是生成结果，不应擅自改成 `Extension`。

## 方法总结

这里的源码文字只是空间载荷的碎片，决定性步骤是选择正确分支并从顶视角重组。分析 Blender 插件时应分开记录“对象内容”和“对象变换”：位置、字距、旋转、视角共同决定最终可见文本。直接 grep `DUCTF` 只会命中大量诱饵；最稳妥的验证是按控制流生成最小对象集合，再从题面提示的 top-down 视角观察。
