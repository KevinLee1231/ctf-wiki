# Hackergame2020 233 同学的 Docker WP

## 题目简述

233 同学在构建字符串工具镜像时把 `/code/flag.txt` 复制进了镜像，随后又用一条新的 `RUN rm` 命令删除它。容器最终文件系统中确实看不到该文件，但 Docker 镜像由不可变层叠加而成，后续删除不会改写已经生成的旧层。

本题需要从容器镜像这一既有数字证据中恢复被删除文件，因此归入取证方向。

## 解题过程

Dockerfile 中每条会改变文件系统的构建指令都会产生新的 layer。删除文件时，新层只记录“这个路径在合并视图中应当不可见”，常见表现是 whiteout 文件；包含原始内容的旧层仍在镜像里。

先拉取并导出镜像：

```sh
docker pull 8b8d3c8324c7/stringtool
docker save -o img.tar 8b8d3c8324c7/stringtool

mkdir image
tar -xf img.tar -C image
```

`image/manifest.json` 的 `Layers` 数组按照从底层到顶层的顺序列出各个 `layer.tar`。可以逐层检查 `/code/flag.txt` 何时出现、又在何时被 whiteout：

```python
import json
import tarfile
from pathlib import Path

root = Path("image")
manifest = json.loads((root / "manifest.json").read_text())[0]

for index, layer_name in enumerate(manifest["Layers"]):
    layer_path = root / layer_name
    with tarfile.open(layer_path) as archive:
        names = archive.getnames()
        hits = [
            name for name in names
            if name.endswith("code/flag.txt") or ".wh." in name and "flag" in name
        ]
        if hits:
            print(index, layer_name, hits)
```

定位到仍含文件的旧层后，直接从该层读取，不必把所有层覆盖解压：

```sh
tar -xOf image/<layer-id>/layer.tar code/flag.txt
```

输出为：

```text
flag{Docker_Layers!=PS_Layers_hhh}
```

## 方法总结

镜像的最终合并视图不等于镜像历史。把秘密复制到某一层后，再在后续层中 `rm`，只能隐藏路径，不能从旧层清除字节。构建时应在同一条 `RUN` 中创建并清理临时文件，更重要的是不要让密钥或 flag 进入任何构建层；即使后来 squash，泄露过的制品和缓存也必须视为已暴露。
