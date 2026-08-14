# MetaSIZE

## 题目简述

题目是导出到 Web 的 Godot 游戏，共两部分：先识别游戏引擎，再从游戏资源中找到隐藏在关卡里的 Flag。网页脚本做了混淆，并把通常名为 `.pck` 的资源包伪装成其他扩展名。

`index.html` 的 `GAME_CONFIG` 仍泄露了 Godot 导出结构和资源包大小：`MetaSIZE.pck` 应为 23878336 字节；目录中恰好存在同样大小的 `helpers.wasm`，服务工作线程也把它列为缓存资源。

## 解题过程

第一部分可从 Godot Web 导出的固定结构识别引擎，包括 `Engine(GAME_CONFIG)`、主 WASM、service worker、PCK 配置及 Godot 风格图标资源。答案为：

```text
godot
```

第二部分先把伪装资源复制回标准扩展名：

```bash
cp helpers.wasm MetaSIZE.pck
```

再使用 [Godot RE Tools / gdsdecomp](https://github.com/GDRETools/gdsdecomp) 恢复项目。该工具能够从未加密 PCK 中提取场景、脚本和资源，并在版本匹配时反编译 GDScript 字节码。正文所需流程是：

1. 加载 `MetaSIZE.pck` 并恢复项目目录；
2. 用兼容版本 Godot 打开恢复项目；
3. 检查关卡场景、碰撞体和脚本，定位正常路线不可达的 Flag 区域；
4. 临时移除碰撞、提高移动能力或直接在编辑器中打开目标关卡，进入隐藏区域读取 Flag。

最终得到：

```text
grey{godot_hakor_2dcf839d}
```

## 方法总结

- 核心技巧：识别 Godot Web 导出结构，通过配置中的预期 PCK 大小找出伪装资源包，再恢复场景和脚本。
- 识别信号：`Engine`、WASM、service worker、`.pck` 配置和体积相同的可疑二进制文件。
- 复用要点：扩展名不可信，应结合文件大小、加载代码和文件魔数定位资源；恢复项目后直接检查场景通常比硬玩完整游戏更高效。
